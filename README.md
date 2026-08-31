# incilab-runner

GitHub Actions runner for the InciLab data pipeline.

## Workflows

| Workflow | Schedule | Trigger | Description |
|----------|----------|---------|-------------|
| `pipeline.yml` | Every 3 days | Automatic + manual | Discover products → scrape images → ingredients → categories → reviews → rescore → moderate_content (independent job — content moderation safety-net sweep, see below) |
| `ingredients.yml` | Daily | Automatic + manual | Fill new ingredients from user searches |
| `categories.yml` | On demand | Manual only | Reclassify `product_category` in `products_cache` using LLM. Input: `force` (bool) — if true, reclassifies all products including those already categorized |
| `rescore.yml` | On demand | Manual only | Recalculate dermico/eco/eficacia scores with algorithm v2 |
| `enrich-full.yml` | 1st Sunday/month | Automatic + manual | Full ingredient regeneration from CosIng via LLM |
| `brands.yml` | Weekly (Sun 5am) | Automatic + manual | Scrape Jolse + Stylevana brand directories, upsert into `brands` table |
| `image_audit.yml` | Daily (5am) | Automatic + manual | OCR + vision check that each product image matches its label; empties confirmed mismatches |
| `retry.yml` | On demand | Manual only | Re-runs the enrichment steps of `pipeline.yml` against whatever failed (`data/incilab_failed.json` + empty columns in DB), skipping discovery and image scraping |

## How it works

Each workflow clones the private `incilab-enrich` repo at runtime and runs scripts from it. **To update pipeline logic, push to `incilab-enrich` — no changes needed here.**

**All workflows live in this repo and nowhere else.** `incilab-enrich` used to carry its own
copies (from before the split); they were removed in Aug 2026 because they had drifted — they
still used `actions/checkout@v4` instead of cloning, and never got the `Strip newlines from
secrets` fix or the concurrency group. Three had been disabled by hand in the GitHub UI, which
is invisible from the repo, so anyone reading that directory reasonably assumed all four ran.
If you ever need a workflow there again, remember that **concurrency groups do not span
repositories** — a copy in `incilab-enrich` can never serialize against these.

## Concurrency

`pipeline.yml`, `ingredients.yml`, `retry.yml` and `enrich-full.yml` share the
`incilab-enrich-writes` group, for two different reasons:

- The first three push state (`data/incilab_failed.json`) back to `incilab-enrich`; running
  two at once produces merge conflicts on that push.
- `enrich-full.yml` writes no state, but `load_supabase_v3.py` upserts all ~26,885 rows of the
  `ingredients` table, which is the same table `ingredients.yml` enriches daily. Before Aug 2026
  it had no group and its Sunday 2am run overlapped the daily 3am one.

`brands.yml`, `image_audit.yml`, `categories.yml` and `rescore.yml` are deliberately outside the
group — they touch neither `data/` state nor the `ingredients` table. See the comments at the top
of each file.

## `enrich-full.yml` — full regeneration

Regenerates all ~26,885 CosIng ingredients through an LLM and upserts them into `ingredients`.
Two things to know before running it:

**It only runs on the first Sunday of the month.** Cron cannot express that: when both
day-of-month and day-of-week are restricted they are OR'd, not AND'd. So the schedule fires every
Sunday and a `guard` job exits early on days 8-31. The guard is a job, not a step, so that
`merge-and-load` skips too — otherwise the shards would produce nothing and the merge would fail
every week. Manual `workflow_dispatch` runs ignore the guard.

**It overwrites `description_es` and `tip_rutina_es`, and that desyncs the translations.**
`load_supabase_v3.py` upserts all 16 columns unconditionally, and those two are the source
`translate_backfill_googletrans.py` translates from. The backfill only fills rows where the target
locale `is.null`, so once `description_en` exists it is never refreshed — after a regeneration the
Spanish text and its en/pt_br/fr translations describe the same ingredient differently, and stay
that way. **This is a known open issue, not a solved one.** Until it is fixed, either avoid the
full regeneration or null out the other locales afterwards so the backfill regenerates them.

**Models**: rotates over OpenRouter's `:free` pool via `utils/openrouter_free.py` in
`incilab-enrich` — never a hardcoded paid slug. The `model` input pins one model for debugging
(losing the fallback); `api_provider: anthropic` is the explicit paid escape hatch.

## Content moderation sweep (`moderate_content` job in `pipeline.yml`)

Backup layer for the real-time moderation in `incilab-web-api` (word filter + async LLM check on every verdict/comment/review). This job re-scans, with an LLM, whatever was published in the last 3 days across `verdicts`, `verdict_comments` and `reviews` — catches anything that slipped through if the web's real-time check failed (e.g. `OPENROUTER_API_KEY` down on Vercel). Flagged content is hidden (`activo=false`) and founders get a notification; it never auto-suspends users (single automated signal, not corroborated by multiple reports). Independent job, no `needs:` — doesn't touch `products_cache`/`ingredients` or local state files, only Supabase community tables.

```bash
python3 incilab_seed.py --task moderate_content                              # last 3 days
python3 incilab_seed.py --task moderate_content --since-days 7 --dry-run     # wider window, no writes
```

## Reclassify categories after schema changes

When new `product_category` values are added (e.g. `essence`), run two steps:

**Step 1 — SQL fix for obvious cases** (run in Supabase SQL editor):
```sql
-- Essences misclassified as serum
UPDATE products_cache
SET product_category = 'essence'
WHERE product_category = 'serum'
  AND (
    product_name ILIKE '%essence%'
    OR product_name ILIKE '%esencia%'
    OR product_name ILIKE '%first essence%'
    OR product_name ILIKE '%ferment essence%'
    OR product_name ILIKE '%treatment essence%'
  )
  AND product_name NOT ILIKE '%sun essence%'
  AND product_name NOT ILIKE '%foam%'
  AND product_name NOT ILIKE '%serum%';
```

**Step 2 — LLM reclassify** (run `categories.yml` manually with `force: true`):
This re-runs the LLM on all products to catch names in Korean or with non-obvious signals.

## Secrets required

| Secret | Description |
|--------|-------------|
| `PRIVATE_REPO_TOKEN` | PAT with `repo` scope — clones the private scripts repo |
| `SUPABASE_URL` | `https://<project>.supabase.co` |
| `SUPABASE_SERVICE_KEY` | Supabase service role key |
| `SUPABASE_PROJECT_ID` | Supabase project ID (used in pipeline.yml) |
| `OPENROUTER_API_KEY` | OpenRouter API key |
| `ANTHROPIC_API_KEY` | Optional — only for `enrich-full.yml` with `api_provider: anthropic` |
| `SERPER_API_KEY` | Optional — enables the reference-image layer in `image_audit.yml` |

Every workflow strips `\n\r` from its secrets before use: a trailing newline in a secret makes
`requests` raise `InvalidHeader` on the `Authorization` header of every call.

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
| `ground-truth-backfill.yml` | On demand | Manual only | Applies `data/ground_truth.json` to rows already in `ingredients`, then chains `--task scores`. Input: `dry_run` (default true) |

## How it works

Each workflow clones the private `incilab-enrich` repo at runtime and runs scripts from it. **To update pipeline logic, push to `incilab-enrich` — no changes needed here.**

**All workflows live in this repo and nowhere else.** `incilab-enrich` used to carry its own
copies (from before the split); they were removed in Aug 2026 because they had drifted — they
still used `actions/checkout@v4` instead of cloning, and never got the `Strip newlines from
secrets` fix or the concurrency group. Three had been disabled by hand in the GitHub UI, which
is invisible from the repo, so anyone reading that directory reasonably assumed all four ran.
If you ever need a workflow there again, remember that **concurrency groups do not span
repositories** — a copy in `incilab-enrich` can never serialize against these.

## Catching up after failures

There is no separate "retry" workflow. `pipeline.yml` already runs every enrichment step
(`retry`, `fill_empty`, `categories`, `reviews`, `generate_htu`, `rescore`), so to catch up on a
backlog you run it manually with `skip_discover`, `skip_jolse` and `skip_images` checked — the
inputs exist for exactly this. `retry.yml` used to be a 236-line copy of those same jobs and was
removed in Aug 2026; it had drifted (its `reviews` step still ran the pre-i18n command and
retried Spanish only).

The one thing it had that `pipeline.yml` lacked, `fill_empty`, is now a step in the
`enrich_ingredients` job. Note that `pipeline.yml` caps reviews at `--limit 150` per locale
while `retry.yml` was uncapped: catching up a large review backlog now takes a few runs, which
is deliberate — the OpenRouter free-tier quota is shared across every workflow.

## `ground-truth-backfill.yml` — correcting rows already in the table

`sanitize_ing()` in `incilab_seed.py` applies the regulatory ground truth to everything
enriched from now on. This workflow is for the rows that were already there — enriched
when the curated lists were smaller, or with `endocrine_disruptor` decided by the LLM
rather than by a regulator. **No LLM involved**: it is a pure transformation against
`data/ground_truth.json`, so it costs nothing and does not touch the OpenRouter quota.

Manual only, and `dry_run` defaults to **true** — it corrects thousands of production
rows at once, so the first run should always be a report you read before applying.

**`--task scores` is chained in the same job, not split into another.** `calcScoreDermico()`
penalises confirmed endocrine disruptors heavily, so moving that column without recomputing
leaves the app showing scores that no longer match their ingredients. That dependency has
been documented in `incilab-db/fixes/fix_disruptors_false_positives.sql` since July 2026.
It is skipped on a dry run, where nothing changed.

The report is gitignored in `incilab-enrich` (196 KB regenerated every run), so the job
uploads it as an artifact — that is the only record of what changed and from which value.

First run, Aug 2026: 1,453 rows corrected, 14,393 scores recomputed.

## Concurrency

`pipeline.yml`, `ingredients.yml`, `enrich-full.yml` and `ground-truth-backfill.yml` share
the `incilab-enrich-writes` group, for two different reasons:

- The first two push state (`data/incilab_failed.json`) back to `incilab-enrich`; running
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

**It no longer writes `description_es` / `tip_rutina_es`** (fixed Aug 2026). Those columns belong
to `incilab_seed.py` (`--task fill_empty` / `routine_tips`), which generates them with a far more
calibrated prompt — and they are the source `translate_backfill_googletrans.py` translates from.
Since that backfill only fills rows where the target locale `is.null`, overwriting the Spanish
used to leave en/pt_br/fr permanently describing the ingredient differently. `load_supabase_v3.py`
now omits both columns from the upsert payload, so Postgres leaves existing values untouched and
new rows arrive `NULL` for `fill_empty` to pick up. Clean split: **this workflow owns the 15
classification and CosIng-metadata columns, the seed owns the content columns.**

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

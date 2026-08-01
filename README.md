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

## How it works

Each workflow clones the private `incilab-enrich` repo at runtime and runs scripts from it. **To update pipeline logic, push to `incilab-enrich` — no changes needed here.**

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

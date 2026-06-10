# incilab-runner

GitHub Actions runner for the InciLab data pipeline.

## Workflows

| Workflow | Schedule | Trigger | Description |
|----------|----------|---------|-------------|
| `pipeline.yml` | Every 3 days | Automatic + manual | Discover products → scrape images → ingredients → categories → reviews → rescore |
| `ingredients.yml` | Daily | Automatic + manual | Fill new ingredients from user searches |
| `categories.yml` | On demand | Manual only | Reclassify `product_category` in `products_cache` using LLM. Input: `force` (bool) — if true, reclassifies all products including those already categorized |
| `rescore.yml` | On demand | Manual only | Recalculate dermico/eco/eficacia scores with algorithm v2 |
| `enrich-full.yml` | 1st Sunday/month | Automatic + manual | Full ingredient regeneration from CosIng via LLM |

## How it works

Each workflow clones the private `incilab-enrich` repo at runtime and runs scripts from it. **To update pipeline logic, push to `incilab-enrich` — no changes needed here.**

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

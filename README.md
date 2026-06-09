# incilab-runner

GitHub Actions runner for the InciLab data pipeline.

## Workflows

| Workflow | Schedule | Description |
|----------|----------|-------------|
| `pipeline.yml` | Every 3 days | Discover products + enrich + rescore |
| `ingredients.yml` | Daily | Fill new ingredients from user searches |
| `enrich-full.yml` | 1st Sunday/month | Full ingredient regeneration from CosIng |

## Secrets required

| Secret | Description |
|--------|-------------|
| `PRIVATE_REPO_TOKEN` | PAT with `repo` scope — clones the private scripts repo |
| `SUPABASE_URL` | `https://<project>.supabase.co` |
| `SUPABASE_SERVICE_KEY` | Supabase service role key |
| `SUPABASE_PROJECT_ID` | Supabase project ID |
| `OPENROUTER_API_KEY` | OpenRouter API key |

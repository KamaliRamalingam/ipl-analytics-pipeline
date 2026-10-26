# IPL Analytics Pipeline

Automated ELT pipeline combining 18 seasons of IPL cricket statistics with Reddit fan sentiment, modelled in Snowflake with dbt and served to a Tableau dashboard.

**Hyper Island · Data Engineering · June 2026 · Pass with Distinction**

![IPL Insights dashboard in Tableau](screenshots/14_tableau_dashboard_final.png)

## Results

| | |
|---|---|
| **Data loaded** | 1,243 IPL matches (2008–2026) and 4,800+ Reddit posts |
| **Warehouse** | Snowflake medallion architecture: 3 RAW tables → staging → 12 MART tables |
| **Transformation** | 14 dbt models, 100 tests passing in the production build |
| **Orchestration** | Two independent Airflow DAGs, 90 pipeline runs, automatic retries |
| **Observability** | Custom `PIPELINE_RUN_LOG` table, with last-run times shown in the dashboard |

## Architecture

![Pipeline architecture](screenshots/38_pipeline_architecture_diagram.svg)

```
Cricsheet (daily)       → Python → Azure Blob raw/ipl/    → Snowflake RAW ─┐
Reddit r/Cricket (15m)  → Python → Azure Blob raw/reddit/ → Snowflake RAW ─┤
                                                                           ↓
                       dbt Cloud: STAGING → FACTS/DIMS → MARTS → Tableau (live connection)
```

## The two pipelines

| | `ipl_pipeline` | `reddit_sentiment_pipeline` |
|---|---|---|
| **Type** | Batch | Micro-batch |
| **Schedule** | Daily at 06:00 Swedish time | Every 15 minutes |
| **Source** | Cricsheet open cricket data (JSON) | r/Cricket posts via the Arctic Shift API (public Reddit archive) |
| **Tasks** | `download_cricsheet` → `parse_json` → `upload_to_blob` → `load_to_snowflake` → `run_dbt` → `log_pipeline_run` | `extract_posts` → `score_sentiment` → `upload_to_blob` → `load_to_snowflake` → `trigger_dbt_job` → `log_pipeline_run` |
| **Extra processing** | Parses nested match and delivery records | VADER sentiment score per post (−1 to +1) |

The DAGs run independently, so a Reddit failure can't block the cricket statistics. Both retry twice on failure and trigger their own dbt Cloud job after loading.

## Data model (dbt)

| Layer | Models |
|---|---|
| **Staging** | `stg_matches`, `stg_deliveries`, `stg_reddit_sentiment` |
| **Dimensions** | `dim_players`, `dim_teams`, `dim_venues` |
| **Facts** | `fct_match_results`, `fct_deliveries` |
| **Marts** | `mart_batting_stats`, `mart_bowling_stats`, `mart_player_career_stats`, `mart_team_performance`, `mart_titles_won`, `mart_ipl_sentiment` |

Every primary and foreign key is tested (`not_null`, `unique`, `accepted_values`), and all tables and columns are documented in YAML.

## Design decisions

| Decision | Why |
|---|---|
| **ELT over ETL** | Raw data stays in Snowflake for reprocessing; transformations are versioned in dbt |
| **Micro-batch over streaming** | API rate limits rule out streaming, and a 15-minute lag is acceptable for sentiment |
| **Airflow over cron** | Task dependencies, automatic retries and run history in one place |
| **Snowflake over Postgres** | Separate compute and storage, and native dbt integration |
| **dbt over raw SQL** | Version control, built-in tests, documentation and lineage |
| **Live connection over extract** | The dashboard always shows current data, including pipeline freshness |

## Repository structure

```
dags/              Airflow DAGs (ipl_pipeline, reddit_sentiment_pipeline)
ingestion/         Cricsheet download, parsing, Blob upload, Snowflake load
include/reddit/    Reddit extraction, VADER scoring, Blob upload, Snowflake load
dbt/               dbt project: staging, dimensions, facts, marts, tests
snowflake/         Database setup and PIPELINE_RUN_LOG table
tests/             pytest tests for parsing and upload
screenshots/       Evidence of runs: Airflow, dbt Cloud, Snowflake, Azure, Tableau
docs/              Build specification
```

## Run it yourself

Requires the [Astro CLI](https://www.astronomer.io/docs/astro/cli/overview), Docker, and accounts for Snowflake, Azure Blob Storage and dbt Cloud.

1. Run `snowflake/setup.sql` and `snowflake/pipeline_run_log.sql` in Snowflake.
2. Copy `.env.example` to `.env` and fill in your Snowflake, Azure and dbt Cloud credentials.
3. Copy `dbt/profiles.yml.example` to `dbt/profiles.yml` if you run dbt locally.
4. Start Airflow with `astro dev start`, then open `localhost:8080` and enable both DAGs.
5. Run the tests with `pytest tests/`.

## Project status

Completed in June 2026. The cloud accounts used for the project have since been closed, so the pipeline is no longer running. The `screenshots/` folder documents the scheduled runs, dbt builds, warehouse tables and final dashboard.

## What I'd improve next

- Run the pipelines under a dedicated Snowflake service user with key-pair authentication
- Make the large dbt models incremental instead of full rebuilds
- Add CI to run `pytest` and `dbt build` on every pull request
- Compare VADER with a cricket-specific sentiment model

## Built with

Python · Apache Airflow (Astro) · Azure Blob Storage · Snowflake · dbt Cloud · Tableau · VADER · GitHub

Built with AI-assisted development (Claude Code). The specification I wrote for it is in [`docs/build-spec.md`](docs/build-spec.md).

# Mini Compass

A small, practice version of a business analytics platform: data is loaded into Google BigQuery, cleaned and tested with dbt, and shown on a dashboard, with access rules that decide who can see what.

It is modelled on the kind of platform a grant-making organisation might use to bring grant, finance and people data into one secure place. It uses sample and public data only. No real personal or financial data is stored in this repository.

## Status

| Part | Status |
|---|---|
| BigQuery project and dbt connection (keyless login) | Done |
| dbt jaffle shop: seeds, staging models, marts, tests | Done |
| Looker Studio dashboard | Planned |
| Python pipeline: public API to BigQuery | Planned |
| Scheduling with GitHub Actions | Planned |
| dbt models and tests on pipeline data | Planned |
| Role-based access in BigQuery | Planned |
| LibreChat in Docker, with a security risk review | Planned |
| Runbook | Planned |

## How it fits together

```
Source (API or CSV)
      |
      v
Python pipeline  ->  raw tables in BigQuery
                          |
                          v
                 dbt: staging models -> final tables (with tests)
                          |
                          v
                 Dashboards and AI chat (role-based access)
```

## Tech stack

- Google BigQuery (sandbox, EU location)
- dbt Core with the `dbt-bigquery` adapter
- Python 3.11, managed with `uv`
- Git and GitHub (GitHub Actions planned)
- Looker Studio (planned)
- Docker and LibreChat (planned)

## Project structure

```
mini-compass/
├── dbt/
│   └── jaffle_shop/        # dbt project (seeds, models, tests)
├── pipelines/              # Python pipelines (planned)
├── docs/                   # screenshots, runbook (planned)
├── .gitignore
└── README.md
```

## Setup

### 1. Prerequisites

- A Google account and a Google Cloud project with the BigQuery sandbox enabled
- The [gcloud CLI](https://cloud.google.com/sdk/docs/install)
- Python 3.11 and [uv](https://docs.astral.sh/uv/)
- Git

### 2. Install

```bash
git clone <your-repo-url> mini-compass
cd mini-compass
uv venv --python 3.11 .venv
source .venv/bin/activate
uv pip install dbt-core dbt-bigquery
```

### 3. Log in to Google (no key file)

```bash
gcloud auth application-default login
gcloud config set project <your-project-id>
gcloud auth application-default set-quota-project <your-project-id>
```

On the Google consent page, tick every permission box, including **Google Cloud data**. Without it, login fails.

### 4. Configure dbt

Create `~/.dbt/profiles.yml`. This file lives outside the repository on purpose, so connection settings are never committed.

```yaml
jaffle_shop:
  target: dev
  outputs:
    dev:
      type: bigquery
      method: oauth
      project: <your-project-id>
      dataset: jaffle_dev
      threads: 4
      location: EU
```

### 5. Build and test

```bash
cd dbt/jaffle_shop
dbt debug    # should end with "All checks passed!"
dbt seed     # load raw CSVs into BigQuery
dbt run      # build the models
dbt test     # run the data quality tests
```

To browse the lineage graph:

```bash
dbt docs generate
dbt docs serve
```

## What the dbt project contains

- **Seeds** (`seeds/`): raw customers, orders and payments as CSV files. They stand in for data that a pipeline would normally load.
- **Staging models** (`models/staging/`): one model per raw table. They rename columns and fix types, and contain no business logic.
- **Final models** (`models/customers.sql`, `models/orders.sql`): join the staging models into tables that are ready to analyse.
- **Tests** (`schema.yml`): `unique`, `not_null`, `accepted_values` and `relationships` checks that catch bad data early.

## Security decisions

- **No key files.** Authentication uses `gcloud` application default credentials (OAuth), so there is no service account key to leak.
- **Connection settings stay out of Git.** `profiles.yml` lives in `~/.dbt/`, and `.gitignore` excludes `.env`, `*.json`, `target/`, `logs/` and `.venv/`.
- **No real data in the repository.** Seeds are sample data only. Real data belongs in BigQuery, loaded by a pipeline, with access rules applied.
- **Dashboard credentials.** Looker Studio data sources should use **viewer's credentials**, so each person's BigQuery permissions apply instead of the owner's.
- **Planned:** service accounts with least privilege for GitHub Actions, secrets in GitHub Secrets, and row and column level access in BigQuery.

## Roadmap

1. Looker Studio dashboard on `orders` and `customers`, with a screenshot in `docs/`.
2. Python pipeline that loads a public API into a raw BigQuery table, with retries, logging and error handling.
3. GitHub Actions workflow to run the pipeline on a schedule.
4. dbt staging model and tests on the pipeline data.
5. Two demo roles (for example finance and grants) with different access to tables, using BigQuery row access policies and policy tags.
6. LibreChat running in Docker with secrets in environment variables, plus a written review of the risks (file uploads, code execution, MCP tools) and how each is reduced.
7. A short runbook: what to do when a pipeline fails, how to rotate credentials, and how to add a new data source.

## Acknowledgements

The dbt project is based on dbt Labs' [jaffle_shop](https://github.com/dbt-labs/jaffle_shop) sample project.

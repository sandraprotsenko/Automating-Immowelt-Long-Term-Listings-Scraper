# Automating-Immowelt-Long-Term-Listings-Scraper
A clean, production‑ready repository template for your webeet project that scrapes, cleans, enriches, and loads long‑term rental listings from Immowelt into a database on a schedule (AWS Lambda‑first).

---

Architecture (high‑level)
Fetch: request Immowelt search pages / APIs and parse listings
Transform & Clean: normalize schema, deduplicate, type/price conversions
Enrich: geocode (optional), add derived metrics
Load: upsert into Postgres (or write to S3/Parquet)
Schedule: AWS EventBridge → Lambda → lambda_function.py

bash```
EventBridge → Lambda → src/immowelt_scraper → (Fetch → Transform → Enrich → Load) → DB/S3 ```

---
## Repository layout

bash
```
immowelt-longterm-scraper/
├── src/
│ └── immowelt_scraper/
│ ├── __init__.py
│ ├── config.py # env handling & constants
│ ├── fetch_immowelt.py # scraping logic
│ ├── transform_clean.py # cleaning/normalization
│ ├── geocode_enrich.py # optional enrichment
│ ├── upload_to_db.py # upsert to Postgres
│ └── cli.py # local entrypoints (click/argparse)
│
├── lambda/
│ ├── lambda_function.py # AWS Lambda handler (thin wrapper)
│ └── requirements.txt # minimal deps for Lambda package
│
├── sql/
│ ├── create_tables.sql
│ ├── upsert_listings.sql
│ ├── check_duplicates.sql
│ └── stats.sql
│
├── scripts/
│ ├── run_local.sh # sample local run
│ └── package_lambda.sh # build zip for Lambda
│
├── notebooks/ # (optional) exploration
│ └── 00_exploration.ipynb
│
├── .github/workflows/
│ ├── ci.yml # lint/test
│ └── deploy.yml # (optional) auto‑package artifact
│
├── tests/
│ ├── test_fetch.py
│ └── test_transform.py
│
├── .env.example
├── .gitignore
├── LICENSE
├── pyproject.toml # or requirements.txt + setup.cfg
├── README.md
└── Makefile
```

---

## Minimal Lambda handler (example)
bash
```
# lambda/lambda_function.py
from immowelt_scraper.cli import run_once

# AWS Lambda entrypoint
def handler(event, context):
# you can route by event to choose city/category if needed
result = run_once()
return {"statusCode": 200, "body": str({"inserted": result.inserted, "updated": result.updated})}

```
----

## CLI entrypoint (example)

bash
```
# src/immowelt_scraper/cli.py
import os
from .fetch_immowelt import fetch_listings
from .transform_clean import clean
from .geocode_enrich import enrich
from .upload_to_db import upsert

class RunResult: # tiny helper for reporting
def __init__(self, inserted=0, updated=0):
self.inserted = inserted
self.updated = updated

def run_once() -> RunResult:
raw = fetch_listings(query=os.getenv("IMMOWELT_QUERY", "berlin-mitte"))
df = clean(raw)
if os.getenv("ENABLE_GEOCODE", "0") == "1":
df = enrich(df)
return upsert(df)

if __name__ == "__main__":
run_once()

```
----

## Environment variables


bash
```
# database
DB_HOST=...
DB_PORT=5432
DB_NAME=...
DB_USER=...
DB_PASSWORD=...

# scraping
IMMOWELT_QUERY="berlin-mitte" # or a full query string
USER_AGENT="Mozilla/5.0 ..."
REQUEST_DELAY_MS=250

# features
ENABLE_GEOCODE=0
GEOCODER_PROVIDER="nominatim"
GEOCODER_API_KEY=

# sinks
S3_BUCKET=
S3_PREFIX=immowelt/longterm/

```
----

## Local development

bash
```
# Python 3.11 recommended
python -m venv .venv && source .venv/bin/activate

# Option A: Poetry
poetry install

# Option B: pip
pip install -r requirements.txt

# run
python -m immowelt_scraper.cli

```
----

## Makefile (targets)

bash
```
install:
pip install -r requirements.txt

lint:
ruff check src tests

test:
pytest -q

run:
python -m immowelt_scraper.cli

package-lambda:
bash scripts/package_lambda.sh

```
----

## Packaging for AWS Lambda
Two good options:
Zip with vendored deps (simple):
pip install -r lambda/requirements.txt -t lambda_pkg/
copy src/ package into lambda_pkg/
add lambda/lambda_function.py
cd lambda_pkg && zip -r ../lambda.zip .
upload lambda.zip to the function
Lambda Layer (cleaner): put heavy deps (e.g., pandas, psycopg2-binary) into a Layer; keep handler zip tiny.
lambda/requirements.txt (minimal):

bash
```
requests
pandas
psycopg2-binary
boto3
python-dotenv
```

----

CI (GitHub Actions)
.github/workflows/ci.yml (example):

bash
```
name: CI
on: [push]
jobs:
test:
runs-on: ubuntu-latest
steps:
- uses: actions/checkout@v4
- uses: actions/setup-python@v5
with: { python-version: '3.11' }
- run: pip install -r requirements.txt
- run: ruff check src tests
- run: pytest -q
```
(Optional) deploy.yml can build lambda.zip as an artifact on main.
____

## SQL model (example fields)
listing_id (PK), title, desc, price_eur, rooms, area_sqm, address, city, lat, lon, url, source, first_seen_at, last_seen_at.
Upsert by listing_id + source. Maintain first_seen_at, update last_seen_at and mutable fields.

---
##Migration from your current repo (quick plan)
Create new root folder immowelt-longterm-scraper.
Move code from old sources/ and duplicate src/ into src/immowelt_scraper/ (keep only one src).
Keep working SQL files and place them under sql/.
Choose one dependency file: prefer pyproject.toml (Poetry) or requirements.txt + setup.cfg. For Lambda, keep a focused lambda/requirements.txt.
Replace your Lambda entry with the minimal handler above and import your package logic.
Test locally, then package and upload.

## Roadmap

## License

MIT (or choose your preferred).

Badges (optional)

Add later: CI status, Python version, License, Code style (ruff), Coverage.










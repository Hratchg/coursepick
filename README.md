# CoursePick

**Find the right professor for any UCSB course.** Search a course and CoursePick ranks every professor who teaches it by a 0 to 100 Gaucho Score, built from 15+ years of official grade distributions and RateMyProfessors reviews.

**Live:** [coursepick.app](https://coursepick.app)

![CoursePick demo: search a course, a book leaves the 3D library, professors are ranked with grade distributions](docs/media/coursepick-demo.gif)

> This is a public showcase copy of the CoursePick codebase. Production runs from a private repository with its scheduled data jobs and deploy secrets.

## What's interesting

- **Messy-data matching.** Official grade CSVs list instructors as `LOKSHTANOV D`; RateMyProfessors has full names. A 4-pass matcher (initials, fuzzy, department disambiguation, dedup) joins the two.
- **Scoring you can inspect.** Gaucho Score combines average GPA, RMP quality, difficulty and review sentiment (VADER + TF-IDF keywords). The [methodology page](https://coursepick.app/methodology) explains every factor and weight.
- **A frontend with some personality.** React + three.js: the landing page is a 3D sphere of books, and the course you pick flies out of it and opens into the professor rankings.
- **Production habits.** FastAPI on Render, Postgres on Neon, Alembic migrations, a migration-drift check, nightly schedule sync and weekly backups as GitHub Actions, and a pytest + Vitest suite.

## Architecture

```mermaid
flowchart LR
  A[UCSB grade CSVs] --> E[ETL: match, NLP, score]
  B[RateMyProfessors GraphQL] --> E
  C[UCSB schedule API] --> E
  E --> D[(Postgres / Neon)]
  D --> F[FastAPI on Render]
  F --> G[React + three.js on Vercel]
```

## Tech Stack

- **Frontend:** React, TypeScript, three.js (react-three-fiber), Tailwind, Vite, Vitest
- **API:** FastAPI, SQLAlchemy, Alembic
- **Database:** PostgreSQL 16 (Neon in production)
- **Data:** curl_cffi (RMP GraphQL), pandas, TheFuzz, VADER + TF-IDF
- **Ops:** GitHub Actions (CI, scheduled syncs, backups), Docker Compose for local dev

## Prerequisites

- Python 3.12+
- Docker Desktop (for PostgreSQL)
- A populated PostgreSQL database (see Quick Start)

## Quick Start

```bash
# Clone
git clone https://github.com/Hratchg/coursepick.git
cd coursepick

# Configure
cp .env.example .env
# Edit .env with your DATABASE_URL

# Launch
docker compose up

# Visit http://localhost:8501
```

## Pipeline

```bash
python scripts/run_pipeline.py              # full pipeline (scrape -> match -> NLP -> score)
python scripts/run_pipeline.py --scrape     # targeted RMP scrape only
python scripts/run_pipeline.py --match      # enhanced professor matching only
python scripts/run_pipeline.py --nlp        # NLP sentiment/keywords only
python scripts/run_pipeline.py --score      # Gaucho Score computation only
```

## Running Tests

```bash
# Install dev dependencies (includes pytest)
pip install -r requirements-dev.txt

# Start a throwaway PostgreSQL (any free local port works)
docker run -d --rm --name gco-ci-pg -e POSTGRES_USER=gco -e POSTGRES_PASSWORD=gco \
  -e POSTGRES_DB=gco_test -p 5445:5432 postgres:16

# Run the full test suite against it
DATABASE_URL=postgresql://gco:gco@localhost:5445/gco_test pytest -v
```

Tests use savepoint-based transactions that roll back after each test, but the session **creates every table at the start and drops every table at the end**, so only ever point them at a throwaway database:

- `DATABASE_URL` comes from the environment only; the tests do not read `.env`, which holds the app's own (production) database. Without it, they use `postgresql://gco:gco@localhost:5432/gco_test`, which is what CI provides.
- pytest refuses to start unless `DATABASE_URL`'s host is `localhost`, `127.0.0.1` or `::1`, and names the host it refused. Set `ALLOW_REMOTE_TEST_DB=1` to override that, only for a disposable remote database.

## Database Management

Export and import scripts are provided for backing up and restoring the database:

```bash
# Export the database to a SQL dump
bash scripts/export_db.sh

# Import a SQL dump into the database
bash scripts/import_db.sh
```

## Project Structure

```
├── scrapers/           # RMP GraphQL scraper + grade CSV ingester
├── etl/                # Name matching, enhanced matching, NLP, scoring
│   ├── name_matcher.py        # TheFuzz fuzzy name matching
│   ├── name_utils.py          # Nexus name parsing (initials, dedup)
│   ├── department_mapper.py   # UCSB dept code <-> RMP dept name mapping
│   ├── enhanced_matcher.py    # 4-pass local matching engine
│   ├── nlp_processor.py       # VADER sentiment + TF-IDF keywords
│   └── scoring.py             # Gaucho Value Score computation
├── db/                 # SQLAlchemy models + Alembic migrations
├── api/                # FastAPI app (courses, professors, quarters, health)
├── frontend/           # React + three.js site (coursepick.app)
├── dashboard/          # Original Streamlit prototype
├── scheduler/          # Job bodies the GitHub Actions workflows call (APScheduler for local runs)
├── scripts/            # CLI pipeline runner + DB export/import
├── tests/              # pytest suite
├── docs/               # PRD, design docs, plans
├── docker-compose.yml
└── pyproject.toml
```

## Data Sources

- **Grades:** [Daily Nexus Grade Distributions](https://github.com/dailynexusdata/grades-data) — Fall 2009 through Fall 2025
- **Reviews:** RateMyProfessors via internal GraphQL API (UCSB school ID: 1077)

## Documentation

- [Product Requirements](docs/PRD.md)
- [Design Document](docs/plans/2026-02-25-gaucho-course-optimizer-design.md)
- [Targeted RMP Pipeline](docs/plans/2026-02-25-targeted-rmp-pipeline-implementation.md)
- [Enhanced Matching Pipeline](docs/plans/2026-02-26-enhanced-matching-implementation.md)
- [Production Deployment](docs/plans/prod.md)

## License

[MIT](LICENSE)

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This directory contains only the Docker Compose infrastructure for running a local MIMIC-IV PostgreSQL database. The NLP coursework notebooks and Python code live in the sibling `Week6/` directory.

## Database Configuration

- **Image**: PostgreSQL 17
- **Container name**: `mimic_postgres`
- **Host**: `localhost:5432`
- **Database**: `mimiciv`
- **User/Password**: `mimicuser` / `mimicpass`
- **Web UI (Adminer)**: `http://localhost:28080`

## Docker Compose Services

| Service | Profile | Purpose |
|---------|---------|---------|
| `db` | (default) | PostgreSQL 17 — always running |
| `schema-gen` | (default) | One-shot: creates MIMIC-IV schemas |
| `loader` | (default) | One-shot: loads CSV data into tables |
| `adminer` | (default) | Web DB admin UI at port 28080 |
| `indexer` | `post-load` | One-shot: creates indexes |
| `constraints` | `post-load` | One-shot: adds foreign keys |
| `concepts` | `post-load` | One-shot: creates derived materialized views |
| `schema-note` | `note` | One-shot: creates `mimiciv_note` schema |
| `loader-note` | `note` | One-shot: loads discharge/radiology note CSVs |
| `validate-note` | `tools` | Validates note row counts |
| `verify` | `tools` | Prints row counts across all schemas |

## Setup Sequence

```bash
# 1. Start the database and load core MIMIC-IV data
docker compose up -d db schema-gen loader

# 2. (Optional) Create indexes, constraints, and derived concepts
docker compose --profile post-load run --rm indexer
docker compose --profile post-load run --rm constraints
docker compose --profile post-load run --rm concepts

# 3. Load clinical notes (required for NLP notebooks in Week6/)
docker compose --profile note run --rm schema-note
docker compose --profile note run --rm loader-note

# 4. Verify data loaded correctly
docker compose --profile tools run --rm verify
docker compose --profile tools run --rm validate-note
```

## Data Directory Layout

```
data/
├── core/      # MIMIC-IV core CSVs (patients, admissions, transfers)
├── hosp/      # MIMIC-IV hospital CSVs (diagnoses, labs, prescriptions, etc.)
├── icu/       # MIMIC-IV ICU CSVs (chartevents, inputevents, etc.)
├── note/      # MIMIC-IV-Note CSVs (discharge.csv.gz, radiology.csv.gz, etc.)
└── db/        # PostgreSQL data directory (persisted on host)
```

## Notes

- The `db` service healthcheck waits up to 120s for PostgreSQL to be ready before dependent services start.
- The `loader` skips tables that already contain data — safe to re-run.
- `schema-gen`, `loader`, `schema-note`, `loader-note` are all one-shot containers; after they exit successfully they do not need to run again.
- The `postgres_buid/` directory holds `create.sql`, `index.sql`, and `constraint.sql`; `postgres_build_note/` holds the note schema DDL and load scripts.

# From Bedside to Model: A Self-Study Tutorial on MIMIC-IV Derived Concepts

This repository is a **self-learning tutorial** for clinicians and healthcare professionals
who want to understand how to apply machine learning to real ICU data — without months of
data engineering work.

The central insight is that the [MIMIC-IV derived concepts](https://github.com/MIT-LCP/mimic-code/tree/main/mimic-iv/concepts_postgres)
layer — peer-reviewed SQL scripts maintained by MIT Laboratory for Computational Physiology —
pre-calculates every major clinical score and feature table from raw EHR data. You get
SOFA scores, Charlson comorbidity indices, sepsis criteria, vasopressor burdens, and much
more, all ready to query with a single SQL join.

This tutorial shows you how to use those pre-built tables to go from raw MIMIC-IV data to
a working ICU mortality prediction model in an afternoon.

---

## The Core Idea

```
Raw EHR data (lab values, chart events, billing codes)
        ↓  [~65 peer-reviewed SQL scripts run once]
mimiciv_derived schema (SOFA, Charlson, Sepsis-3, KDIGO, ...)
        ↓  [one SQL query]
Feature matrix  →  ML model  →  Clinical insight
```

Two concept tables power the main tutorial:

| Concept | Table | Measures | Key range |
|---------|-------|----------|-----------|
| **SOFA Score** | `mimiciv_derived.first_day_sofa` | Acute organ failure in first 24 h of ICU | 0 – 24 |
| **Charlson Comorbidity Index** | `mimiciv_derived.charlson` | Chronic disease burden | 0 – 37 |

SOFA answers *"how sick is this patient right now?"*
Charlson answers *"what is their baseline physiological reserve?"*
Together, they predict in-hospital ICU mortality with AUC ~0.79 — capturing most of the
signal that complex models achieve with dozens of variables.

---

## Repository Contents

| File | Purpose |
|------|---------|
| `concepts_tutorial.ipynb` | **Main tutorial** — end-to-end walkthrough for clinicians |
| `concepts_tutorial.html` | Static rendered version of the tutorial (no database needed) |
| `Concepts_Self_Study_Tutorial.pptx` | Slide deck — conceptual overview and teaching guide |
| `concepts.ipynb` | Broader explorer — all 65+ derived concept tables with examples |
| `build_mimic.ipynb` | Database build pipeline — load MIMIC-IV into local PostgreSQL |
| `test_mimic.ipynb` | Integration tests — verify data integrity after loading |
| `TUTORIAL.md` | Full setup guide — prerequisites, data download, pipeline steps |

---

## What the Tutorial Covers

The notebook (`concepts_tutorial.ipynb`) walks through seven steps, each with clinical
framing and annotated Python code:

| Step | Clinical framing | Technical skill |
|------|------------------|-----------------|
| 1 | Connect Python to a medical database | Database connection with `psycopg` |
| 2 | Understand SOFA and Charlson as *features* | SQL joins across schemas |
| 3 | Build a labeled ICU cohort | Multi-table SQL query |
| 4 | Explore score distributions by survival | Exploratory data analysis |
| 5 | Handle missing scores from incomplete charts | Median imputation (no data leakage) |
| 6 | Train a model to predict in-hospital mortality | Random Forest classifier |
| 7 | Interpret model performance clinically | AUC-ROC, feature importance |

No prior Python or ML experience is assumed — every code pattern is explained.

---

## Prerequisites

### 1. PhysioNet Access

MIMIC-IV requires credentialed access:
1. Create an account at [physionet.org](https://physionet.org)
2. Complete the required CITI training course
3. Request access to [MIMIC-IV](https://physionet.org/content/mimiciv/)

### 2. Docker Desktop

Required to run the PostgreSQL container:
- **macOS**: [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/)
- **Linux**: [Docker Engine](https://docs.docker.com/engine/install/)

### 3. uv (Python package manager)

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
# Or: brew install uv
```

---

## Quick Start

### Step 1 — Clone and set up

```bash
git clone https://github.com/marcofanti/LearningProject2.git
cd LearningProject2

# Clone the MIT-LCP build scripts
git clone https://github.com/MIT-LCP/mimic-code.git

# Install Python dependencies
uv sync
```

### Step 2 — Configure

```bash
cp .env.local .env
# Edit .env: set POSTGRES_PASSWORD and MIMIC_DATA_DIR
```

### Step 3 — Load MIMIC-IV and build derived concepts

```bash
# Run the full pipeline (loads data + builds ~65 derived concept tables)
uv run python build_mimic.py
```

> **Note:** The initial data load takes 4–12 hours (the `chartevents` table has ~433 million rows).
> Building the derived concepts schema takes an additional 15–60 minutes.
> See [TUTORIAL.md](TUTORIAL.md) for full details and troubleshooting.

### Step 4 — Run the tutorial

```bash
uv run jupyter lab concepts_tutorial.ipynb
```

If you want to explore all derived concepts first:

```bash
uv run jupyter lab concepts.ipynb
```

---

## Database Connection

Once the database is running, connect with:

```python
import psycopg

conn = psycopg.connect(
    host="localhost", port=5432,
    dbname="mimiciv", user="mimicuser", password="mimicpass",
)
```

Web admin UI (Adminer): [http://localhost:28080](http://localhost:28080)

---

## Why Derived Concepts Matter

| Without derived concepts | With derived concepts |
|--------------------------|----------------------|
| Pull raw labs from `labevents`, `chartevents`, `outputevents` | Single `LEFT JOIN` |
| Implement 24-hour ICU timing windows | Already handled |
| Convert units, impute sub-scores | Already handled |
| Validate SQL against published clinical definition | Already peer-reviewed |
| Weeks of preprocessing | Minutes of SQL |

The `mimiciv_derived` schema is built from the [MIT-LCP mimic-code](https://github.com/MIT-LCP/mimic-code)
repository and covers severity scores (SOFA, SAPS II, OASIS), sepsis criteria, blood gas
analysis, AKI staging (KDIGO), vasopressor dosing, antibiotic exposure, comorbidity indices,
and more. Full description: [Pollard et al., JAMIA 2018](https://academic.oup.com/jamia/article/25/1/32/4259424).

---

## Verify Your Setup

After loading, run the integration test suite:

```bash
uv run python test_mimic.py
```

Expected output:
```
══════════════════════════════════════════════════════════════
MIMIC-IV PostgreSQL Integration Tests
══════════════════════════════════════════════════════════════
  PASS   d_labitems: blood gas item IDs + labels
  PASS   inputevents: vasopressor units
  PASS   all concept tables have ≥1 row
  PASS   sofa: unique (stay_id, hr)
  PASS   sepsis3: unique stay_id
  ...
══════════════════════════════════════════════════════════════
```

---

## License and Data Access

- Code in this repository: MIT License
- MIMIC-IV data: requires PhysioNet credentialing and CITI human subjects training
- Built on [MIMIC-IV](https://physionet.org/content/mimiciv/) (Johnson et al., 2023)

For full setup documentation, see [TUTORIAL.md](TUTORIAL.md).

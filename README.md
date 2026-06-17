# Healthcare SQL Pipeline

A PostgreSQL data pipeline that transforms a raw healthcare dataset into a clean, normalized, analytics-ready database using a **Bronze → Silver → Gold** medallion architecture.

---

## Objective

The goal of this project is to practice real-world data engineering and SQL analytical skills by building a complete pipeline around a single dataset, from raw ingestion all the way to business-ready insights. Specifically, this lab is designed to:
- Practice ingesting raw, messy, real-world data into a relational database 
- Apply database normalization principles (3NF) to design a clean relational schema 
- Build comfort with core PostgreSQL operations: schema creation, constraints, foreign keys, and data loading
- Progress toward writing analytical SQL queries using joins, CTEs, aggregations, and window functions
- Document and structure a data project the way it would be organized in a real analytics/data engineering role

This project uses the [Healthcare Dataset by prasad22](https://www.kaggle.com/datasets/prasad22/healthcare-dataset) from Kaggle, which contains hospital admission records including patient demographics, doctors, hospitals, insurance providers, billing, and medical outcomes.

---

## Tools Used

| Tool | Purpose |
|---|---|
| **PostgreSQL** | Core relational database used to store and query all data |
| **SQL** | Schema design, data transformation, and analytical querying |
| **Kaggle Dataset** | Source of raw healthcare data (CSV) |
| **psql / pgAdmin** | Database interaction and script execution |
| **Git & GitHub** | Version control and project documentation |

---

## Schema Design — Medallion Architecture

This project follows the **medallion architecture** pattern, which organizes data into three progressive layers, each with a distinct purpose. Rather than transforming raw data in one single step, the data moves through layers of increasing structure and quality.

| Layer | Purpose | What it contains |
|---|---|---|
| **Bronze** | Raw ingestion | An untouched, exact copy of the source CSV. No cleaning, no transformations. This is the permanent source of truth. |
| **Silver** | Cleaned & normalized | The raw data broken into a proper relational structure (3NF), with redundancy removed and relationships enforced via foreign keys. |
| **Gold** | Business-ready aggregates | Curated views/tables built to directly answer specific analytical questions (e.g. average billing by condition, admissions by month). |

Each layer lives in its **own PostgreSQL schema** (`bronze`, `silver`, `gold`), so the database structure itself documents the data's journey from raw to refined.

Separating raw data from cleaned data means the original source is never lost or overwritten — if a transformation step turns out to be wrong, Silver and Gold can always be rebuilt from Bronze without re-downloading anything. It's the same pattern used in real production data pipelines.

The **Silver layer schema** (the normalized design) looks like this:

- `patients` — one row per unique patient (demographics)
- `doctors` — lookup table of unique doctors
- `hospitals` — lookup table of unique hospitals
- `insurance_providers` — lookup table of unique insurance companies
- `admissions` — the central fact table; one row per hospital stay, referencing all four tables above via foreign keys

This eliminates the repetition present in the original flat CSV (e.g. the same doctor's name no longer needs to be re-typed for every admission) and sets up the database for efficient, JOIN-based analytical queries in the Gold layer.

---

## Bronze Stage — Walkthrough

The Bronze stage is the foundation of the pipeline: getting the raw dataset into PostgreSQL **exactly as it is**, with zero transformation. The purpose of this stage is purely ingestion — preserving an unaltered copy of the source data before any cleaning or restructuring happens.

### Step 1 — Reset the schema (safe to re-run)

Since this project is still actively being developed, the Bronze schema is dropped and recreated at the start of the script. This avoids errors from re-running the script multiple times during development, and guarantees a clean slate every time.

```sql
DROP SCHEMA IF EXISTS bronze CASCADE;
CREATE SCHEMA bronze;
```

### Step 2 — Create the staging table

A single staging table was created to mirror the raw CSV's structure column-for-column. No primary keys, no foreign keys, no constraints — this table's only job is to hold the data exactly as it exists in the source file.

```sql
CREATE TABLE bronze.staging_healthcare (
    name VARCHAR(100),
    age INT,
    gender VARCHAR(10),
    blood_type VARCHAR(5),
    medical_condition VARCHAR(100),
    admission_date DATE,
    doctor VARCHAR(100),
    hospital VARCHAR(150),
    insurance_provider VARCHAR(100),
    billing_amount DECIMAL(10,2),
    room_number INT,
    admission_type VARCHAR(20),
    discharge_date DATE,
    medication VARCHAR(100),
    test_results VARCHAR(20)
);
```

### Step 3 — Load the CSV into the staging table

The raw CSV file was loaded directly into `bronze.staging_healthcare` using PostgreSQL's `COPY` command.

```sql
COPY bronze.staging_healthcare
FROM '/path/to/healthcare_dataset.csv'
DELIMITER ','
CSV HEADER;
```

### Outcome of the Bronze Stage

At the end of this stage, `bronze.staging_healthcare` contains a complete, unmodified copy of the Kaggle dataset inside PostgreSQL. This table is treated as **read-only** going forward — it is never edited or cleaned directly. Instead, it serves as the single source that the Silver layer will be built from, meaning the pipeline can always be re-run from scratch without needing to re-download or re-inspect the original file.

---

All scripts referenced in this section can be found in [`/scripts/bronze`](./scripts/bronze).

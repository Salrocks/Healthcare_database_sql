# 🏥 Healthcare SQL Pipeline — Full Project Walkthrough

A PostgreSQL data pipeline that transforms a raw healthcare dataset into a clean, query-ready database and a set of business-facing analytical views, using a **Bronze → Silver → Gold** medallion architecture.

---

## 🗺️ Roadmap — What Happens at Each Layer

Before diving into the walkthrough, here's a high-level map of what each layer is responsible for, used as a quick reference throughout this project.

| Task | Source | Bronze | Silver | Gold |
|---|---|---|---|---|
| **Data Validation?** | N/A — raw external file | Yes, confirm all data imported correctly | Yes | Yes |
| **Import Data?** | No | Yes, from Kaggle | Yes, from `bronze.staging_table` | Yes, from `silver.healthcare_table` |
| **Transform Data?** | No | No, 1:1 with source | Yes — Data Cleaning, Standardization/Normalization of Values, Parsing/Feature Extraction, Null Handling | No |
| **Tables or Views?** | N/A | Tables | Tables | Views |
| **Data lives in which Schema?** | N/A | Bronze | Silver | Gold |
| **Load Type** | N/A | Bulk | Truncate-and-reload (full refresh) | N/A (views, not materialized) |

---

## 🎯 1. Objective of the Lab

The goal of this project is to practice real-world data engineering and SQL analytical skills by building a complete pipeline around a single dataset, from raw ingestion all the way to business-ready insights. Specifically, this lab was designed to:

- Practice ingesting raw, messy, real-world data into a relational database
- Apply data cleaning techniques: string parsing, regular expressions, null/negative handling, and standardization
- Build comfort with core PostgreSQL operations: schemas, staging tables, stored procedures, and views
- Practice analytical SQL using aggregations, `CASE` logic, window functions, CTEs, and date functions
- Document and structure a data project the way it would be organized in a real analytics/data engineering role

This project uses the [Healthcare Dataset by prasad22](https://www.kaggle.com/datasets/prasad22/healthcare-dataset) from Kaggle, which contains hospital admission records including patient demographics, doctors, hospitals, insurance providers, billing, and medical outcomes.

---

## 🛠️ 2. Tools Used

| Tool | Purpose |
|---|---|
| **PostgreSQL** | Core relational database used to store, transform, and query all data |
| **SQL / plpgsql** | Data cleaning, stored procedures, views, and analytical querying |
| **Kaggle Dataset** | Source of raw healthcare data (CSV) |
| **psql / pgAdmin** | Database interaction and script execution |
| **Git & GitHub** | Version control and project documentation |

---

## 🏗️ 3. Schema Design — Medallion Architecture

This project follows the **medallion architecture** pattern, which organizes data into three progressive layers, each with a distinct purpose. Rather than transforming raw data in one single step, the data moves through layers of increasing structure and quality.

| Layer | Purpose | What it contains |
|---|---|---|
| **Bronze** | Raw ingestion | An untouched, exact copy of the source CSV. No cleaning, no transformations. The permanent source of truth. |
| **Silver** | Cleaned & standardized | A single cleaned table — names parsed, hospital names standardized, billing amounts corrected. Fully query-ready. |
| **Gold** | Business-ready analytics | Eight views built on top of Silver, each answering a specific analytical question. |

Each layer lives in its own PostgreSQL schema (`bronze`, `silver`, `gold`), so the database structure itself documents the data's journey from raw to refined.

Separating raw data from cleaned data means the original source is never lost or overwritten. If a transformation step turns out to be wrong, Silver and Gold can always be rebuilt from Bronze without re-downloading anything. It's the same pattern used in real production data pipelines.

**Why one table was the right call for this dataset specifically:**

1. **No reliable identity key.** The source data has no patient MRN, no doctor ID, no hospital ID — nothing that uniquely and permanently identifies an entity across rows. Normalizing into a `patients` table requires deciding what makes two rows "the same patient." The only available option was approximating identity from `name + age`, which is genuinely unsafe: two different people can share a name and age, and the same person's age can shift between visits. Building foreign keys on top of a guessed identity bakes a data quality problem into the schema's structure itself, rather than just leaving it as a flat, honest representation of what the source actually contains.

2. **No analytical question required the join.** Every one of the 8 Gold views could be fully answered from one flat table. Normalization earns its cost when you need to update an entity in one place (e.g. correct a hospital's name once instead of in 10,000 rows) or enforce referential integrity. Neither was needed here — there's no write-heavy application sitting on top of this data, just read-only analytics.

---

## 🥉 4. Bronze Layer — Walkthrough

### Purpose

Get the raw dataset into PostgreSQL **exactly as it is**, with zero transformation. This layer exists purely for ingestion — preserving an unaltered copy of the source data before any cleaning happens.

### Step 1 — Create the schema architecture

```sql
CREATE SCHEMA bronze;
CREATE SCHEMA silver;
CREATE SCHEMA gold;
```

All three schemas are created up front, even though Silver and Gold are empty at this stage — this establishes the full pipeline structure from the start.

### Step 2 — Create the Bronze staging table

```sql
CREATE TABLE bronze.staging_table (
    name                VARCHAR(60),
    age                 INT,
    gender              VARCHAR(50),
    blood_type          VARCHAR(50),
    medical_condition   VARCHAR(50),
    date_of_admission   DATE,
    doctor_name         VARCHAR(50),
    hospital            VARCHAR(50),
    insurance_provider  VARCHAR(50),
    billing_amount      NUMERIC,
    room_number         INT,
    admission_type      VARCHAR(50),
    discharge_date      DATE,
    medication          VARCHAR(50),
    test_results        VARCHAR(50)
);
```

**Key design decision — `billing_amount NUMERIC` with no precision/scale.** The raw CSV contains billing values with long, irregular decimal tails (e.g. `43282.28335770435`). Three options were considered:

- `FLOAT` — Rejected, because floating-point types store decimals as binary approximations. Converting a value like `43282.28335770435` to `FLOAT` can silently shift digits far down the decimal place, which technically alters the source data.
- `DECIMAL(10,2)` — Rejected for Bronze specifically, because rounding to 2 decimal places at ingestion is a transformation, not raw storage. It would violate the principle that Bronze should be untouched.
- `NUMERIC` (no precision/scale) — Chosen, because it stores the exact decimal value with no rounding and no binary approximation. This preserves the source data perfectly, and the rounding decision is made deliberately later, in Silver.

### Step 3 — Load the CSV into the staging table

```sql
COPY bronze.staging_table
FROM '/tmp/healthcare_dataset.csv'
DELIMITER ','
CSV HEADER;
```

### Step 4 — Verify the load

```sql
SELECT * FROM bronze.staging_table LIMIT 4;
SELECT COUNT(*) FROM bronze.staging_table;
```

A quick sanity check to confirm the row count matches the source file and that values loaded into the correct columns.

### Outcome

`bronze.staging_table` now contains a complete, unmodified copy of the Kaggle dataset. It is treated as **read-only** from this point forward — never edited directly. It serves as the permanent source that Silver is rebuilt from.

---

## 🔍 5. Data Quality Testing — Bridging Bronze and Silver

Before writing the Silver transformation logic, every column in `bronze.staging_table` was tested individually to identify what needed fixing. This step matters because it turns "I think the data is messy" into "I know exactly which rows are messy and why."

The general pattern used throughout:

```sql
SELECT DISTINCT <column>
FROM bronze.staging_table
WHERE <condition that should never be true>;
```

If the query returns zero rows, the assumption holds. If it returns rows, that's a confirmed issue to address in Silver.

### Issues identified, column by column

| Column | Issue found | Fix applied in Silver |
|---|---|---|
| `name` | Inconsistent casing; some values had `Mr.`/`Mrs.`/`Dr.` prefixes or `PhD`/`Jr`/`Sr`/`MD`/`II`/`III` suffixes | Standardized casing with `INITCAP`, stripped prefix/suffix with `REGEXP_REPLACE`, split into `patient_first_name` / `patient_last_name` |
| `age` | Checked for nulls and out-of-range values (≤0 or >100) | None found — passed validation as-is |
| `gender` | Checked for values outside Male/Female | None found — passed validation as-is |
| `blood_type` | Checked for leading/trailing whitespace | None found — passed validation as-is |
| `medical_condition` | Checked for nulls/inconsistent casing | None found — passed validation as-is |
| `date_of_admission` | Checked for null dates or dates beyond the current date | None found — passed validation as-is |
| `doctor_name` | Same prefix/suffix issue as `name`, plus credential abbreviations (e.g. `DDS`) at the end | Stripped title prefix, split into `doctor_first_name` / `doctor_last_name` |
| `hospital` | Inconsistent punctuation — stray hyphens, commas, and the word "and" at the start/end of some names | Replaced hyphens with spaces, removed commas, collapsed extra whitespace, stripped leading/trailing "and" |
| `insurance_provider` | Checked for nulls/formatting issues | None found — passed validation as-is |
| `billing_amount` | Contained `NULL` values and negative values | Nulls converted to `0`, negatives converted to their absolute value, all values rounded to 2 decimal places |
| `room_number` | Checked for nulls | None found — passed validation as-is |
| `admission_type` | Checked for whitespace and unexpected category values | None found — passed validation as-is |
| `discharge_date` | Checked for nulls and discharge dates earlier than the admission date | None found — passed validation as-is |
| `medication` | Checked for nulls/formatting | None found — passed validation as-is |
| `test_results` | Checked for nulls/formatting | None found — passed validation as-is |

This table is the bridge between Bronze and Silver — every transformation written into the Silver load procedure exists because a specific test surfaced a specific problem here.

---

## 🥈 6. Silver Layer — Walkthrough

### Purpose

Take the validated issues from Bronze and apply explicit, deliberate transformations to produce one clean, standardized, query-ready table: `silver.healthcare_table`.

### Table structure

```sql
CREATE TABLE silver.healthcare_table (
    patient_first_name   VARCHAR(100),
    patient_last_name    VARCHAR(100),
    age                  INT,
    gender               VARCHAR(50),
    blood_type           VARCHAR(50),
    medical_condition    VARCHAR(50),
    date_of_admission    DATE,
    doctor_first_name    VARCHAR(100),
    doctor_last_name     VARCHAR(100),
    hospital             VARCHAR(100),
    insurance_provider   VARCHAR(50),
    billing_amount       NUMERIC,
    room_number          INT,
    admission_type       VARCHAR(50),
    discharge_date       DATE,
    medication           VARCHAR(50),
    test_results         VARCHAR(50)
);
```

### Key transformation 1 — Name parsing (patients and doctors)

Both `name` and `doctor_name` required the same three steps: standardize casing, strip unwanted prefixes/suffixes, and split into first/last name.

```sql
TRIM(REGEXP_REPLACE(REGEXP_REPLACE(INITCAP(LOWER(name)), '^(Mr|Mrs|Ms|Dr)\.?\s*', '', 'i'),
    '\s*,?\s*(PhD|Jr|Sr|MD|II|III)\.?$', '', 'i'))
```

Breaking this down:
- `INITCAP(LOWER(name))` — lowercases the whole string first, then capitalizes the first letter of every word. This guarantees consistent casing regardless of how the name was originally entered (`JOHN SMITH`, `john smith`, `John SMITH` all become `John Smith`).
- The first `REGEXP_REPLACE` strips a title prefix (`Mr`, `Mrs`, `Ms`, `Dr`) from the **start** of the string, with an optional period and trailing whitespace.
- The second `REGEXP_REPLACE` strips a suffix (`PhD`, `Jr`, `Sr`, `MD`, `II`, `III`) from the **end** of the string, with an optional leading comma.
- `'i'` on both makes the match case-insensitive.

**A bug caught and fixed during development:** the first version of this logic calculated `POSITION(' ' IN name)` using the *original, uncleaned* name, then applied that position to the *cleaned* string. Since stripping the prefix shortens the string, the space position no longer lined up — this would have cut names in the wrong place. The fix was to compute the space position from the same cleaned string being split, not the original.

```sql
-- First name = everything before the first space in the CLEANED name
SUBSTRING(
    TRIM(REGEXP_REPLACE(REGEXP_REPLACE(INITCAP(LOWER(name)), '^(Mr|Mrs|Ms|Dr)\.?\s*', '', 'i'),
        '\s*,?\s*(PhD|Jr|Sr|MD|II|III)\.?$', '', 'i')),
    1,
    POSITION(' ' IN TRIM(REGEXP_REPLACE(REGEXP_REPLACE(INITCAP(LOWER(name)), '^(Mr|Mrs|Ms|Dr)\.?\s*', '', 'i'),
        '\s*,?\s*(PhD|Jr|Sr|MD|II|III)\.?$', '', 'i'))) - 1
) AS patient_first_name
```

### Key transformation 2 — Doctor credential handling

Doctor names had an extra wrinkle: trailing credentials like `DDS` or `MD`. Rather than a fixed list of known suffixes, the doctor-name cleanup uses a more general pattern to catch any short, all-uppercase abbreviation at the end of the string:

```sql
'\s*,?\s*\b([A-Z]{2,5}(?:\.?,?\s*[A-Z]{2,5})*)\.?\s*$'
```

This matches one or more 2–5 letter uppercase abbreviations at the end of the name, separated by optional punctuation — so it generalizes beyond a hardcoded list. **Decision:** in the final Silver table, only `doctor_first_name` and `doctor_last_name` were kept; the credential itself was not persisted as a separate column, since it wasn't required by any of the planned Gold-layer questions.

### Key transformation 3 — Hospital name cleanup

```sql
TRIM(
    REGEXP_REPLACE(
        REGEXP_REPLACE(
            TRIM(REGEXP_REPLACE(REPLACE(REPLACE(hospital, '-', ' '), ',', ''), '\s+', ' ', 'g')),
            '^and\s+', '', 'i'
        ),
        '\s+and$', '', 'i'
    )
) AS hospital
```

Working from the inside out:
1. `REPLACE(hospital, '-', ' ')` — turns hyphens into spaces
2. `REPLACE(..., ',', '')` — removes commas entirely
3. `REGEXP_REPLACE(..., '\s+', ' ', 'g')` — collapses any resulting multiple spaces into one
4. `REGEXP_REPLACE(..., '^and\s+', '', 'i')` — strips "and" if it's the very first word
5. `REGEXP_REPLACE(..., '\s+and$', '', 'i')` — strips "and" if it's the very last word
6. Outer `TRIM()` — removes any leading/trailing whitespace left over

### Key transformation 4 — Billing amount correction

```sql
CASE
    WHEN billing_amount IS NULL THEN 0
    WHEN billing_amount < 0 THEN ROUND(ABS(billing_amount), 2)
    ELSE ROUND(billing_amount, 2)
END AS billing_amount
```

This is the deliberate transformation that the Bronze layer intentionally deferred. Null billing amounts are treated as `0` (rather than left null) so that aggregate functions like `SUM` and `AVG` in Gold behave predictably. Negative values are converted to their positive equivalent using `ABS()`, since a negative hospital bill isn't logically valid and was treated as a data entry error. All values are rounded to 2 decimal places — the real-world precision of currency — at this stage, not in Bronze.

### Making Silver repeatable — the load procedure

Rather than running the `CREATE TABLE` + `INSERT` manually every time the data changes, the Silver load was wrapped in a stored procedure:

```sql
CALL silver.load_healthcare_table();
```

The procedure truncates `silver.healthcare_table` and re-runs the full transformation from `bronze.staging_table` in one call. This means Silver can always be confidently rebuilt from Bronze with a single command, and `RAISE NOTICE` messages at each phase (truncating, inserting, complete) provide visibility into what the procedure is doing when it runs.

### Outcome

`silver.healthcare_table` now holds one row per hospital admission, with clean, standardized names, hospital names, and corrected billing amounts — ready to be queried directly or aggregated into the Gold layer.

---

## 🥇 7. Gold Layer — Walkthrough

### Purpose

Answer specific, business-relevant analytical questions by building views on top of Silver. Gold views don't store their own data — they're saved queries that run against `silver.healthcare_table` every time they're selected from, so they automatically reflect the latest Silver data with no extra rebuild step.

Eight views were built, each demonstrating a different SQL technique:

### 7.1 — `gold.avg_billing_by_condition`

**Question:** What is the average billing amount for each medical condition?

```sql
SELECT medical_condition, ROUND(AVG(billing_amount), 2) AS average_billing
FROM silver.healthcare_table
GROUP BY medical_condition
ORDER BY AVG(billing_amount) DESC;
```

Technique: basic `GROUP BY` aggregation.

### 7.2 — `gold.monthly_admission_trends`

**Question:** How many patients were admitted each month?

```sql
SELECT DATE_TRUNC('month', date_of_admission) AS admission_month, COUNT(*) AS total_admissions
FROM silver.healthcare_table
GROUP BY DATE_TRUNC('month', date_of_admission)
ORDER BY admission_month;
```

Technique: `DATE_TRUNC` for time-series bucketing. `DATE_TRUNC` was chosen over formatting the month as text (e.g. `TO_CHAR(..., 'Month YYYY')`) because it sorts correctly in chronological order automatically, whereas a text month name would sort alphabetically.

### 7.3 — `gold.avg_length_of_stay`

**Question:** What is the average length of stay by admission type?

```sql
SELECT admission_type, ROUND(AVG(discharge_date - date_of_admission), 1) AS average_time_per_admission
FROM silver.healthcare_table
GROUP BY admission_type
ORDER BY average_time_per_admission DESC;
```

Technique: date subtraction. In PostgreSQL, subtracting two `DATE` values returns a plain integer number of days — simpler and more directly usable in `AVG()` than `AGE()`, which returns an interval broken into years/months/days and requires extra steps to convert into a single day count.

### 7.4 — `gold.doctor_workload_ranking`

**Question:** Which doctors handle the most admissions?

```sql
SELECT doctor_first_name, doctor_last_name, COUNT(*) AS patients_seen_by_doctor,
       RANK() OVER (ORDER BY COUNT(*) DESC) AS doctor_rank
FROM silver.healthcare_table
GROUP BY doctor_first_name, doctor_last_name
ORDER BY doctor_rank;
```

Technique: `RANK()` window function — assigns a rank to each doctor based on admission count, while still returning one row per doctor (unlike `COUNT() OVER (PARTITION BY ...)`, which would return one row per admission).

### 7.5 — `gold.billing_by_insurance_provider`

**Question:** What's the billing range (min/max/average) for each insurance provider?

```sql
SELECT insurance_provider,
       ROUND(MAX(billing_amount), 1) AS maximum_billing_amount,
       ROUND(MIN(billing_amount), 1) AS minimum_billing_amount,
       ROUND(AVG(billing_amount), 1) AS avg_billing_amount,
       COUNT(*) AS total_bill_to_insurance
FROM silver.healthcare_table
GROUP BY insurance_provider;
```

Technique: multiple aggregate functions in a single `GROUP BY`.

### 7.6 — `gold.test_result_distribution`

**Question:** For each medical condition, what's the breakdown of test results?

```sql
SELECT medical_condition,
       SUM(CASE WHEN test_results = 'Normal' THEN 1 ELSE 0 END) AS normal,
       SUM(CASE WHEN test_results = 'Abnormal' THEN 1 ELSE 0 END) AS abnormal,
       SUM(CASE WHEN test_results = 'Inconclusive' THEN 1 ELSE 0 END) AS inconclusive,
       COUNT(*) AS results,
       ROUND(100.0 * SUM(CASE WHEN test_results = 'Normal' THEN 1 ELSE 0 END) / COUNT(*), 2) AS normal_percentage_occurance
       -- (same pattern repeated for abnormal and inconclusive)
FROM silver.healthcare_table
GROUP BY medical_condition
ORDER BY medical_condition;
```

Technique: conditional aggregation using `SUM(CASE WHEN...)`, plus deriving percentages from raw counts. Multiplying by `100.0` (not `100`) forces PostgreSQL to use decimal division instead of integer division, which would otherwise truncate the result to a whole number.

### 7.7 — `gold.top_10_highest_billed`

**Question:** Which 10 admissions had the highest billing amount?

```sql
SELECT patient_first_name, patient_last_name, doctor_first_name, doctor_last_name,
       billing_amount, hospital, medical_condition, admission_type,
       date_of_admission, discharge_date
FROM silver.healthcare_table
ORDER BY billing_amount DESC
LIMIT 10;
```

Technique: simple `ORDER BY` + `LIMIT`. An earlier version of this view used `RANK()` with a `WHERE rank <= 10` filter — both approaches are valid, but `RANK()` would return more than 10 rows if multiple admissions tie for 10th place, while `LIMIT` always returns exactly 10. `LIMIT` was kept for simplicity since exact-10 output was preferred for this view.

### 7.8 — `gold.patient_readmission_report`

**Question:** Which patients have been admitted more than once, and what's their billing summary?

```sql
WITH multiple_visit AS (
    SELECT patient_first_name, patient_last_name, COUNT(date_of_admission) AS times_at_hospital
    FROM silver.healthcare_table
    GROUP BY patient_first_name, patient_last_name
    HAVING COUNT(date_of_admission) > 1
),
amounts AS (
    SELECT patient_first_name, patient_last_name,
           ROUND(AVG(billing_amount), 2) AS pt_avg_billing,
           ROUND(SUM(billing_amount), 2) AS pt_total_billing
    FROM silver.healthcare_table
    GROUP BY patient_first_name, patient_last_name
)
SELECT mv.patient_first_name, mv.patient_last_name, mv.times_at_hospital,
       a.pt_avg_billing, a.pt_total_billing
FROM multiple_visit AS mv
LEFT JOIN amounts AS a
    ON mv.patient_first_name = a.patient_first_name
    AND mv.patient_last_name = a.patient_last_name
ORDER BY mv.times_at_hospital DESC;
```

Technique: Combining two CTEs and a `HAVING` clause. `multiple_visit` identifies which patients qualify as "readmitted" (more than one admission), filtering with `HAVING` at the aggregation level rather than filtering afterward — this is more efficient because the dataset is reduced *before* the join happens, not after. `amounts` independently calculates billing totals for every patient. The two are joined together in the final `SELECT`, producing one row per readmitted patient with their full billing picture.

---

## ✅ 8. Summary — End-to-End Pipeline

The full pipeline, run from scratch, looks like this:

```sql
-- 1. Bronze: schema + raw ingestion (run once, or whenever resetting)
CREATE SCHEMA bronze; CREATE SCHEMA silver; CREATE SCHEMA gold;
CREATE TABLE bronze.staging_table (...);
COPY bronze.staging_table FROM '/tmp/healthcare_dataset.csv' DELIMITER ',' CSV HEADER;

-- 2. Silver: clean and standardize (repeatable)
CREATE TABLE silver.healthcare_table (...);
CALL silver.load_healthcare_table();

-- 3. Gold: create the analytical views (run once; views auto-update)
CREATE OR REPLACE VIEW gold.avg_billing_by_condition AS ...
CREATE OR REPLACE VIEW gold.monthly_admission_trends AS ...
-- ... (all 8 views)
```

Once set up, refreshing the entire analytical layer after any Bronze data change requires only:

```sql
CALL silver.load_healthcare_table();
```

Every Gold view immediately reflects the refreshed Silver data — no manual rebuilding required.

---

All scripts referenced in this document can be found in [`/scripts`](./scripts), organized by layer: `/scripts/bronze`, `/scripts/silver`, `/scripts/gold`.

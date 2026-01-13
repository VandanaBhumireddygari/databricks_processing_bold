# 🏥 Healthcare Eligibility Pipeline (S3 + Databricks | Config-Driven)

A configuration-driven data pipeline that ingests eligibility files from multiple healthcare partners (different delimiters + column names), standardizes them into a single schema, and publishes a unified dataset for downstream analytics.

This solution is designed for Databricks + S3 using Spark and Delta Lake.

> Built to match the "Healthcare Eligibility Pipeline" technical assessment requirements (config-driven ingestion, standardized schema, unified output).  


## ✅ What This Pipeline Solves

Healthcare partners send eligibility files in inconsistent formats:
- Different delimiters (pipe vs comma)
- Different column names (MBI vs subscriber_id)
- Mixed formatting (DOB, phone, casing)

Hardcoding partner-specific logic does not scale.

This project separates:
- **Partner-specific rules → configuration**
- **Core processing logic → reusable Spark functions**

So adding a new partner requires **only a config update**, not code changes.

---

## 🧱 Architecture (Bronze → Silver → Gold)

### Bronze (Raw in S3)
Partner files land in S3 as-is.

Example:
```

s3://databricks-age-bold/partner/acme/acme.txt
s3://databricks-age-bold/partner/bettercare/bettercare.csv

```

### Silver (Standardized Delta in S3)
Spark reads raw files, maps to a standard schema, applies transformations, and writes curated Delta output:
```

s3://databricks-age-bold/silver/eligibility_delta

```

### Gold (Unified Consumption Table)
A Delta table is created for downstream consumption:
- `eligibility_gold_unified`

### Rejects (Quarantine)
Rows missing `external_id` are written to:
```

s3://databricks-age-bold/rejects/eligibility_delta

````

---

## 📌 Standardized Output Schema

The final dataset includes:

| Field | Rule |
|------|------|
| external_id | mapped from partner ID field |
| first_name | Title Case |
| last_name | Title Case |
| dob | ISO format `YYYY-MM-DD` |
| email | lowercase |
| phone | formatted `XXX-XXX-XXXX` |
| partner_code | constant per partner (lineage) |

---

## 🔁 Config-Driven Partner Onboarding

Partner rules are stored in a single dictionary `PARTNER_CONFIGS`.

Each partner defines:
- `path` (S3 location)
- `delimiter`
- `dob_format`
- `column_mapping` (partner → standardized)
- `partner_code`

### ✅ Add a new partner
To add a third partner:
1. Add a new config block in `PARTNER_CONFIGS`
2. Provide delimiter, mapping, date format, and partner_code  
3. Re-run the notebook — no code changes required

---

## ▶️ How to Run (Databricks)

### Prerequisites
- Databricks workspace with access to S3 (via External Location/IAM)
- Raw partner files exist in S3

### Run Steps
1. Open the notebook in Databricks
2. Update paths if needed:
   - `SILVER_PATH`
   - `REJECTS_PATH`
   - input file paths in config
3. Run all cells

### Output
- Silver Delta files written to S3
- Gold Delta table created in metastore (`eligibility_gold_unified`)
- Optional rejects dataset written to S3

---

## ✅ Data Quality & Error Handling

This pipeline includes:
- **Validation:** `external_id` is required → missing values go to rejects
- **Graceful parsing:**
  - Invalid DOB formats become `NULL`
  - Invalid phone numbers become `NULL`
- Uses Spark permissive mode to avoid pipeline failure on minor issues

---

## 📊 Quick Verification Queries

```sql
SELECT partner_code, COUNT(*) AS cnt
FROM eligibility_gold_unified
GROUP BY partner_code;
````

```sql
SELECT *
FROM eligibility_gold_unified
ORDER BY partner_code, external_id;
```

---

## 📁 Suggested Repo Structure (GitHub)

```
healthcare-eligibility-pipeline/
├── README.md
├── Data_Processing.ipynb
└── (optional) docs/
    └── architecture.png
```

---

## 💡 Notes

* This project focuses on transformation/standardization once raw files are available in S3 (Bronze).
* Record-level CDC/merge logic is not implemented because the assessment does not provide update semantics (effective dates, change flags, etc.).
* The design supports incremental file-level processing by adding partitioned paths (e.g., ingest_date folders) if needed.

```

---

## If you want, I can also:
- convert your notebook into **`src/` Python scripts** (production style)
- add an **architecture diagram** for GitHub (S3 → Databricks → Delta Gold)
- add **incremental file-level ingestion** using `ingest_date=YYYY-MM-DD` folders (no guessing CDC)

Just tell me what repo name + folder structure you want.
```

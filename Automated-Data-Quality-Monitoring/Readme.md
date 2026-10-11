# Automated Data Quality Monitoring System (n8n)

An n8n workflow that checks any CSV dataset for common data quality problems, gives it a
quality score out of 100, writes a report, **returns a cleaned version of the CSV**, and
saves every result to an n8n Data Table so you can track quality over time.

> Best for: Data Science / Data Engineering portfolios
> Skills demonstrated: data validation, JavaScript, conditional logic, error handling,
> data cleaning, data quality reporting, workflow automation

---

## Features

| Check | What it finds | How it is fixed in the cleaned CSV |
|---|---|---|
| Schema validation | Unreadable file, no rows, blank or duplicate headers, missing expected columns | File is rejected with a clear message |
| Missing values | Blank cells, `NA`, `N/A`, `null`, `none`, `-`, `not available`, `unknown` | Numbers filled with the column median; categories filled with `Unknown`; IDs and dates left blank |
| Duplicate records | Rows that are identical after cleaning | Removed |
| Invalid data types | Text in number columns (`five`, `7 units`, `$49.00`), wrong date formats | Converted (`five`→5, `$49.00`→49, `18/02/2025`→`2025-02-18`); impossible values cleared |
| Outliers | Values outside 1.5×IQR (reported) | Only extreme values (beyond 5×IQR) and negatives in positive-only columns are cleared |
| Inconsistent categories | `Electronics` / `electronics` / `Electonic`, `Office-Supplies`, `Wst` | Mapped to the most common spelling (also fixes small typos) |

Every cleaned row has a `dq_notes` column that explains exactly what was changed.

---

## How it works

Upload CSV (form)
│
▼
Parse CSV ──► Validate Schema ──► Schema Valid?
│ │
true false
│ │
▼ ▼
Check Data Quality Save Rejected Dataset
│ │
▼ ▼
Calculate Quality Score Show Schema Error
│
▼
Generate Report
│
▼
Clean Data
│
▼
Save Results (Data Table)
│
▼
Prepare Cleaned Rows ──► Build Cleaned CSV
│
▼
Show Report (score + download cleaned CSV)


### Node by node

1. **Upload CSV** (Form Trigger) asks for a dataset name, the CSV file, and an optional list of expected columns.
2. **Parse CSV** (Extract from File) turns the file into rows. If parsing fails, the run carries on so the schema step can report the error.
3. **Validate Schema** (Code) checks that the file is readable and has rows, that header names are valid and unique, and that the expected columns are present.
4. **Schema Valid?** (IF) sends valid files to scoring and invalid ones to the rejection branch.
5. **Check Data Quality** (Code) works out the type of each column, then counts missing values, duplicates, type errors, IQR outliers and inconsistent categories, per column.
6. **Calculate Quality Score** (Code) gives a weighted score out of 100 (see below), a grade and a status.
7. **Generate Report** (Code) builds a plain-text and an HTML report.
8. **Clean Data** (Code) fixes the dataset as described in the Features table and counts what it changed.
9. **Save Results** (Data Table) stores one row per run in `data_quality_reports`.
10. **Prepare Cleaned Rows** → **Build Cleaned CSV** turn the cleaned rows into `<original>_cleaned.csv`.
11. **Show Report** (Form ending) shows the score and cleaning summary, and downloads the cleaned CSV.
12. **Save Rejected Dataset** / **Show Schema Error** log rejected files and tell the uploader why.

### Quality score

| Dimension | Weight | Formula |
|---|---|---|
| Completeness | 30 | 1 − missing cells / total cells |
| Uniqueness | 20 | 1 − duplicate rows / rows |
| Validity | 20 | 1 − type errors / non-missing cells |
| Consistency | 15 | 1 − inconsistent category values / non-missing cells |
| Outlier-free | 15 | 1 − outliers / numeric cells |

**Grade:** A ≥ 90, B ≥ 80, C ≥ 70, D ≥ 60, F < 60
**Status:** PASS ≥ 80, WARNING 60–79, FAIL < 60, REJECTED = failed schema validation

---

## Setup

### Requirements
- n8n (Cloud, or self-hosted version 1.x or later with Data Tables enabled)
- No external credentials or API keys

### 1. Create the Data Table
In n8n, go to **Data Tables → Create** and name it `data_quality_reports`, with these columns:

| Column | Type |
|---|---|
| dataset_name | string |
| file_name | string |
| row_count | number |
| column_count | number |
| quality_score | number |
| grade | string |
| status | string |
| schema_valid | boolean |
| schema_issues | string |
| missing_values | number |
| duplicate_rows | number |
| type_issues | number |
| outliers | number |
| inconsistent_categories | number |
| report_text | string |
| checked_at | date |
| cleaned_row_count | number |
| rows_removed | number |
| values_fixed | number |
| cleaning_summary | string |

### 2. Import the workflow
1. In n8n, go to **Workflows → Import from File** and choose `workflow.json` from this repo.
2. Open **Save Results** and select your `data_quality_reports` table.
3. Do the same in **Save Rejected Dataset**.
4. Save the workflow.

### 3. Test it
1. Click **Execute workflow**. A test form opens.
2. Upload `sample-data/messy_sales_dataset.csv` (or any CSV).
3. You should see the score page, and the cleaned CSV should download.
4. Check the `data_quality_reports` table for the new row.

### 4. Go live (optional)
**Publish** the workflow. The form is then available at:
`https://<your-n8n-domain>/form/data-quality-check`

---

## Usage

| Form field | Required | Example |
|---|---|---|
| Dataset name | yes | `Sales Data Q3` |
| CSV file | yes | `sales.csv` (first row must be the headers) |
| Expected columns | no | `Customer_ID, Order_Date, Quantity` (makes schema validation stricter) |

**Output:**
- Browser page: score, grade, status and a cleaning summary
- Download: `<file>_cleaned.csv` with a `dq_notes` column
- Data Table: one row per run, for history and dashboards

### Example result (100-row messy sales file)
Score: 98.3/100 | Grade: A | Status: PASS
Missing values: 11 | Duplicates: 1 | Invalid types: 7 | Outliers: 13 | Inconsistent categories: 6
Cleaned 100 -> 99 rows. numbers fixed: 152, categories standardized: 7,
extreme outliers cleared: 3, values imputed: 14, duplicates removed: 1


---

## Customization

| What | Where |
|---|---|
| Score weights | `weights` in **Calculate Quality Score** |
| PASS/WARNING/FAIL thresholds | `status` line in **Calculate Quality Score** |
| Values treated as missing | `MISSING` list in **Clean Data**, `isEmpty` in **Check Data Quality** |
| Outlier sensitivity | `1.5 * iqr` (reporting) in **Check Data Quality**; `5 * iqr` (removal) in **Clean Data** |
| Fill strategy | Step 5 in **Clean Data** (median / `Unknown`) |
| Form URL path | **Upload CSV** → Options → Path |

---

## Limitations
- Processes the whole file in memory. Very large CSVs (hundreds of MB) may hit n8n memory limits.
- Column types are detected automatically: a column counts as typed when at least 70% of its values agree.
- `values_fixed` also counts format-only changes (e.g. `1.0` → `1`), so it can look high.
- Filling gaps with the median or `Unknown` is a simple default. Review `dq_notes` before using the cleaned data for modeling.

---

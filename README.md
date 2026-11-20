# Messy CRM & E-Commerce Data Cleanup (SQL + Google Sheets)

> An end-to-end data quality pipeline that turns a messy, multi-format CRM transactions export into a clean, validated, analysis-ready table. The same logic is implemented twice, in **BigQuery SQL** and in **Google Sheets formulas**, and fully documented.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [The Problem](#the-problem)
3. [Tech Stack](#tech-stack)
4. [Repository Structure](#repository-structure)
5. [Data Quality Issues Found](#data-quality-issues-found)
6. [Cleaning Pipeline (SQL)](#cleaning-pipeline-sql)
7. [Spreadsheet Workflow (Google Sheets)](#spreadsheet-workflow-google-sheets)
8. [Before / After Examples](#before--after-examples)
9. [Output Schema](#output-schema)
10. [Key Design Decisions](#key-design-decisions)
11. [Known Limitations & Next Steps](#known-limitations--next-steps)
12. [How to Reproduce](#how-to-reproduce)
13. [Skills Demonstrated](#skills-demonstrated)
14. [About the Author](#about-the-author)

---

## Project Overview

Real-world CRM exports are rarely clean. This project takes a raw transactional dataset (**20 rows, 7 columns**) full of inconsistent formats and builds a **documented, repeatable cleaning process** that:

- Standardizes names, emails, phone numbers, dates, and regions
- Converts mixed-currency revenue (USD, EUR, GBP, MAD) into a single currency (**USD**)
- Detects and removes duplicate customer records
- Quarantines test, dummy, and corrupted records
- Adds **audit and validation flag columns**, so nothing is silently changed or hidden

The result is a trustworthy dataset that a BI tool, finance team, or marketing team can use without second-guessing it.

---

## The Problem

A business exports its CRM and e-commerce transactions, but the data cannot be used for reporting because:

| Area | What's wrong |
|---|---|
| **Customers** | The same customer appears 2-3 times with slightly different formatting |
| **Phone numbers** | 8+ formats (`06-12345678`, `(555) 321-9876`, `+212 612-987654`, `06.12.34.56.78`) |
| **Revenue** | Mixed symbols and codes (`$150.00`, `200.50 USD`, `450.00$`, `€125.50`, `£210.00`, `1450.00 MAD`) |
| **Dates** | 5 different layouts plus invalid text (`Invalid_Date`) |
| **Regions** | Typos and aliases (`calfornia`, `ca`, `ny`, `marrakesh`, `Marrakech`) |
| **Emails / Names** | Dummy emails, `N/A` names, stray whitespace, random casing |

---

## Tech Stack

| Tool | Used for |
|---|---|
| **Google BigQuery (Standard SQL)** | Main cleaning pipeline (CTEs, regex, window functions) |
| **Google Sheets** | Formula-based version of the same pipeline, plus a documentation sheet |
| **CSV** | Raw input and cleaned output |

**SQL features used:** `CTE` chaining, `REGEXP_CONTAINS`, `REGEXP_REPLACE`, `SAFE.PARSE_DATE`, `SAFE_CAST`, `COALESCE`, `NULLIF`, `INITCAP`, `CASE WHEN`, `ROW_NUMBER() OVER (PARTITION BY ...)`, `INFORMATION_SCHEMA`.

---

## Repository Structure

```
messy-crm-data-cleanup/
│
├── data/
│   ├── raw_crm_transactions.csv        # Original messy dataset (20 rows)
│   └── cleaned_crm_transactions.csv    # Cleaned output
│
├── sql/
│   └── clean_crm_transactions.sql      # Documented BigQuery pipeline
│
├── assets/                             # README graphics (SVG + PNG)
│
├── spreadsheet/
│   └── crm_cleanup_workbook.xlsx       # Raw sheet, formula-driven cleaned sheet, Pipeline_Documentation
│
└── README.md
```

---

## Data Quality Issues Found

| # | Issue | Example from raw data | How it was handled |
|---|---|---|---|
| 1 | Duplicate customers | `CUST-001` appears 3 times, `CUST-002` and `CUST-010` twice | `ROW_NUMBER()` keeps one row per customer (latest date, then highest amount) |
| 2 | Inconsistent name casing / spaces | `john doe `, `JANE SMITH`, `  spacey mcbias  ` | `TRIM` + `INITCAP` produce `John Doe` |
| 3 | Placeholder names | `N/A`, empty | `NULLIF` converts to `Unknown Customer` |
| 4 | Invalid / dummy emails | `invalid_email@`, `unknown@test.com`, `mary.b@domain.com` | Classified as `Valid`, `Missing Format`, or `Test/Dummy`; invalid addresses set to `NULL` |
| 5 | Mixed phone formats | `06-12345678`, `+1-555-456-7890`, `0760112233` | Strip non-digits, detect country, rebuild in international format (`+212 ...`, `+1 ...`, `+44 ...`) |
| 6 | Test phone numbers | `555-...` numbers | Flagged `Test_Or_Dummy_Number` |
| 7 | Mixed date formats | `2026-10-01`, `10/02/2026`, `2026/10/08`, `14/10/2026`, `10-09-2026` | `COALESCE` over several `SAFE.PARSE_DATE` formats, producing a real `DATE` type |
| 8 | Invalid dates | `Invalid_Date` | Parsed as `NULL` (no crash, no guess) |
| 9 | Region typos / aliases | `calfornia`, `ca`, `ny`, `marrakesh`, `London`, `Paris` | Mapped to one canonical name per region |
| 10 | Mixed-currency revenue | `€125.50`, `£210.00`, `1450.00 MAD`, `450.00$` | Extract number, detect currency, convert to USD |
| 11 | Negative amounts | `$-50.00` | Kept and labeled `Refund / Negative` |
| 12 | Missing revenue | `N/A`, `NULL`, empty | Labeled `Missing Revenue` |
| 13 | Test / corrupted records | `CUST-004`, `CUST-007`, `CUST-011`, `CUST-016` | Quarantined from the final table |

**Row accounting:** 20 raw rows, minus 4 quarantined rows, minus 4 duplicate rows, leaves 12 clean rows.

<p align="center">
  <img src="assets/row_accounting.svg" alt="Row accounting from 20 raw rows to 12 clean rows" width="100%">
</p>

---

## Cleaning Pipeline (SQL)

The script `clean_crm_transactions.sql` is written as a chain of CTEs, so each stage does **one job** and can be debugged on its own.

<p align="center">
  <img src="assets/pipeline.svg" alt="7-stage cleaning pipeline diagram" width="100%">
</p>

<details>
<summary>CTE-by-CTE breakdown</summary>

```
raw_crm_transactions
        │
        ▼
 STEP 1  Audit               → row count, column names and data types
        │
        ▼
 STEP 2  Cleaned_base                    → trim IDs, standardize names
         Transformed_Attributes          → email validation and sanitization
         extracted_phone_digits_...      → date parsing, digits-only phone, country detection
         Standardized_Phones_And_Regions → phone formatting, phone validation, region mapping
         revenue_standardized            → currency parsing, USD conversion, transaction status
         Target_Vector_Quarantine_...    → remove test / corrupted customers
         deduplicated_transactions       → ROW_NUMBER() de-duplication
        │
        ▼
 STEP 3  Final SELECT        → clean columns and market/region cross-validation flag
        │
        ▼
 cleaned_crm_transactions (permanent BigQuery table)
```

</details>

### Key Technique: Multi-Format Date Parsing

```sql
COALESCE(
  SAFE.PARSE_DATE('%Y-%m-%d', TRIM(order_date)),
  SAFE.PARSE_DATE('%m/%d/%Y', TRIM(order_date)),
  SAFE.PARSE_DATE('%Y/%m/%d', TRIM(order_date)),
  SAFE.PARSE_DATE('%d-%m-%Y', TRIM(order_date)),
  SAFE.PARSE_DATE('%d/%m/%Y', TRIM(order_date))
) AS parsed_order_date
```

`SAFE.` returns `NULL` instead of throwing an error, so bad values fall through to the next format.

### Key Technique: Currency Normalization

```sql
ROUND(
  COALESCE(
    SAFE_CAST(REGEXP_REPLACE(raw_revenue, r'[^0-9.-]', '') AS FLOAT64) *
    CASE
      WHEN UPPER(raw_revenue) LIKE '%MAD%'                          THEN 0.1018
      WHEN UPPER(raw_revenue) LIKE '%EUR%' OR raw_revenue LIKE '%€%' THEN 1.1177
      WHEN UPPER(raw_revenue) LIKE '%GBP%' OR raw_revenue LIKE '%£%' THEN 1.2800
      ELSE 1.0
    END, 0.0), 2) AS revenue_amount_usd
```

| Currency | Rate to USD (static) |
|---|---|
| USD | 1.0000 |
| EUR | 1.1177 |
| GBP | 1.2800 |
| MAD | 0.1018 |

### Key Technique: De-duplication

```sql
ROW_NUMBER() OVER (
  PARTITION BY customer_id
  ORDER BY parsed_order_date DESC, revenue_amount_usd DESC
) AS row_num
...
WHERE row_num = 1
```

### Key Technique: Cross-Validation Flag

A final check compares the **country detected from the phone number** with the **standardized region**, to catch contradictory records (for example, a Moroccan phone number with a California region).

---

## Spreadsheet Workflow (Google Sheets)

`crm_cleanup_workbook.xlsx` reproduces the pipeline with formulas, for teams that work in spreadsheets instead of a data warehouse.

| Sheet | Purpose |
|---|---|
| `raw_crm_transactions` | Untouched source data |
| `cleaned_crm_transactions` | Raw columns next to formula-driven cleaned columns, so every change is traceable |
| `Pipeline_Documentation` | A data dictionary: for each output column, its data type, source field, formula, and business rationale |

**Formula techniques:** nested `IF` / `IFS`, `SEARCH`, chained `SUBSTITUTE` (to strip phone symbols), `PROPER`, `TRIM`, `MID` / `LEFT`, `REGEXMATCH`, and `LET` for readable revenue parsing.

---

## Before / After Examples

<p align="center">
  <img src="assets/before_after.svg" alt="Before and after cleaning examples" width="100%">
</p>

<details>
<summary>More examples (table)</summary>

| Field | Raw value | Cleaned value |
|---|---|---|
| Name | `JANE SMITH` | `Jane Smith` |
| Name | `N/A` | `Unknown Customer` |
| Email | `jane.smith@yahoo.com ` | `jane.smith@yahoo.com` |
| Phone | `+1 (555) 987-6543` | `+1 5559876543` |
| Region | `calfornia` | `California` |
| Region | `London` | `Greater London` |
| Date | `14/10/2026` | `2026-10-14` |
| Revenue | `1450.00 MAD` | `147.66` (USD) |
| Revenue | `£210.00` | `268.80` (USD) |
| Revenue | `3200 MAD` | `325.87` (USD) |
| Revenue | `$-50.00` | `-50.00`, labeled `Refund / Negative` |

</details>

---

## Output Schema

| Column | Type | Description |
|---|---|---|
| `customer_id` | STRING | Trimmed unique customer key |
| `full_name` | STRING | Title Case name, `Unknown Customer` if missing |
| `Cleaned_email` | STRING | Lowercase valid email, `NULL` if invalid |
| `email_validation_status` | STRING | `Valid Email`, `Invalid: Missing`, `Invalid: Missing Format`, `Invalid: Test/Dummy` |
| `standardized_phone` | STRING | International-format phone number |
| `country_market` | STRING | `Morocco`, `North America`, `United Kingdom`, `Other / International` |
| `phone_validation_status` | STRING | `VALID`, `Test_Or_Dummy_Number`, `Invalid_Length` |
| `parsed_order_date` | DATE | Parsed date, `NULL` if unparseable |
| `standardized_region` | STRING | Canonical region name |
| `revenue_amount_usd` | FLOAT | Revenue converted to USD |
| `transaction_status` | STRING | `Standard Sale`, `Refund / Negative`, `Missing Revenue` |
| `market_region_cross_check` | STRING | `Aligned` or a `Warning: ...` message |

---

## Key Design Decisions

- **Flag, don't hide.** Validation columns (`email_validation_status`, `phone_validation_status`, `transaction_status`, `market_region_cross_check`) record *why* a value is suspicious instead of silently deleting data.
- **`NULL` over guessing.** Unparseable dates and invalid emails become `NULL`, so no wrong value reaches a report.
- **Keep raw values for audit.** Original phone, email, region, and revenue are carried through the intermediate CTEs.
- **Safe functions.** `SAFE.PARSE_DATE` and `SAFE_CAST` keep the pipeline from failing on bad rows.
- **One CTE, one job.** The structure is easier to read, test, and maintain.
- **Two implementations.** SQL provides scale and automation; Google Sheets serves non-technical users and quick review.

---

## Known Limitations & Next Steps

The following trade-offs are documented for transparency:

- **Ambiguous dates.** Formats like `10-09-2026` or `10/02/2026` can be read as day-first or month-first. The parser tries month-first for `/` dates and day-first for `-` dates; the correct choice depends on the source system's convention. A production version would confirm this with the data owner.
- **Static exchange rates.** Rates are hard-coded. A production pipeline would join a dated FX-rate table and use the rate for each order date.
- **Hard-coded quarantine list.** Test and corrupted customers (`CUST-004`, `CUST-007`, `CUST-011`, `CUST-016`) are excluded by ID. A rule-based quarantine (for example, when email is invalid **and** revenue is missing) would scale better.
- **`555` test-number rule.** All US-style numbers in this sample use `555`, so they are all flagged as dummy. This suits a synthetic dataset but is too broad for real data.
- **Country detection by prefix.** This works for the sample's markets (Morocco, US/Canada, UK). A library such as `libphonenumber` is better for full international coverage.
- **De-duplication key.** Rows are collapsed by `customer_id` only, which assumes each customer has one valid transaction in the dataset. With real data a customer can have several legitimate orders, so the key should include an order ID.

**Possible extensions:** a data quality scorecard (percentage of valid emails and phones), a Looker Studio dashboard, scheduled queries, and unit tests with `ASSERT` statements.

---

## How to Reproduce

### BigQuery

1. Create a dataset (for example `customer_clean_staging`) in your BigQuery project.
2. Upload `data/raw_crm_transactions.csv` as a table named `raw_crm_transactions`. Set `order_date` and `revenue_amount` as **STRING** columns, because the raw values are in mixed formats.
3. Open `sql/clean_crm_transactions.sql` and replace the project and dataset path with your own: `your-project.customer_clean_staging`.
4. Run the script. It creates the table `cleaned_crm_transactions`.

### Google Sheets

1. Import `spreadsheet/crm_cleanup_workbook.xlsx` into Google Sheets. Some formulas, such as `REGEXMATCH` and `LET`, are designed for Google Sheets.
2. Review the `Pipeline_Documentation` sheet for the column-by-column logic.

---

## Skills Demonstrated

`Data Cleaning` · `Data Quality Auditing` · `SQL (BigQuery)` · `Regex` · `Window Functions` · `Data Standardization` · `Currency Conversion` · `Google Sheets Formulas` · `Technical Documentation` · `Data Validation`

---

## About the Author

**Ismail Hachouni** is a Data Analyst based in Morocco, focused on turning messy data into reliable insight. Open to freelance data cleaning, SQL, and reporting projects.

- LinkedIn: `[add your link]`
- GitHub: `[add your link]`
- Email: `[add your email]`

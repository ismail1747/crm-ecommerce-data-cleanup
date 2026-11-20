# Messy CRM & E-Commerce Data Sanitation (SQL + Spreadsheets)

**Author:** Ismail Hachouni · **Environment:** Google BigQuery, Google Sheets

---

## 1. Executive Summary

CRM and e-commerce exports rarely arrive analysis-ready. Customer records are typed by hand, merged from multiple systems, and re-submitted by retrying integrations. The result is inconsistent phone formats, mixed text casing, misspelled regions, multi-currency revenue strings, test accounts, and duplicate orders.

Left uncleaned, these issues quietly distort the numbers leadership relies on:

| Business Metric | How Dirty Data Distorts It |
|---|---|
| Revenue and order volume | Duplicate submissions and test transactions inflate totals |
| Customer counts | The same person appears under multiple spellings or phone formats |
| Regional performance | `calfornia`, `CA` and `California` split into three segments |
| Marketing reach | Invalid or dummy emails and phone numbers waste campaign spend |
| Financial reporting | Mixed currencies (`$`, `USD`, `MAD`, `EUR`, `GBP`) cannot be summed |

This project builds the same data quality pipeline **twice**: as a scalable, repeatable **BigQuery SQL** script and as a transparent, auditable **Google Sheets** formula workflow. The output is a single standardized table, with validation flags, that is safe for dashboards, CRM imports, and finance reporting.

---

## 2. Dataset Overview

The raw dataset is a CRM transaction export with seven columns:

`customer_id`, `full_name`, `email`, `phone_number`, `order_date`, `region_state`, `revenue_amount`

### Raw data sample

| customer_id | full_name | email | phone_number | order_date | region_state | revenue_amount |
|---|---|---|---|---|---|---|
| CUST-001 | john doe | john.doe@gmail.com | 06-12345678 | 2026-10-01 | california | $150.00 |
| CUST-002 | JANE SMITH | jane.smith@yahoo.com | +212 612-987654 | 10/02/2026 | Marrakech-Safi | 200.50 USD |
| CUST-003 | bob smith | bob.smith@outlook.com | (555) 321-9876 | 2026-10-03 | calfornia | $75.25 |
| CUST-001 | john doe | john.doe@gmail.com | 06-12345678 | 2026-10-01 | california | $150.00 |

### Issues identified

| Category | Examples in the Data |
|---|---|
| **Dirty phone numbers** | `06-12345678`, `+212 612-987654`, `(555) 321-9876`: three different formats, with dummy `555` numbers |
| **Inconsistent text casing** | `john doe`, `JANE SMITH`, `bob smith` |
| **Mixed date formats** | `2026-10-01` vs `10/02/2026` |
| **Misspelled / inconsistent regions** | `calfornia`, `california`, `Marrakech-Safi` |
| **Mixed currency formats** | `$150.00`, `200.50 USD`, plus `MAD`, `EUR`, `GBP` handled by the pipeline |
| **Exact duplicates** | `CUST-001` appears twice with identical values |
| **Missing / placeholder values** | `N/A` names, blank emails, null revenue |
| **Test and corrupted records** | Sandbox accounts (`CUST-004`, `CUST-007`, `CUST-016`) and a corrupted payload (`CUST-011`) |

### Cleaned output sample

| customer_id | full_name | Cleaned_email | standardized_phone | country_market | phone_validation_status | parsed_order_date | standardized_region | revenue_amount_usd | transaction_status | market_region_cross_check |
|---|---|---|---|---|---|---|---|---|---|---|
| CUST-001 | John Doe | john.doe@gmail.com | +212 612345678 | Morocco | VALID | 2026-10-01 | California | 150.00 | Standard Sale | Warning: Moroccan Market with Non-Local Region |
| CUST-002 | Jane Smith | jane.smith@yahoo.com | +212612987654 | Morocco | VALID | 2026-10-02 | Marrakech-Safi | 200.50 | Standard Sale | Aligned |
| CUST-003 | Bob Smith | bob.smith@outlook.com | +5553219876 | Other / International | Test_Or_Dummy_Number | 2026-10-03 | California | 75.25 | Standard Sale | Aligned |

> The duplicate `CUST-001` row is collapsed to one record. Note how the cross-check flag surfaces a genuine business question on `CUST-001` (Moroccan phone, California region) instead of silently passing it through.

---

## 3. Side-by-Side Solution Comparison

| Data Quality Issue | BigQuery SQL Method | Google Sheets Method |
|---|---|---|
| **Whitespace and missing IDs** | `TRIM(customer_id)` | `=IF(ISBLANK(A2), "MISSING_ID", TRIM(A2))` |
| **Inconsistent name casing / `N/A` names** | `COALESCE(INITCAP(NULLIF(UPPER(TRIM(full_name)),'N/A')),'Unknown Customer')` | `PROPER(TRIM(C2))` wrapped in an `IF(OR(ISBLANK…, "n/a", "unknown"))` guard |
| **Invalid, malformed, test, or placeholder-domain emails** | `CASE` with `REGEXP_CONTAINS` matching test, dummy, and placeholder-domain patterns (`@domain.com`, `@example.com`); `NULL` for the cleaned value | Nested `IF` + `SEARCH` for the status label; `IF(OR(…), "", LOWER(TRIM(E2)))` for the cleaned value |
| **Unstructured phone formats** | `REGEXP_REPLACE(phone, r'[^0-9]', '')` to extract digits | Nested `SUBSTITUTE` removing `(`, `)`, `-`, space, `+` |
| **Local vs international numbers** | `CASE` on prefix, producing `+212`, `+1`, `+44` formats via `CONCAT` and `SUBSTR` | Nested `IF` on `LEFT()` and `LEN()` with `CONCAT("+212 ", MID(…))` |
| **Market detection** | `CASE` on `+212` / `06` / `07` / `+1` / `+44` | Nested `IF` on `LEFT()` prefixes |
| **Dummy phone numbers** | Digits starting with `555` flagged as `Test_Or_Dummy_Number` | `SEARCH("555", H2)` flagged as `Test_Or_Dummy_Number` |
| **Mixed date formats** | `COALESCE` of five `SAFE.PARSE_DATE` patterns | `REGEXMATCH` branching, then reassembly into ISO `YYYY-MM-DD` |
| **Misspelled regions** | `CASE` with `IN (...)` and `LIKE '%marrakech%'`, with `INITCAP` fallback | Nested `IF` with `SEARCH` on keywords and common typos |
| **Multi-currency revenue** | `REGEXP_REPLACE` to strip symbols, `SAFE_CAST`, then `CASE` multiplier per currency | `LET` block: `REGEXREPLACE`, `VALUE`, then conversion by currency marker |
| **Refunds and missing revenue** | `CASE` on negative cast value or null / `N/A` | `IF` on sign and zero value |
| **Test and corrupted records** | Quarantine filter with `customer_id NOT IN (...)` | Flagged as `Test_Data` and exempted from cross-checks |
| **Duplicates** | `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY parsed_order_date DESC, revenue_amount_usd DESC)`, keep `row_num = 1` | Manual audit rules, then `Data > Data cleanup > Remove duplicates` |
| **Market vs region contradictions** | `CASE` flag comparing `country_market` to `standardized_region` | Currency-to-region `Match` / `Mismatch` check |

---

## 4. SQL Implementation (BigQuery)

The full pipeline is a single `CREATE OR REPLACE TABLE ... AS WITH ...` statement built from chained CTEs. Each CTE primarily handles one specific transformation stage, keeping the logic modular, readable, and easy to audit, with the final output saved to `durable-boulder-507309-q2.customer_clean_staging.cleaned_crm_transactions`.

### Step 1: Initial audit

Before changing anything, check the row count and schema of the raw table.

```sql
SELECT COUNT(*) AS total_rows
FROM `durable-boulder-507309-q2.customer_clean_staging.raw_crm_transactions`;

SELECT column_name, data_type
FROM `durable-boulder-507309-q2.customer_clean_staging.INFORMATION_SCHEMA.COLUMNS`
WHERE table_name = 'raw_crm_transactions';
```

### Step 2: Materialize the output table and clean the base (IDs and names)

The result of the whole pipeline is saved as a permanent table for downstream reporting. The first CTE trims IDs and standardizes names to Title Case, converting `N/A` placeholders to `Unknown Customer`.

```sql
-- Materializes the final, deduplicated dataset for downstream reporting.
CREATE OR REPLACE TABLE `durable-boulder-507309-q2.customer_clean_staging.cleaned_crm_transactions` AS

WITH Cleaned_base AS (
  SELECT
    TRIM(customer_id) AS customer_id,
    COALESCE(INITCAP(NULLIF(UPPER(TRIM(full_name)), 'N/A')), 'Unknown Customer') AS full_name,
    email, phone_number, order_date, region_state, revenue_amount
  FROM `durable-boulder-507309-q2.customer_clean_staging.raw_crm_transactions`
),
```

### Step 3: Email validation and sanitization

Two outputs are produced: a **status label** for auditing and a **cleaned value** (malformed and test emails become `NULL` so they never reach downstream tools). The status logic checks, in order, for missing values, malformed addresses (no `@` followed by a dot, or containing spaces), and test, dummy, or placeholder-domain addresses. The placeholder-domain patterns (`@domain.com`, `@example.com`, `@sample.com`) replace what used to be a manual, ID-based audit rule for `CUST-006`.

```sql
Transformed_Attributes AS (
  SELECT
    customer_id, full_name, email AS raw_email,
    CASE
      WHEN email IS NULL OR TRIM(email) = '' THEN 'Invalid: Missing'
      WHEN LOWER(TRIM(email)) NOT LIKE '%@%.%'
        OR LOWER(TRIM(email)) LIKE '% %' THEN 'Invalid: Missing Format'
      WHEN REGEXP_CONTAINS(
        LOWER(TRIM(email)),
        r'(test|unknown|invalid|dummy|fake|noreply|@domain\.com|@example\.com|@sample\.com)'
      ) THEN 'Invalid: Test/Dummy'
      ELSE 'Valid Email'
    END AS email_validation_status,
    CASE
      WHEN email NOT LIKE '%@%' OR email NOT LIKE '%.%'
        OR LOWER(TRIM(email)) LIKE '%test%'
        OR LOWER(TRIM(email)) LIKE '%unknown%'
        OR LOWER(TRIM(email)) LIKE '%invalid%' THEN NULL
      ELSE LOWER(TRIM(email))
    END AS Cleaned_email,
    phone_number, order_date, region_state, revenue_amount
  FROM Cleaned_base
),
```

### Step 4: Date parsing and phone market detection

`SAFE.PARSE_DATE` returns `NULL` instead of failing on a bad format, so `COALESCE` can try several layouts in order. Phone digits are extracted with a regex, and the market is inferred from the prefix.

```sql
extracted_phone_digits_and_country_market AS (
  SELECT
    customer_id, full_name, raw_email, email_validation_status, Cleaned_email,
    COALESCE(
      SAFE.PARSE_DATE('%Y-%m-%d', TRIM(order_date)),
      SAFE.PARSE_DATE('%m/%d/%Y', TRIM(order_date)),
      SAFE.PARSE_DATE('%Y/%m/%d', TRIM(order_date)),
      SAFE.PARSE_DATE('%d-%m-%Y', TRIM(order_date)),
      SAFE.PARSE_DATE('%d/%m/%Y', TRIM(order_date))
    ) AS parsed_order_date,
    phone_number AS original_phone_number,
    REGEXP_REPLACE(TRIM(phone_number), r'[^0-9]', '') AS digits_only,
    CASE
      WHEN phone_number LIKE '+212%' OR phone_number LIKE '06%' OR phone_number LIKE '07%' THEN 'Morocco'
      WHEN phone_number LIKE '+1%'   THEN 'North America'
      WHEN phone_number LIKE '+44%'  THEN 'United Kingdom'
      ELSE 'Other / International'
    END AS country_market,
    order_date,
    region_state AS raw_region,
    revenue_amount
  FROM Transformed_Attributes
),
```

### Step 5: Phone standardization, phone validation, and region mapping

```sql
Standardized_Phones_And_Regions AS (
  SELECT
    customer_id, full_name, raw_email, email_validation_status, Cleaned_email,
    original_phone_number, parsed_order_date, digits_only, country_market,
    CASE
      WHEN country_market = 'Morocco' AND digits_only LIKE '06%' THEN CONCAT('+212 ', SUBSTR(digits_only, 2, 9))
      WHEN country_market = 'Morocco' AND digits_only LIKE '07%' THEN CONCAT('+212 ', SUBSTR(digits_only, 2, 9))
      WHEN country_market = 'North America'  THEN CONCAT('+1 ', REGEXP_REPLACE(digits_only, r'^1', ''))
      WHEN country_market = 'United Kingdom' THEN CONCAT('+44 ', SUBSTR(digits_only, 3))
      ELSE CONCAT('+', digits_only)
    END AS standardized_phone,
    CASE
      WHEN REGEXP_REPLACE(original_phone_number, r'[^0-9]', '') LIKE '555%' THEN 'Test_Or_Dummy_Number'
      WHEN country_market = 'Morocco'
        AND LENGTH(REGEXP_REPLACE(original_phone_number, r'[^0-9]', '')) NOT IN (9, 10, 12) THEN 'Invalid_Length'
      ELSE 'VALID'
    END AS phone_validation_status,
    CASE
      WHEN LOWER(raw_region) IN ('ny', 'new york')                 THEN 'New York'
      WHEN LOWER(raw_region) IN ('ca', 'calfornia', 'california')  THEN 'California'
      WHEN LOWER(raw_region) = 'texas'                             THEN 'Texas'
      WHEN LOWER(raw_region) = 'london'                            THEN 'Greater London'
      WHEN LOWER(raw_region) = 'paris'                             THEN 'Île-de-France'
      WHEN LOWER(raw_region) LIKE '%marrakech%'                    THEN 'Marrakech-Safi'
      WHEN LOWER(raw_region) LIKE '%casablanca%'                   THEN 'Casablanca-Settat'
      WHEN LOWER(raw_region) LIKE '%rabat%'                        THEN 'Rabat-Sale-Kenitra'
      WHEN LOWER(raw_region) LIKE '%tangier%'                      THEN 'Tangier-Tetouan-Al Hoceima'
      ELSE INITCAP(raw_region)
    END AS standardized_region,
    revenue_amount AS raw_revenue
  FROM extracted_phone_digits_and_country_market
),
```

### Step 6: Revenue normalization to USD and transaction classification

Non-numeric characters are stripped, the value is cast to `FLOAT64`, and a currency multiplier is applied based on the symbol or code found in the original string.

```sql
revenue_standardized AS (
  SELECT
    customer_id, full_name, raw_revenue, raw_email, email_validation_status, Cleaned_email,
    original_phone_number, digits_only, country_market, standardized_phone,
    phone_validation_status, parsed_order_date, standardized_region,
    ROUND(COALESCE(
      SAFE_CAST(REGEXP_REPLACE(raw_revenue, r'[^0-9.-]', '') AS FLOAT64) *
      CASE
        WHEN UPPER(raw_revenue) LIKE '%MAD%' THEN 0.1018329938900204
        WHEN UPPER(raw_revenue) LIKE '%EUR%' OR raw_revenue LIKE '%€%' THEN 1.1177
        WHEN UPPER(raw_revenue) LIKE '%GBP%' OR raw_revenue LIKE '%£%' THEN 1.2800
        WHEN UPPER(raw_revenue) LIKE '%USD%' OR raw_revenue LIKE '%$%' THEN 1.0
        ELSE 1
      END, 0.0), 2) AS revenue_amount_usd,
    CASE
      WHEN SAFE_CAST(REGEXP_REPLACE(raw_revenue, r'[^0-9.-]', '') AS FLOAT64) < 0 THEN 'Refund / Negative'
      WHEN raw_revenue IS NULL OR UPPER(raw_revenue) IN ('N/A', 'NULL', '') THEN 'Missing Revenue'
      ELSE 'Standard Sale'
    END AS transaction_status
  FROM Standardized_Phones_And_Regions
),
```

### Step 7: Quarantine and deduplication

Known test accounts and the corrupted payload are removed, then `ROW_NUMBER()` keeps one record per customer, preferring the most recent date.

```sql
Target_Vector_Quarantine_And_Conflict AS (
  SELECT *
  FROM revenue_standardized
  WHERE customer_id NOT IN ('CUST-004', 'CUST-007', 'CUST-016')  -- sandbox / test vectors
    AND customer_id != 'CUST-011'                                -- corrupted payload
),
deduplicated_transactions AS (
  SELECT *,
    ROW_NUMBER() OVER (
      PARTITION BY customer_id
      ORDER BY parsed_order_date DESC, revenue_amount_usd DESC
    ) AS row_num
  FROM Target_Vector_Quarantine_And_Conflict
)
```

### Step 8: Final output with market-to-region cross-validation

```sql
SELECT
  customer_id, full_name, Cleaned_email, email_validation_status,
  standardized_phone, country_market, phone_validation_status,
  parsed_order_date, standardized_region, revenue_amount_usd, transaction_status,
  CASE
    WHEN country_market = 'Morocco'
      AND standardized_region NOT IN ('Marrakech-Safi', 'Casablanca-Settat', 'Rabat-Sale-Kenitra', 'Tangier-Tetouan-Al Hoceima')
      THEN 'Warning: Moroccan Market with Non-Local Region'
    WHEN country_market = 'North America'
      AND standardized_region NOT IN ('New York', 'California', 'Texas')
      THEN 'Warning: NA Market with Non-US Region'
    WHEN country_market = 'United Kingdom'
      AND standardized_region NOT IN ('Greater London')
      THEN 'Warning: UK Market with Non-UK Region'
    WHEN country_market = 'France'
      AND standardized_region NOT IN ('Île-de-France')
      THEN 'Warning: France Market with Non-French Region'
    ELSE 'Aligned'
  END AS market_region_cross_check
FROM deduplicated_transactions
WHERE row_num = 1
ORDER BY customer_id ASC;
```

---

## 5. Spreadsheet Implementation (Google Sheets)

The spreadsheet version mirrors the SQL pipeline as a **left-to-right formula chain**: each new column reads from raw columns or earlier helper columns. Cell references (`A2`, `E2`, etc.) assume the raw data starts in row 2 and that cleaned columns sit to the right of it. Adjust them to your own layout.

> `LET`, `IFS`, `REGEXMATCH`, and `REGEXREPLACE` are native Google Sheets functions, so these formulas work as written.

### Step 1: Identity fields

| Column | Formula | Purpose |
|---|---|---|
| `customer_id` | `=IF(ISBLANK(A2), "MISSING_ID", TRIM(A2))` | Strip spaces and flag missing IDs |
| `full_name` | `=IF(OR(ISBLANK(C2), LOWER(C2)="n/a", LOWER(C2)="unknown"), "Unknown", PROPER(TRIM(C2)))` | Handle placeholders and standardize to Title Case |

### Step 2: Email validation and cleaning

```
email_validation_status:
=IF(G2="", "Invalid: Empty/Missing",
  IF(NOT(ISNUMBER(SEARCH("@", G2))), "Invalid: Format",
  IF(OR(ISNUMBER(SEARCH("test", G2)),
        ISNUMBER(SEARCH("dummy", G2)),
        ISNUMBER(SEARCH("unknown", G2))), "Invalid: Test/Dummy",
  IF(OR(ISNUMBER(SEARCH("@domain.com", G2)),
        ISNUMBER(SEARCH("@example.com", G2))), "Invalid: Test/Dummy",
  "Valid Email"))))

cleaned_email:
=IF(
  OR(NOT(ISNUMBER(SEARCH("@", E2))),
     ISNUMBER(SEARCH("test", LOWER(E2))),
     ISNUMBER(SEARCH("unknown", LOWER(E2))),
     ISNUMBER(SEARCH("invalid", LOWER(E2)))),
  "", LOWER(TRIM(E2)))
```

The status formula checks the email in order: empty, missing `@`, test or dummy keywords, and finally placeholder domains (`@domain.com`, `@example.com`). Because it catches placeholder domains automatically, records such as `CUST-006` no longer need a manual audit rule. Malformed and test entries are blanked out in `cleaned_email`, so they cannot be imported into a CRM or mailing tool.

### Step 3: Phone pipeline (helper column, standardization, market, validation)

```
digits_only (helper):
=SUBSTITUTE(SUBSTITUTE(SUBSTITUTE(SUBSTITUTE(SUBSTITUTE(H2,"(",""),")",""),"-","")," ",""),"+","")

standardized_phone:
=IF(ISBLANK(I2), "",
  IF(LEFT(I2,3)="212", CONCAT("+", I2),
  IF(LEFT(I2,2)="06",  CONCAT("+212 ", MID(I2,2,9)),
  IF(LEFT(I2,2)="07",  CONCAT("+212 ", MID(I2,2,9)),
  IF(LEN(I2)=9,        CONCAT("+212 ", I2),
  CONCAT("+", I2))))))

country_market:
=IF(ISBLANK(J2), "Unknown",
  IF(OR(LEFT(J2,4)="+212", LEFT(J2,2)="06", LEFT(J2,2)="07"), "Morocco",
  IF(LEFT(J2,2)="+1",  "North America",
  IF(LEFT(J2,4)="+44", "United Kingdom",
  "Other / International"))))

phone_validation_status:
=IF(ISBLANK(H2), "Missing",
  IF(ISNUMBER(SEARCH("555", H2)), "Test_Or_Dummy_Number",
  IF(AND(K2="Morocco",
         NOT(OR(LEFT(SUBSTITUTE(H2," ",""),6)="+2126",
                LEFT(SUBSTITUTE(H2," ",""),6)="+2127",
                LEFT(SUBSTITUTE(H2," ",""),3)="06",
                LEFT(SUBSTITUTE(H2," ",""),3)="07"))),
     "Invalid_Prefix_Or_Format", "VALID")))
```

### Step 4: Date normalization

```
parsed_order_date:
=IF(ISBLANK(M2), "Missing_Date",
  IF(REGEXMATCH(TO_TEXT(M2), "^\d{4}-\d{2}-\d{2}$"), TO_TEXT(M2),
  IF(REGEXMATCH(TO_TEXT(M2), "^\d{2}/\d{2}/\d{4}$"),
     CONCATENATE(RIGHT(TO_TEXT(M2),4), "-", MID(TO_TEXT(M2),4,2), "-", LEFT(TO_TEXT(M2),2)),
  TEXT(M2, "YYYY-MM-DD"))))
```

All dates are converted to ISO `YYYY-MM-DD`, the format that sorts correctly and loads cleanly into BI tools.

### Step 5: Region standardization

A nested `IF` with `SEARCH` performs fuzzy keyword matching, so `casa`, `Casablanca`, and `CASABLANCA` all resolve to one region. Common typos (`calfornia`) and abbreviations (`ca`, `ny`, `tx`) are also handled. Anything unrecognized falls back to `Other / International`.

```
standardized_region (abridged):
=IF(ISBLANK(O2), "Unknown",
  IF(OR(ISNUMBER(SEARCH("marrakech", LOWER(O2))), ISNUMBER(SEARCH("marrakesh", LOWER(O2)))), "Marrakesh-Safi",
  IF(OR(ISNUMBER(SEARCH("casablanca", LOWER(O2))), ISNUMBER(SEARCH("casa", LOWER(O2)))), "Casablanca-Settat",
  ... Rabat, Tangier, California, New York, Texas, London, Paris ...
  "Other / International")))
```

### Step 6: Revenue normalization and transaction status

```
revenue_amount_usd:
=IF(OR(ISBLANK(Q2), ISNUMBER(SEARCH("n/a", LOWER(Q2))), ISNUMBER(SEARCH("nul", LOWER(Q2)))), 0,
  LET(
    raw, TRIM(Q2),
    is_neg, LEFT(raw,1)="-",
    cleaned_txt, REGEXREPLACE(raw, "[^0-9.]", ""),
    num_val, VALUE(IF(cleaned_txt="", "0", cleaned_txt)),
    signed_val, IF(is_neg, -num_val, num_val),
    IF(OR(ISNUMBER(SEARCH("mad", LOWER(raw))), ISNUMBER(SEARCH("dh", LOWER(raw)))), signed_val / 9.82,
    IF(OR(ISNUMBER(SEARCH("€", raw)), ISNUMBER(SEARCH("eur", LOWER(raw)))), signed_val * 1.1177,
    IF(OR(ISNUMBER(SEARCH("£", raw)), ISNUMBER(SEARCH("gbp", LOWER(raw)))), signed_val * 1.28,
    signed_val)))))

transaction_status:
=IF(OR(ISBLANK(Q2), ISNUMBER(SEARCH("test", LOWER(F2))), ISNUMBER(SEARCH("dummy", LOWER(F2)))), "Test_Data",
  IF(R2 < 0, "Refund / Negative",
  IF(OR(R2 = 0, ISNUMBER(SEARCH("n/a", LOWER(Q2)))), "Missing Revenue",
  "Standard Sale")))
```

The `LET` function names each intermediate value (`raw`, `is_neg`, `num_val`, ...), which keeps a long formula readable and preserves the sign of refunds.

### Step 7: Cross-field validation

`market_region_cross_check` confirms that the currency notation agrees with the region (for example, `MAD` with a Moroccan region, `USD` or `$` with a US state). Matching rows return `Match`, test rows return `Exempt - Test`, and everything else returns `Mismatch` for review.

### Step 8: Manual audit rules (duplicates and quarantine)

| Audit Rule | Records | Action | Business Rationale |
|---|---|---|---|
| Exact duplicate ingestion | CUST-001, CUST-002 | Removed identical re-submissions (same ID, email, date, revenue) | Order counts reflect true unique orders |
| Data integrity correction | CUST-003 | Restored a valid $75.25 transaction that an earlier cleaning step had zeroed out | Raw-to-clean parity: real sales are never lost |
| Sandbox and test quarantine | CUST-004, CUST-007 | Screened out test accounts, anonymous names, dummy records | Dashboards show only real customer sales |
| Duplicate and conflict resolution | CUST-010 | Collapsed repeated IDs on identical timestamps with conflicting phone data | Customer counts stay accurate |
| Corrupted payload isolation | CUST-011 | Flagged missing name plus invalid email despite carrying revenue | Incomplete records are isolated before aggregation |
| Sandbox and test quarantine | CUST-016 | Removed zero-revenue records with placeholder phones | Same as above |

> **Placeholder domains are now automated.** The former manual rule for `CUST-006` (placeholder email domain) was removed. The `email_validation_status` formula and the SQL `REGEXP_CONTAINS` check now flag `@domain.com` and `@example.com` addresses as `Invalid: Test/Dummy` for every record, not only one ID.

---

## 6. Verification & Business Value

### Verification checks

| Check | Method |
|---|---|
| **Row reconciliation** | Compare raw row count to cleaned count; the difference must equal duplicates plus quarantined records |
| **Revenue parity** | Confirm cleaned total (excluding quarantined records) reconciles to the raw total after currency conversion |
| **Email flags reviewed** | `SELECT * WHERE email_validation_status != 'Valid Email' AND Cleaned_email IS NOT NULL` lists flagged emails that were not nulled (see known limitation 7) |
| **Unique keys** | `SELECT customer_id, COUNT(*) ... GROUP BY 1 HAVING COUNT(*) > 1` should return no rows |
| **Date integrity** | No `NULL` in `parsed_order_date` for non-quarantined records |
| **SQL vs Sheets parity** | Run both pipelines on the same input and diff the outputs (see known limitations below) |

### What the cleaned data enables

- **Reliable reporting.** Revenue is in a single currency, dates are ISO-formatted, and regions are consistent, so BI tools can group and trend without further cleanup.
- **Safe CRM imports.** Invalid emails are nulled, phones are in a uniform `+CC` format, and validation flags let CRM and marketing teams filter to contactable customers.
- **Trustworthy KPIs.** Duplicates and test accounts are removed, so order counts, customer counts, and average order value are not inflated.
- **Built-in data governance.** Flag columns (`phone_validation_status`, `transaction_status`, `market_region_cross_check`) turn quality issues into a reviewable queue rather than hidden errors.
- **Segmentation by market.** `country_market` and `standardized_region` support localized campaigns and regional performance analysis.

### Known limitations and next steps

Being explicit about trade-offs is part of good data work. These are the areas I would address in a production version:

1. **Ambiguous dates.** The SQL parser tries `MM/DD/YYYY` before `DD/MM/YYYY`, while the spreadsheet formula reads `DD/MM/YYYY`. A value like `10/02/2026` therefore resolves to different dates in each tool. A production pipeline should fix the format per source system.
2. **Deduplication grain.** The SQL partitions by `customer_id` only, which keeps a single row per customer. That is right for this exercise, but a true order table should partition by `customer_id`, `order_date`, and `revenue_amount` (or an order ID) so legitimate repeat purchases are preserved.
3. **Hard-coded quarantine list.** Test IDs are listed explicitly. A more scalable approach is rule-based detection (name, email, and phone patterns) feeding a quarantine table.
4. **Region spelling differences.** The SQL outputs `Marrakech-Safi` and `Rabat-Sale-Kenitra`, while the sheet outputs `Marrakesh-Safi` and `Rabat-Salé-Kénitra`. Align both to one reference list before joining outputs.
5. **Cross-check coverage.** Markets classified as `Other / International` are not region-checked, so `CUST-003` returns `Aligned`. A lookup table of market-to-region pairs would close this gap.
6. **Static exchange rates.** Rates are hard-coded constants. A dated FX rate table would make conversions auditable and reproducible.
7. **Email status vs. cleaned value.** The status check also flags `dummy`, `fake`, `noreply`, and placeholder domains, but `Cleaned_email` still only blanks out addresses containing `test`, `unknown`, or `invalid`, or lacking `@` or a dot. A placeholder-domain email such as `mary.b@domain.com` is therefore labeled `Invalid: Test/Dummy` yet keeps its cleaned value. Aligning the two conditions (or setting `Cleaned_email` to `NULL` whenever the status is not `Valid Email`) would close the gap.
8. **Status labels differ between tools.** SQL uses `Invalid: Missing` and `Invalid: Missing Format`; the sheet uses `Invalid: Empty/Missing` and `Invalid: Format`. Use one shared set of labels before comparing outputs.

---

## Repository Structure

```
├── README.md
├── sql/
│   └── clean_crm_transactions.sql
├── spreadsheets/
│   └── crm_cleanup_workbook.xlsx
└── data/
    ├── raw_crm_transactions.csv
    └── cleaned_crm_transactions.csv
```

## Skills Demonstrated

`BigQuery SQL` · `CTE pipelines` · `Regex` · `Window functions` · `Data validation` · `Google Sheets formulas` · `Data quality auditing` · `Technical documentation`

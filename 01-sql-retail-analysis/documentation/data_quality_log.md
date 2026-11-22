:::writing{variant="document" id="58392"}
# Data Quality Log

## Project 01 — UK Retail Business Intelligence

**Dataset:** Online Retail II  
**Database:** SQLite  
**Assessment date:** 22 November 2026

## 1. Dataset validation

Both imported tables were checked using SQL COUNT queries.

| Table | Expected records | Actual records |
|---|---:|---:|
| retail_2009_2010 | 525,461 | 525,461 |
| retail_2010_2011 | 541,910 | 541,910 |
| **Total** | **1,067,371** | **1,067,371** |

The imported record counts match the expected dataset totals.

## 2. Missing Customer IDs

Missing values were investigated using SQL checks for NULL and blank strings.

| Metric | 2009–2010 | 2010–2011 | Total |
|---|---:|---:|---:|
| Total records | 525,461 | 541,910 | 1,067,371 |
| NULL Customer IDs | 0 | 0 | 0 |
| Blank Customer IDs | 107,927 | 135,080 | 243,007 |
| Valid Customer IDs | 417,534 | 406,830 | 824,364 |

**Overall missing Customer IDs: 22.76%**

## 3. Data quality finding

The imported dataset contains missing Customer IDs stored as blank strings rather than SQL NULL values.

The initial COUNT(column) check did not identify these blank strings because COUNT(column) includes empty strings.

A revised SQL query using TRIM() and CAST() was used to identify missing values correctly.

## 4. Business impact

Transactions without Customer IDs cannot reliably be linked to individual customers.

This affects:
- Customer lifetime value analysis.
- Repeat purchase analysis.
- Customer segmentation.
- Customer retention calculations.

However, these transactions can still contribute to suitable sales, product and geographical analyses.

## 5. Data handling decision

- Retain transactions with missing Customer IDs.
- Do not delete records solely because Customer ID is missing.
- Exclude missing Customer IDs from customer-level calculations.
- Investigate other missing fields and unusual transaction values before final analysis.

## 6. Outstanding checks

- Identify duplicate transaction records.
- Investigate negative quantities and cancellations.
- Check zero and negative unit prices.
- Validate date formats and date ranges.
- Check missing values in other columns.

**Status:** Initial data profiling in progress.
:::

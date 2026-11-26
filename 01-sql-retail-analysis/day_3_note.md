# Day 3 — SQL Data Cleaning Strategies and Validation Rules

## 1. Learning objectives

The aim of Day 3 was to investigate data-quality issues in the Online Retail II dataset using SQLite.

The work focused on:
- Creating a separate working table.
- Checking data types and values.
- Identifying potential duplicate transactions.
- Investigating negative quantities and prices.
- Standardising product descriptions.
- Identifying missing descriptions and Customer IDs.
- Measuring data completeness by country.
- Making evidence-based data-cleaning decisions.

**Database:** `uk_retail_analysis.db`  
**Working table:** `retail_2009_2010_clean`  
**Dataset period:** 2009–2010  
**SQL environment:** SQLiteStudio 3.4.21

## 2. Working table and data validation

A separate working table was created to protect the original dataset.

```sql
CREATE TABLE retail_2009_2010_clean AS
SELECT *
FROM retail_2009_2010;
```

| Check | Result |
|---|---:|
| Original table records | 525,461 |
| Working table records | 525,461 |
| Columns | 8 |
| Quantity storage type | TEXT |
| Price storage type | TEXT |

All 525,461 Quantity and Price values were stored as TEXT.

Basic conversion screening found:
- Suspicious Quantity values: 0
- Suspicious Price values: 0

This screening did not prove that every value was analytically valid. Numeric conversion and validation remain important before calculations.

## 3. Quantity and Price investigation

The following combinations were identified:

| Quantity | Price | Records |
|---|---|---:|
| Negative | Positive | 10,205 |
| Negative | Zero | 2,121 |
| Positive | Negative | 3 |
| Positive | Positive | 511,566 |
| Positive | Zero | 1,566 |
| **Total** | | **525,461** |

Three records had positive quantities and negative prices. All were described as `Adjust bad debt`.

Their combined signed line value was **-£136,552.02**.

These appear consistent with possible accounting adjustments, but their exact treatment was not verified. They were retained for investigation rather than classified as errors.

## 4. Missing StockCodes

| Metric | Records |
|---|---:|
| Total records | 525,461 |
| NULL StockCodes | 0 |
| Blank StockCodes | 0 |
| Valid StockCodes | 525,461 |

There were no missing or blank StockCodes. However, populated codes still require business validation.

## 5. Potential duplicate transactions

Identical records were identified by grouping across all eight columns.

| Metric | Result |
|---|---:|
| Duplicate groups | 6,418 |
| Extra duplicate records | 6,865 |
| Highest occurrences in one group | 6 |

A duplicate group represents a set of identical records. Extra duplicate records count the occurrences beyond one retained record per group.

These are potential duplicates, not confirmed accidental transactions. No records were deleted.

## 6. Product description formatting

### Leading and trailing spaces

| Metric | Result |
|---|---:|
| Total records | 525,461 |
| Descriptions with leading or trailing spaces | 99,583 |
| Percentage affected | 18.95% |

The `TRIM()` function was used to identify unnecessary spaces at the beginning and end of descriptions.

### Repeated internal spaces

| Metric | Result |
|---|---:|
| Total records | 525,461 |
| Descriptions with repeated internal spaces | 30,418 |
| Percentage affected | 5.79% |

The `INSTR()` function was used to detect two consecutive spaces inside descriptions.

Examples included:
- `LUNCH BAG  BLACK SKULL`
- `HAND OVER THE CHOCOLATE   SIGN`
- `STRAWBERRY   PICNIC BAG`

### Description standardisation test

Nested `REPLACE()` functions and `TRIM()` were tested to reduce repeated spaces and remove leading and trailing spaces.

The validation query returned:

| Metric | Result |
|---|---:|
| Records checked | 525,461 |
| Records still containing repeated spaces | 0 |

The transformation successfully removed repeated spaces in the tested dataset.

The original descriptions were preserved. No permanent table update was performed.

### Description variations

A comparison using `TRIM(Description)` identified different original descriptions that became identical after trimming.

Examples:
- `BATHROOM METAL SIGN`
- `3 STRIPEY MICE FELTCRAFT`
- `WHITE CHERRY LIGHTS`
- `RED/WHITE DOT MINI CASES`

This demonstrated how inconsistent spacing can split the same product into separate groups during analysis.

## 7. Missing product descriptions

| Metric | Result |
|---|---:|
| Total records | 525,461 |
| NULL descriptions | 0 |
| Blank descriptions | 2,928 |
| Non-blank descriptions | 522,533 |

The 2,928 blank descriptions represented approximately 0.56% of the dataset.

Investigation showed that all 2,928 blank-description records also had missing Customer IDs.

The records were retained because StockCodes may still provide useful product information.

## 8. Missing Customer IDs

The original missing Customer ID count was:

**107,927 records.**

Among these:

| Description status | Records |
|---|---:|
| Blank descriptions | 2,928 |
| Valid descriptions | 104,999 |
| Total | 107,927 |

All records with blank descriptions had missing Customer IDs. However, only approximately 2.71% of records with missing Customer IDs had blank descriptions.

This showed that missing customer information was a much wider issue than missing product descriptions.

## 9. Missing Customer IDs by country

The largest numbers of missing Customer IDs were:

| Country | Missing Customer IDs |
|---|---:|
| United Kingdom | 106,429 |
| EIRE | 960 |
| United Arab Emirates | 114 |
| Portugal | 77 |
| Hong Kong | 76 |
| Bahrain | 65 |
| France | 62 |

The United Kingdom accounted for approximately 98.61% of all records with missing Customer IDs.

This is a volume comparison, not a country-level missing-data rate.

### Missing Customer ID percentage

| Country | Missing percentage |
|---|---:|
| Lebanon | 100% |
| Hong Kong | 100% |
| Bermuda | 100% |
| Bahrain | 60.75% |
| RSA | 41.44% |
| United Arab Emirates | 26.39% |
| United Kingdom | 21.91% |
| EIRE | 9.93% |
| Unspecified | 9.68% |
| Portugal | 6.99% |
| Nigeria | 6.25% |
| Sweden | 2.11% |
| France | 1.07% |

Some countries had small sample sizes. Their percentages should therefore be interpreted cautiously.

### Valid Customer ID percentage

Examples from the country completeness analysis:

| Country | Total records | Valid Customer IDs | Valid percentage |
|---|---:|---:|---:|
| Germany | 8,129 | 8,129 | 100% |
| Spain | 1,278 | 1,278 | 100% |
| France | 5,772 | 5,710 | 98.93% |
| EIRE | 9,670 | 8,710 | 90.07% |
| United Kingdom | 485,852 | 379,423 | 78.09% |
| United Arab Emirates | 432 | 318 | 73.61% |
| RSA | 111 | 65 | 58.56% |
| Bahrain | 107 | 42 | 39.25% |
| Lebanon | 13 | 0 | 0% |
| Hong Kong | 76 | 0 | 0% |
| Bermuda | 34 | 0 | 0% |

The United Kingdom had the largest volume of valid Customer IDs, but Germany had 100% completeness among the countries with substantial transaction volumes shown.

## 10. Data-cleaning decisions

The investigation produced the following decisions:

| Issue | Decision |
|---|---|
| TEXT storage for Quantity and Price | Validate and convert for numerical analysis |
| Negative quantities | Investigate business meaning; do not automatically delete |
| Negative prices | Retain and investigate accounting adjustments |
| Potential duplicate records | Flag for review; do not automatically delete |
| Leading and trailing spaces | Standardise in a separate cleaned description |
| Repeated internal spaces | Standardise and validate |
| Blank descriptions | Retain and flag |
| Missing Customer IDs | Retain for suitable non-customer-level analysis |
| Missing customer information by country | Monitor using both counts and percentages |

## 11. Main business findings

The investigation identified several data-quality issues that could affect business reporting.

First, inconsistent product descriptions can cause the same product to appear under multiple text values, affecting product grouping and reporting.

Second, missing Customer IDs affect customer-level analysis. In particular, 107,927 records cannot be reliably attributed to an identified customer.

Third, missing customer information is concentrated in UK transactions by volume, but the missing-data rate differs substantially between countries.

Finally, potential duplicate records and unusual financial values require investigation rather than automatic deletion.

## 12. SQL functions and concepts practised

| SQL function or concept | Purpose |
|---|---|
| `COUNT()` | Count records and groups |
| `SUM()` | Calculate totals |
| `MAX()` | Find the highest occurrence count |
| `CASE WHEN` | Apply conditional logic |
| `CAST()` | Convert values to another data type |
| `typeof()` | Inspect SQLite storage types |
| `TRIM()` | Remove leading and trailing spaces |
| `REPLACE()` | Replace specified text |
| `INSTR()` | Find text within a string |
| `GROUP BY` | Group records for analysis |
| `HAVING` | Filter grouped results |
| `COUNT(DISTINCT ...)` | Count distinct values |
| `ROUND()` | Round calculated percentages |
| `ORDER BY` | Sort query results |
| `LIMIT` | Restrict returned rows |

## 13. Final conclusion

Day 3 focused on investigating and validating data quality without changing the original source data.

The working table was preserved, potential duplicates were identified, product descriptions were tested for standardisation, and missing customer information was investigated across countries.

The most important lesson is that data cleaning should be evidence-based. Missing values, unusual prices and identical transactions should not automatically be treated as errors.

**Status:** Investigation and validation completed. Permanent cleaning and final analytical table creation remain outstanding.

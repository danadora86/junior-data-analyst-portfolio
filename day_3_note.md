# Day 3 — SQL Data Cleaning Strategies and Validation Rules

## 1. Learning objectives

The purpose of this session was to understand how to prepare a dataset for analysis without modifying or damaging the original source data.

The main objectives were:

- Create a separate working table.
- Inspect SQLite column storage types.
- Convert text values into numeric values when performing calculations.
- Identify unusual quantity and price combinations.
- Investigate negative unit prices.
- Record data-quality findings and recommended actions.

## 2. Creating a working copy

A separate working table named `retail_2009_2010_clean` was created from the original `retail_2009_2010` table.

```sql
CREATE TABLE retail_2009_2010_clean AS
SELECT *
FROM retail_2009_2010;
```

**Result:** Both tables contained 525,461 records.

**Learning point:** Keeping the original table unchanged makes it possible to compare results and maintain an audit trail.

## 3. Inspecting column storage types

The SQLite storage types for `Quantity` and `Price` were inspected using `typeof()`.

**Result:** All 525,461 records stored both fields as TEXT.

This means numeric-looking values should be explicitly converted before numeric comparisons.

For example:

```sql
CAST(Price AS REAL)
```

**Learning point:** A value that looks like a number is not necessarily stored as a numeric data type.

## 4. Basic conversion screening

A screening query was used to check whether values could be converted into numbers.

**Result:** 525,461 records were checked, with zero suspicious Quantity values and zero suspicious Price values in this basic screening.

This does not prove that every value is suitable for every analysis, but it provides an initial validation result.

## 5. Quantity and price classification

Transactions were grouped according to whether their quantities and prices were negative, zero or positive.

| Quantity and price pattern | Records |
|---|---:|
| Negative quantity, positive price | 10,205 |
| Negative quantity, zero price | 2,121 |
| Positive quantity, negative price | 3 |
| Positive quantity, positive price | 511,566 |
| Positive quantity, zero price | 1,566 |
| **Total** | **525,461** |

**Interpretation:** Negative and zero values are not automatically errors. They may represent cancellations, adjustments or other operational transactions and require investigation.

## 6. Query 8 — Investigating negative unit prices

**Validation rule:** Flag records where `Price` is below zero.

The query returned three records.

| Invoice | Description | Quantity | Unit price |
|---|---|---:|---:|
| A506401 | Adjust bad debt | 1 | -53,594.36 |
| A516228 | Adjust bad debt | 1 | -44,031.79 |
| A528059 | Adjust bad debt | 1 | -38,925.87 |

All three transactions were recorded in the United Kingdom and had blank Customer IDs.

**Combined signed line value:** -£136,552.02.

### Business interpretation

The repeated description `Adjust bad debt`, together with the negative prices, suggests these transactions may represent bad-debt accounting adjustments or credits rather than ordinary product sales.

However, the exact accounting treatment cannot be confirmed from the dataset alone.

### Recommended action

Retain the records and investigate their accounting treatment separately. Do not automatically delete them or classify them as errors.

**Investigation status:** Flagged for further investigation; not confirmed as errors.

## 7. Key learning points

1. Preserve the original dataset before applying cleaning rules.
2. SQLite storage types affect comparisons and calculations.
3. Use `CAST()` when numeric-looking text needs numeric treatment.
4. Validation rules identify records that need attention; they do not automatically prove an error.
5. Business context is essential when interpreting unusual transactions.
6. Documenting decisions makes data analysis more transparent and reproducible.

## 8. Outstanding work

- Continue the remaining Day 3 validation exercises.
- Review missing or invalid StockCodes.
- Investigate duplicates without automatically deleting records.
- Compare the original and working tables after applying any approved cleaning rules.
- Complete the Day 3 data-quality documentation.

## 9. Dataset source

Chen, D. (2012). *Online Retail II* [Dataset]. UCI Machine Learning Repository.

https://doi.org/10.24432/C5CG6D

Licence: Creative Commons Attribution 4.0 International (CC BY 4.0).

---

**End of Day 3 learning record — work in progress.**

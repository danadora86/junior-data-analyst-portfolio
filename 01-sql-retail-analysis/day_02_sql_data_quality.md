# Day 2 --- SQL Data Quality and Data Cleaning Fundamentals

**Project:** Online Retail II --- UK Retail Analysis\
**Dataset:** UCI Machine Learning Repository, Online Retail II (Dataset
502)\
**Study date:** 23 November 2026\
**Database:** `uk_retail_analysis.db`\
**Tables:** `retail_2009_2010` and `retail_2010_2011`\
**Tool:** SQLiteStudio 3.4.21

------------------------------------------------------------------------

## 1. Learning objectives

This session focused on identifying and investigating data-quality
issues before building sales reports or drawing business conclusions.

Skills practised: - identifying missing values, including blank
strings; - detecting fully identical transaction rows; - investigating
cancellation records and negative quantities; - inspecting SQLite
storage types and converting text values for numeric comparisons; -
identifying missing and suspicious product descriptions; - investigating
inconsistent descriptions associated with the same stock code; - making
evidence-based decisions without deleting or changing source records
prematurely.

## 2. Dataset overview

  Table                        Records
  -------------------- ---------------
  `retail_2009_2010`           525,461
  `retail_2010_2011`           541,910
  **Total**              **1,067,371**

Columns used: `Invoice`, `StockCode`, `Description`, `Quantity`,
`InvoiceDate`, `Price`, `Customer ID`, `Country`.

The source data was retained unchanged throughout this session.

## 3. Missing Customer IDs

The check counted `NULL` values and blank strings separately. The
imported missing Customer IDs were blank strings rather than SQL `NULL`
values.

  Table          Total records   NULL IDs     Blank IDs     Valid IDs
  ------------ --------------- ---------- ------------- -------------
  2009--2010           525,461          0       107,927       417,534
  2010--2011           541,910          0       135,080       406,830
  **Total**      **1,067,371**      **0**   **243,007**   **824,364**

Overall, 243,007 records (approximately **22.76%**) have a blank
Customer ID.

**Business interpretation and decision:** A blank Customer ID prevents
reliable customer-level attribution, but it does not automatically make
the transaction unusable. Retain these records for analyses that do not
require customer identification. Exclude them only from analyses that
genuinely require a valid Customer ID, and document the exclusion.

## 4. Duplicate transaction investigation

The duplicate check grouped records using all eight available columns
and retained groups with more than one occurrence. It calculated the
number of duplicate groups and the number of extra occurrences beyond
the first record in each group.

### SQL pattern

``` sql
-- PURPOSE:
-- Find groups of completely identical transaction rows.
-- METHOD:
-- Group by every source column and retain groups occurring
-- more than once.
-- BUSINESS RELEVANCE:
-- Repeated transaction rows may affect counts and summaries.
-- IMPORTANT NOTES:
-- Identical rows are candidates for investigation, not
-- automatically confirmed data-entry errors.

SELECT
    COUNT(*) AS duplicate_groups,
    SUM(record_count - 1) AS extra_duplicate_rows
FROM (
    SELECT
        "Invoice", "StockCode", "Description", "Quantity",
        "InvoiceDate", "Price", "Customer ID", "Country",
        COUNT(*) AS record_count
    FROM retail_2009_2010
    GROUP BY
        "Invoice", "StockCode", "Description", "Quantity",
        "InvoiceDate", "Price", "Customer ID", "Country"
    HAVING COUNT(*) > 1
);
```

The same query was run against `retail_2010_2011`, changing the table
name.

  Table            Duplicate groups   Extra duplicate rows
  -------------- ------------------ ----------------------
  2009--2010                  6,418                  6,865
  2010--2011                  4,879                  5,268
  **Combined**           **11,297**             **12,133**

The extra-row count is not the same as the number of duplicate groups. A
group occurring six times contributes five extra rows.

### Provisional financial impact estimate

For 2009--2010, the estimated value associated with extra occurrences
was calculated by multiplying the line value by the number of
occurrences beyond the first.

``` sql
-- PURPOSE:
-- Estimate the signed value associated with extra occurrences
-- in fully identical transaction groups.
-- IMPORTANT NOTES:
-- This is provisional, not a confirmed loss or accounting error.

SELECT
    SUM((record_count - 1) * Quantity * Price)
        AS potential_duplicate_value
FROM (
    SELECT
        "Invoice", "StockCode", "Description", "Quantity",
        "InvoiceDate", "Price", "Customer ID", "Country",
        COUNT(*) AS record_count
    FROM retail_2009_2010
    GROUP BY
        "Invoice", "StockCode", "Description", "Quantity",
        "InvoiceDate", "Price", "Customer ID", "Country"
    HAVING COUNT(*) > 1
);
```

**Actual result:** `33,709.07`.

This is a provisional gross value associated with extra repeated rows,
assuming the additional occurrences are accidental duplicates. That
assumption has not been verified. It may include cancellation or
return-related records and needs reconciliation before being treated as
a financial impact.

**Decision:** Do not delete identical rows automatically. Investigate
their context and retain an audit trail before any cleaning action.

## 5. Cancellation invoices and negative quantities

The dataset convention used in the investigation is that invoice numbers
beginning with `C` indicate cancellations.

### Cancellation counts

``` sql
-- PURPOSE: Count records whose invoice number begins with C.
SELECT COUNT(*) AS cancellation_records
FROM retail_2009_2010
WHERE "Invoice" LIKE 'C%';
```

**Actual result:** 10,206 cancellation-related records.

``` sql
-- PURPOSE: Count distinct cancellation-prefixed invoice numbers.
SELECT COUNT(DISTINCT "Invoice") AS unique_cancelled_invoices
FROM retail_2009_2010
WHERE "Invoice" LIKE 'C%';
```

**Actual result:** 4,592 distinct cancellation-prefixed invoices, or
approximately 2.22 cancellation lines per cancellation-prefixed invoice.

### Signed cancellation totals

``` sql
-- PURPOSE: Summarise quantities and signed line values for
-- cancellation-prefixed records.
-- IMPORTANT NOTES: These are not confirmed cash refunds.

SELECT
    COUNT(*) AS cancellation_records,
    SUM(Quantity) AS total_quantity,
    SUM(Quantity * Price) AS signed_transaction_value
FROM retail_2009_2010
WHERE "Invoice" LIKE 'C%';
```

**Actual result:** 10,206 records; total quantity `-213,418`; signed
transaction value `-629,855.37`.

### SQLite storage type issue

An initial negative-quantity classification returned zero negative
records even though `SUM(Quantity)` was negative. The storage type check
was:

``` sql
-- PURPOSE: Inspect how SQLite stores values in Quantity.
SELECT
    typeof(Quantity) AS storage_type,
    COUNT(*) AS record_count
FROM retail_2009_2010
GROUP BY typeof(Quantity);
```

**Actual result:** all 525,461 `Quantity` values were stored as `text`.

Numeric-looking text must be explicitly converted before numeric
comparisons. `SUM()` can coerce numeric-looking text for arithmetic, but
a comparison such as `Quantity < 0` did not produce the intended result
with this column's storage type.

### Corrected negative-quantity classification

``` sql
-- PURPOSE: Count negative and non-negative quantities by invoice type.
-- METHOD: Convert Quantity to a numeric value before comparing it.
-- IMPORTANT NOTES: This query does not modify the source data.

SELECT
    CASE
        WHEN "Invoice" LIKE 'C%' THEN 'Cancellation'
        ELSE 'Non-cancellation'
    END AS invoice_type,
    COUNT(*) AS total_records,
    SUM(CASE WHEN CAST(Quantity AS REAL) < 0 THEN 1 ELSE 0 END)
        AS negative_quantity_records,
    SUM(CASE WHEN CAST(Quantity AS REAL) >= 0 THEN 1 ELSE 0 END)
        AS non_negative_quantity_records
FROM retail_2009_2010
GROUP BY
    CASE
        WHEN "Invoice" LIKE 'C%' THEN 'Cancellation'
        ELSE 'Non-cancellation'
    END;
```

  ---------------------------------------------------------------------------
  Invoice type            Total records           Negative       Non-negative
                                                quantities         quantities
  ------------------ ------------------ ------------------ ------------------
  Cancellation                   10,206             10,205                  1

  Non-cancellation              515,255              2,121            513,134

  **Total**                 **525,461**         **12,326**        **513,135**
  ---------------------------------------------------------------------------

### Negative quantities on non-cancellation invoices

A sample of 20 records was inspected. It included blank descriptions,
zero prices, and descriptions such as `damages` and `Dotcom multiples`.

All 2,121 negative-quantity records on non-cancellation invoices were
then grouped by price category.

``` sql
-- PURPOSE: Analyse negative quantities on non-cancellation invoices
-- according to their unit price.
-- BUSINESS RELEVANCE: Helps investigate possible operational
-- adjustments separately from ordinary sales.
-- IMPORTANT NOTES: This does not establish the records' exact purpose.

SELECT
    CASE
        WHEN CAST(Price AS REAL) = 0 THEN 'Zero price'
        ELSE 'Non-zero price'
    END AS price_category,
    COUNT(*) AS record_count,
    SUM(CAST(Quantity AS REAL)) AS total_quantity,
    SUM(CAST(Quantity AS REAL) * CAST(Price AS REAL)) AS total_value
FROM retail_2009_2010
WHERE
    "Invoice" NOT LIKE 'C%'
    AND CAST(Quantity AS REAL) < 0
GROUP BY
    CASE
        WHEN CAST(Price AS REAL) = 0 THEN 'Zero price'
        ELSE 'Non-zero price'
    END;
```

**Actual result:** 2,121 records; total quantity `-366,128`; total value
`0`. All had a zero unit price.

**Interpretation:** These could represent stock movements, damaged goods
or other operational adjustments. Their exact purpose has not been
confirmed. They have zero recorded transaction value but may still
matter to quantity or stock analysis.

### Positive-quantity cancellation exception

The one cancellation-prefixed record with a non-negative quantity was:

  Field         Value
  ------------- ------------------
  Invoice       C496350
  StockCode     M
  Description   Manual
  Quantity      1
  InvoiceDate   01/02/2010 08:24
  Price         373.57
  Customer ID   Blank
  Country       United Kingdom

Cancellation records grouped by quantity sign produced:

  Quantity type        Records   Total quantity      Signed value
  --------------- ------------ ---------------- -----------------
  Negative              10,205         -213,419       -630,228.94
  Positive                   1                1            373.57
  **Total**         **10,206**     **-213,418**   **-629,855.37**

The positive manual record reconciles the signed cancellation total to
the earlier result.

**Decision:** Flag the manual record for investigation. The signed total
is not proof of actual refunds, cash paid back or accounting losses. Do
not automatically classify the manual record as an ordinary sale.

## 6. Missing product descriptions

### Check NULL, blank and valid descriptions

``` sql
-- PURPOSE: Count missing and valid product descriptions.
-- METHOD: Check NULL separately from blank text; TRIM detects
-- empty strings and strings containing spaces.

SELECT
    COUNT(*) AS total_records,
    SUM(CASE WHEN Description IS NULL THEN 1 ELSE 0 END)
        AS null_descriptions,
    SUM(CASE WHEN TRIM(Description) = '' THEN 1 ELSE 0 END)
        AS blank_descriptions,
    SUM(
        CASE
            WHEN Description IS NOT NULL AND TRIM(Description) <> ''
            THEN 1 ELSE 0
        END
    ) AS valid_descriptions
FROM retail_2009_2010;
```

**Actual result:** 525,461 total records; 0 NULL descriptions; 2,928
blank descriptions; 522,533 valid descriptions.

Approximately **0.56%** of records have blank descriptions.

**Decision:** Retain transactions in the source data. A blank
description does not necessarily make the transaction unusable if
StockCode, Quantity and Price are available. Product-description
reporting may require separate treatment.

### Investigate blank descriptions by StockCode

The blank-description records were grouped by StockCode. The 20 most
frequent stock codes were inspected.

Examples:

  StockCode     Blank-description records   Total quantity
  ----------- --------------------------- ----------------
  22950                                10              -44
  84990                                 9               90
  22139                                 8                4
  84795D                                7               53
  79321                                 7             -165
  21768                                 7               -8
  37446                                 6             -484

Missing descriptions affect multiple stock codes, and affected records
can have positive or negative total quantities.

### Look for available descriptions for affected StockCodes

A grouped query retrieved a non-blank description using `MAX()`.

Examples from the results:

  StockCode   Available description                   Blank records
  ----------- ------------------------------------- ---------------
  22950       SET OF 36 DOILIES VINTAGE CHRISTMAS                10
  84990       ?                                                   9
  22139       amazon                                              8
  84795D      missing                                             7
  79321       CHILLI LIGHTS                                       7
  21768       WALL MOUNTED VINTAGE ORGANISER                      7

**Important:** `MAX()` on text returns the alphabetically highest value
according to SQLite's text comparison rules. It does not return the most
frequent or most accurate description. A non-blank value is not
necessarily a valid product description.

## 7. Suspicious descriptions

The investigation checked `?`, `missing` and `amazon`, using `LOWER()`
and `TRIM()`.

  Suspicious description     Records   Affected StockCodes
  ------------------------ --------- ---------------------
  `?`                             45                    44
  `missing`                       26                    26
  `amazon`                         3                     2
  **Total**                   **74**                **72**

This demonstrates the difference between: - **Missing-value detection:**
checking whether a value is absent, such as `NULL` or an empty string. -
**Invalid-value detection:** checking whether a value is present but
unsuitable or meaningless for the field.

These 74 records are candidates for investigation, not automatic
deletion.

### Investigate all descriptions for suspicious StockCodes

The query retrieved every description associated with StockCodes that
had at least one suspicious value.

Examples from the results:

  ------------------------------------------------------------------------
  StockCode                           Examples found
  ----------------------------------- ------------------------------------
  16168S                              `FUNKY MONKEY GIFT BAG SMALL`,
                                      `missing`, blank

  21041                               `RED SPOTTY OVEN GLOVE DOUBLE`,
                                      `RED RETROSPOT OVEN GLOVE DOUBLE`,
                                      `missing`, blank

  22139                               Several tea-set descriptions,
                                      `amazon`, blank

  22734                               Product description, `amazon`,
                                      `amendment`, `amazon adjustment`,
                                      operational note

  47566B                              Product description, stock-credit
                                      and adjustment notes, `missing`,
                                      blank

  85099B                              Three different jumbo-bag
                                      descriptions and `missing`
  ------------------------------------------------------------------------

**Interpretation:** The Description column is not always a clean
product-name field. It can contain product names, placeholders,
variations and operational notes.

**Decision:** Preserve the original Description field. Any future
standardised description should be created in a separate cleaned field,
with mapping rules documented.

## 8. Multiple descriptions per StockCode

The final query counted distinct non-blank descriptions per StockCode,
excluding `?`, `missing` and `amazon`.

``` sql
-- PURPOSE: Identify StockCodes associated with multiple descriptions.
-- METHOD: COUNT(DISTINCT TRIM(Description)) counts unique descriptions.
-- GROUP BY groups records by product; HAVING filters grouped results.
-- IMPORTANT NOTES: Multiple descriptions are candidates for
-- investigation, not automatically confirmed errors.

SELECT
    StockCode,
    COUNT(DISTINCT TRIM(Description)) AS different_descriptions,
    COUNT(*) AS total_records
FROM retail_2009_2010
WHERE
    Description IS NOT NULL
    AND TRIM(Description) <> ''
    AND LOWER(TRIM(Description)) NOT IN ('?', 'missing', 'amazon')
GROUP BY StockCode
HAVING COUNT(DISTINCT TRIM(Description)) > 1
ORDER BY different_descriptions DESC, total_records DESC
LIMIT 20;
```

**Actual top 20 results:**

  StockCode     Different descriptions   Total records
  ----------- ------------------------ ---------------
  22423                              6           2,220
  21843                              4           1,149
  22384                              4           1,093
  20685                              4           1,078
  21523                              4             655
  22236                              4             375
  22734                              4             355
  79000                              4             338
  48173C                             4             328
  84016                              4              76
  22345                              4              60
  22344                              4              59
  22346                              4              46
  85099B                             3           2,056
  20914                              3           1,276
  22139                              3           1,194
  22138                              3           1,178
  22356                              3             881
  21524                              3             701
  22952                              3             609

**Interpretation:** Several StockCodes have multiple descriptions,
including high-volume products. This may affect product-level grouping
and reporting.

**Limitation:** Multiple descriptions are not necessarily errors. They
may reflect legitimate naming changes or variations. Examine
descriptions, frequencies and transaction dates before selecting a
standard description.

## 9. Day 2 data-quality decisions

  -----------------------------------------------------------------------
  Issue                   Finding                 Decision
  ----------------------- ----------------------- -----------------------
  Missing Customer IDs    243,007 blank IDs       Retain transactions;
                          across both tables      exclude only from
                                                  analyses requiring
                                                  customer identification

  Identical transaction   11,297 duplicate groups Investigate before
  rows                    and 12,133 extra rows   removing

  Potential duplicate     £33,709.07 in           Provisional, not
  value                   2009--2010              confirmed loss

  Quantity stored as TEXT All 525,461 Quantity    Explicit numeric
                          values in 2009--2010    conversion for numeric
                          stored as TEXT          comparisons

  Cancellation records    10,206 C-prefixed       Analyse separately;
                          records                 signed value is not
                                                  proof of actual refunds

  Negative quantities on  2,121 records, all      Investigate as possible
  non-C invoices          zero-price              operational or stock
                                                  adjustments

  Positive cancellation   One manual record,      Flag for investigation
  exception               £373.57                 

  Blank descriptions      2,928 records           Retain source records;
                                                  investigate
                                                  descriptions separately

  Suspicious descriptions 74 records across 72    Flag and investigate
                          StockCodes              

  Multiple descriptions   Multiple StockCodes     Review descriptions and
                          have more than one      dates before
                          valid description       standardising
  -----------------------------------------------------------------------

## 10. Key learning points

1.  `NULL` and an empty string are different in SQL.
2.  A missing value in one column does not automatically make an entire
    transaction unusable.
3.  `COUNT(DISTINCT ...)` counts unique values, while `COUNT(...)`
    counts non-NULL values.
4.  `GROUP BY` groups records; `HAVING` filters grouped results.
5.  SQLite storage types matter. Numeric-looking text should be
    explicitly converted before numeric comparisons.
6.  `TRIM()` removes surrounding spaces; `LOWER()` helps make text
    comparisons case-insensitive.
7.  A populated field can still contain invalid or unsuitable
    information.
8.  Duplicate-looking records need investigation before deletion.
9.  Business interpretation matters as much as writing technically
    correct SQL.
10. Preserve original data and document cleaning decisions so the work
    is traceable and reproducible.

## 11. Assessment reflection

Ten multiple-choice consolidation questions were completed. The final
assessment result was **6 correct out of 10**. Incorrect answers were
reviewed with explanations.

The questions reinforced: - the difference between `NULL` and blank
strings; - why records with missing descriptions should not
automatically be deleted; - why grouping by StockCode helps investigate
missing descriptions; - how `MAX()` behaves on text; - why description
variations require investigation; - the difference between missing and
invalid values; - why `COUNT(DISTINCT TRIM(Description))` is useful.

## 12. Next steps --- Day 3

The next session will build on this investigation: - define
data-cleaning rules; - decide how to handle missing and invalid
values; - investigate duplicates safely; - create a cleaned analytical
dataset without overwriting the source; - apply and document validation
checks.

## 13. Dataset source and attribution

Chen, D. (2012). *Online Retail II* \[Dataset\]. UCI Machine Learning
Repository. https://doi.org/10.24432/C5CG6D

Dataset page: https://archive.ics.uci.edu/dataset/502/online+retail+ii

The dataset is published under the Creative Commons Attribution 4.0
International (CC BY 4.0) licence. Retain this attribution when
redistributing or publishing derived work, and review the licence terms.

------------------------------------------------------------------------

**End of Day 2 learning record.**

All reported figures are the results recorded during this learning
session. No source records were deleted or modified during the
investigation.

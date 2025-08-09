# Data Cleaning Project

This project focuses on cleaning a layoff dataset by addressing several common data quality issues.

## Objectives

The following data problems were identified and resolved:

- Duplicate records
- Null values and inconsistent null formats
- Incorrect data types
- Non-standardised text entries
- Incomplete or partial data

## Cleaning Process

1. **Created a working table**  
   A duplicate of the raw `layoffs` table was created (`layoffs_working`) to ensure the original dataset remains unchanged.

2. **Removed duplicate records**  
   Duplicates were detected using combinations of multiple columns and removed using `ROW_NUMBER()` and `ctid`.

3. **Standardised data entries**  
   - Trimmed whitespace from `company`, `industry`, and `country` columns.
   - Normalised variations of specific terms, e.g. all variations of “Crypto” were unified to a single value.
   
4. **Corrected data types**  
   - The `date` column was converted from inconsistent text to a proper `DATE` format using `TO_DATE()` and regular expressions.

5. **Handled null and empty values**  
   - Standardised empty strings (`''`) and textual "NULL" entries to SQL `NULL`.
   - Imputed missing values in the `industry` column by matching on company names using a self-join.

6. **Deleted unrecoverable entries**  
   - Rows with both `total_laid_off` and `percentage_laid_off` missing were removed due to insufficient data.

## Tools & Environment

- SQL (PostgreSQL)
- Functions: `ROW_NUMBER()`, `CTID`, `TO_DATE()`, regex, `TRIM()`, self-joins

---

## Summary of Key Complex SQL Functions

- **`ROW_NUMBER()` window function**  
  Used to identify duplicates based on combinations of multiple fields.

- **`CTID` system column**  
  Used to uniquely reference and delete rows where no primary key exists.

- **Regular expressions (`~`)**  
  Employed to validate and filter properly formatted date strings.

- **`TO_DATE()`**  
  Converts string-formatted dates into PostgreSQL `DATE` types.

- **`TRIM()`**  
  Removes unwanted whitespace from text fields.

- **Self-JOIN update**  
  Populates missing `industry` values based on other rows with the same `company`.

---


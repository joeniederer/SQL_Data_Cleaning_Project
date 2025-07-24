# Data Cleaning Project 1

This project focuses on cleaning a layoff dataset by addressing several common data quality issues.

## Issues with the data

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
   - Normalised variations of specific terms.
   
4. **Corrected data types**  
   - The `date` column was converted from inconsistent text to a proper `DATE` format using `TO_DATE()` and regular expressions.

5. **Handled null and empty values**  
   - Standardised empty strings (`''`) and textual "NULL" entries to SQL `NULL`.
   - Imputed missing values in the `industry` column by matching on company names using a self-join.

6. **Deleted unrecoverable entries**  
   - Rows with both `total_laid_off` and `percentage_laid_off` missing were removed due to insufficient data.

## Highlight of key functions

- SQL (PostgreSQL)
- Functions: `ROW_NUMBER()`, `CTID`, `TO_DATE()`, regex, `TRIM()`, self-joins



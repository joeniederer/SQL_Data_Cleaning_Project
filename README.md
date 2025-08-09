# Layoffs Data Cleaning (PostgreSQL)

This README documents a **repeatable, step‑by‑step PostgreSQL data‑cleaning workflow** for a layoffs dataset.  
It creates a working table, loads data from a raw source, removes duplicates, standardises text fields, parses dates safely, normalises nulls, backfills missing industries via a self‑join, and finally drops rows that cannot be repaired.

> **Assumptions**
> - A raw table named `layoffs` already exists and contains the same columns used below.
> - You have permissions to create tables and update/delete rows.
> - You're using PostgreSQL 12+ (uses `LIKE ... INCLUDING ALL` and window functions).

---

## Quick start

1. Open a psql session connected to the target database.
2. (Optional but recommended) Run everything inside a transaction so you can roll back if needed:
   ```sql
   BEGIN;
   -- run the script below
   COMMIT; -- or ROLLBACK;
   ```
3. Copy/paste the **Full Script** (below) or run it from a file:
   ```bash
   psql -d your_database -f cleanup_layoffs.sql
   ```

---

## What the script does

1. **Create a working table** mirroring the raw table (`INCLUDING ALL` keeps constraints, defaults, etc.).  
2. **Load** data from `layoffs` → `layoffs_working`.  
3. **Remove duplicates** with a `ROW_NUMBER()` partition and delete using `ctid`.  
4. **Standardise text**:
   - Trim company names.
   - Consolidate `industry` variants beginning with `Crypto` → `Crypto`.
   - Trim trailing punctuation and whitespace in `country`.  
5. **Parse and convert `date`** (string → `date`) for well‑formed values like `M/D/YYYY` or `MM/DD/YYYY`.  
6. **Normalise nulls** (convert `''` and `'NULL'` text to real `NULL`s across several columns).  
7. **Backfill missing `industry`** by self‑joining on `company` where another row for the same company has a populated `industry`.  
8. **Prune unrecoverable rows** where both `total_laid_off` and `percentage_laid_off` are `NULL`.

> ⚠️ **Notes**
> - The script treats any `industry` starting with `Crypto` as `Crypto` (e.g., `Crypto / Web3` → `Crypto`). Adjust if you need finer granularity.
> - `ctid` is PostgreSQL‑specific and suitable here because no natural key exists. In production, prefer stable primary keys.
> - The `date` conversion only updates rows that strictly match `^\d{1,2}/\d{1,2}/\d{4}$`. Extend if your data has other formats.

---

## Full script

<details>
<summary>Click to expand</summary>

```sql
-- 1) Create working table mirroring the raw table
CREATE TABLE IF NOT EXISTS layoffs_working (LIKE layoffs INCLUDING ALL);

-- 2) Load data from raw table
INSERT INTO layoffs_working
SELECT *
FROM layoffs;

-- 3) Remove duplicates
-- Tag duplicates by partitioning on the set of columns that should define uniqueness.
-- Keep row_number = 1, delete the rest via ctid.
DELETE FROM layoffs_working
WHERE ctid IN (
  SELECT ctid
  FROM (
    SELECT
      ctid,
      ROW_NUMBER() OVER (
        PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, date, stage, country, funds_raised_millions
      ) AS rn
    FROM layoffs_working
  ) AS t
  WHERE rn > 1
);

-- (Optional) sanity check: expect zero rows returned
SELECT *
FROM (
  SELECT
    *,
    ROW_NUMBER() OVER (
      PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, date, stage, country, funds_raised_millions
    ) AS rn
  FROM layoffs_working
) AS chk
WHERE rn > 1;

-- 4) Standardise text fields

-- Trim company names
UPDATE layoffs_working
SET company = TRIM(company)
WHERE company IS NOT NULL AND company <> TRIM(company);

-- Consolidate industry variants starting with 'Crypto' to 'Crypto'
UPDATE layoffs_working
SET industry = 'Crypto'
WHERE industry ILIKE 'Crypto%';

-- Normalise country values: trim whitespace and trailing dots
UPDATE layoffs_working
SET country = TRIM(TRAILING '.' FROM TRIM(country))
WHERE country IS NOT NULL;

-- 5) Parse and convert date (text -> date) for values like M/D/YYYY or MM/DD/YYYY
-- First, update convertible strings into real dates in-place using to_date
UPDATE layoffs_working
SET date = TO_DATE(date, 'FMMM/FMDD/YYYY')
WHERE date IS NOT NULL
  AND date ~ '^[0-9]{1,2}/[0-9]{1,2}/[0-9]{4}$';

-- Then, ensure the column is of type DATE (safe cast only for rows that matched)
ALTER TABLE layoffs_working
  ALTER COLUMN date TYPE DATE
  USING (
    CASE
      WHEN date ~ '^[0-9]{1,2}/[0-9]{1,2}/[0-9]{4}$' THEN TO_DATE(date, 'FMMM/FMDD/YYYY')
      WHEN date ~ '^[0-9]{4}-[0-9]{2}-[0-9]{2}$' THEN date::date   -- already ISO format
      ELSE NULL::date
    END
  );

-- 6) Normalise null-like strings to actual NULLs
UPDATE layoffs_working
SET total_laid_off = NULL
WHERE total_laid_off = '' OR total_laid_off ILIKE 'null';

UPDATE layoffs_working
SET percentage_laid_off = NULL
WHERE percentage_laid_off = '' OR percentage_laid_off ILIKE 'null';

UPDATE layoffs_working
SET industry = NULL
WHERE industry = '' OR industry ILIKE 'null';

UPDATE layoffs_working
SET company = NULL
WHERE company = '' OR company ILIKE 'null';

-- 7) Backfill missing industry by company (assumes single industry per company)
UPDATE layoffs_working t1
SET industry = t2.industry
FROM layoffs_working t2
WHERE t1.company = t2.company
  AND t1.industry IS NULL
  AND t2.industry IS NOT NULL;

-- 8) Remove rows that cannot be repaired (no layoff magnitude info at all)
DELETE FROM layoffs_working
WHERE total_laid_off IS NULL
  AND percentage_laid_off IS NULL;
```

</details>

---

## Validation snippets (optional)

Use these to spot‑check results:

```sql
-- Expect no duplicates (should return zero rows)
SELECT *
FROM (
  SELECT
    *,
    ROW_NUMBER() OVER (
      PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, date, stage, country, funds_raised_millions
    ) AS rn
  FROM layoffs_working
) AS rows
WHERE rn > 1;

-- Inspect distinct industries
SELECT DISTINCT industry
FROM layoffs_working
ORDER BY 1;

-- Inspect companies with NULL industry (ideally none after backfill)
SELECT company
FROM layoffs_working
WHERE industry IS NULL
GROUP BY company
ORDER BY company;
```

---

## Troubleshooting

- **`date` parse misses some rows**: Add more `WHEN` branches to the `USING (...)` expression for additional formats.
- **You prefer a deterministic delete**: Instead of `ctid`, add a surrogate `id` primary key to `layoffs_working` and delete by `id`.
- **Multiple industries per company**: Replace the backfill step with a mapping table or a more specific join condition (e.g., company + location).

---

## License

MIT — use and adapt freely.

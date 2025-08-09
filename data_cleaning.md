
# Data Cleaning


## 1. Create Working Table

```sql
create table layoffs_working
    (like layoffs including all)
    
    -- new working table, separate from the raw table
    
select
    *
    from
    layoffs_working
    
    -- input the data from the raw table
    
insert into layoffs_working
select
    *
    from layoffs
```
    
## 2. Remove duplicates
### 2.1 Identify Duplicate Rows

```sql
    -- identify number of rows combination of variables, assuming the combinations of variables are unique
    
select
    *,
    row_number() over(
    partition by company, location, industry, total_laid_off, percentage_laid_off, date, stage, country, funds_raised_millions) as row_count
    from layoffs_working
```

```sql
    -- identify where there are more than one rows for each unique combination
    
select
    *
    from
    (select
    *,
    row_number() over(
    partition by company, location, industry, total_laid_off, percentage_laid_off, date, stage, country, funds_raised_millions) as row_number
    from layoffs_working) as rows
    where row_number > 1
```

```sql
    -- no unique identifying primary key, ctid function is required to reference the rows in question
    
select
    *
    from
    (select ctid, row_number() over(
    partition by company, location, industry, total_laid_off, percentage_laid_off, date, stage, country, funds_raised_millions)
    as row_number
    from layoffs_working)
    as sub
    where row_number > 1
```
    
### 2.2 Delete Duplicate Rows
  
```sql
    -- delete the identified duplicates using ctid

delete from layoffs_working
    where ctid IN
    (select ctid
    from
    (select ctid, row_number() over(
    partition by company, location, industry, total_laid_off, percentage_laid_off, date, stage, country, funds_raised_millions)
    as row_number
    from layoffs_working)
    as sub
    where row_number > 1)


 -- perform checks to ensure delete cleaning successful

select
    *
    from
    (select
    *,
    row_number() over(
    partition by company, location, industry, total_laid_off, percentage_laid_off, date, stage, country, funds_raised_millions) as row_number
    from layoffs_working) as rows
    where row_number > 1
```
    
## 3. Standardise Data

### 3.1 'Company' Column

```slq
update layoffs_working
    set company = trim(company)
```

### 3.2 'Industry' Columm

```sql
     -- multiple varitiations of the word 'Crpyto'
     -- identify the most common variation and standardise using this

select
    industry,
    count(industry)
    from (select industry
    from layoffs_working
    where industry like 'Crypto%') as x
    group by 1

update layoffs_working
    set industry = 'Crypto'
    where industry like 'Crypto%'
```

### 3.3 'Country' Columm

```sql
update layoffs_working
    set country = trim(trailing'.' from trim(country))
```

### 3.4 Convert 'Date' Column to Proper Date Format

```sql
    -- changing 'date' column data type from text to date
    -- there are some abnormalities in the formatting and date entries i.e '', null, 'NULL' (text)
    -- which required regex to fix

select
    to_date(date, 'FMMM/FMDD/YYYY')
    from layoffs_working
    where date is not null
    and date ~ '^\d{1,2}/\d{1,2}/\d{4}$'

update layoffs_working
    set date = to_date(date, 'FMMM/FMDD/YYYY')
    where date is not null
    and date ~ '^\d{1,2}/\d{1,2}/\d{4}$'

alter table layoffs_working
alter column date type data
    using (
    case
    when date ~ '^\d{1,2}/\d{1,2}/\d{4}$'
    then to_date(date, 'FMMM/FMDD/YYYY')
    else
    end
    )
```    

    
## 4. Null and Missing Values

### 4.1 Standardise Null Variants

```sql
    -- there are different variations of null ('', 'NULL') which need to be standardised

select total_laid_off
    from layoffs_working
    where total_laid_off ilike 'null'
    
update layoffs_working
    set total_laid_off = null
    where total_laid_off = ''
    or total_laid_off ilike 'null';

update layoffs_working
    set percentage_laid_off = null
    where percentage_laid_off = ''
    or percentage_laid_off ilike 'null';

update layoffs_working
    set industry = null
    where industry = ''
    or industry ilike 'null';

update layoffs_working
    set company = null
    where company = ''
    or company ilike 'null';

    -- now the null functions should work due to standardisation
```

## 4.2 Populating Missing Values



```sql
     -- looking through the data we can see there are some missing entries that can be populated

select
    company,
    industry
    from layoffs_working
    where industry is null

    -- for instance we can see that some 'industry' entries are missing for some 'company' entries, however they aren't missing on other rows of the same company (for instance 'Airbnb')

select
    company,
    industry
    from layoffs_working
    where company = 'Airbnb'

    -- execute a join function to identify where data can be populated
    
select
    *
    from layoffs_working as t1
    join layoffs_working as t2
    on t1.company = t2.company
    
    -- assuming that each company name is only operating within one industry, we can use this join to match the tables
    -- however what we want it to also match up on the condition that t1 missing industry values are matched with t2 values
    
select
    t1.industry,
    t2.industry
    from layoffs_working as t1
    join layoffs_working as t2
    on t1.company = t2.company
    where t1.industry is null
    and t2.industry is not null
    
    -- the two columns are matched up, t1 is null, t2 is populated

    -- this can be used to update the main table
    

update layoffs_working t1
    set industry = t2.industry
    from layoffs_working t2
    where t1.company = t2.company
    and t1.industry is null
    and t2.industry is not null;
```

## 5. Removing Unrecoverable Data

```sql
    -- data that can't be recovered, calculated or scraped from the internet, we can delete because we still have the raw data

delete
    from layoffs_working
    where total_laid_off is null
    and percentage_laid_off is null;
```

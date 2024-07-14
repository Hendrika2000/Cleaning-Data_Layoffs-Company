# Layoffs Company Data Cleaning

## Table of Contents

- [Project Overview](#project-overview)
- [Data Sources](#data-sources)

### Project Overview
---

This time we want to cleaning the layoffs company data. So, the data can be used for the analysis.


### Data Sources

Sales Data: The primary dataset used for this analysis is the "Layoffs.csv" file, containing detailed information about the total laid off of the companies.

### Tools

Mysql

### Data Cleaning/Preparation

In the initial data preparation phase, we performed the following tasks:
1. Remove Duplicates
2. Standardize the Data
3. Null Values or blank values
4. Remove Any Columns or Rows

SELECT * FROM layoffs;

First thing we create a staging table, a table with the raw data in case something happens
```sql
CREATE TABLE layoffs_staging
LIKE layoffs;
```
```sql
INSERT layoffs_staging
SELECT * FROM layoffs;
```

1. Remove Duplicates

Let's check for duplicates
```sql
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, 'date', stage, 
country, funds_raised_millions)
 AS 'row_num'
FROM layoffs_staging;
```
these are the ones we want to delete where the row number is > 1 
```sql
WITH duplicate_cte AS
(
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, 'date', stage, 
country, funds_raised_millions)
 AS 'row_num'
FROM layoffs_staging
)
SELECT *
FROM duplicate_cte
WHERE row_num > 1;
```
Let's just look at Casper to confirm
```sql
SELECT * FROM layoffs_staging
WHERE company = 'Casper';
```
 we need to create a new column and add those row numbers in. Then delete where row numbers are over 1, then delete that column
 ```sql
CREATE TABLE layoffs_staging2 (
company text,
location text,
industry text,
total_laid_off int DEFAULT NULL,
percentage_laid_off text,
date text,
stage text,
country text,
funds_raised_millions int DEFAULT NULL,
row_num INT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```
```sql
INSERT INTO layoffs_staging2
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, 'date', stage, 
country, funds_raised_millions)
 AS row_num
FROM layoffs_staging;

SELECT * FROM layoffs_staging2
WHERE row_num > 1;
```
now that we have this we can delete row where row_num is greater than 1
```sql
DELETE 
FROM layoffs_staging2
WHERE row_num > 1;
```

2.Standardize data
```sql
SELECT company, (TRIM(company)) 
FROM layoffs_staging2;
```
```sql
UPDATE layoffs_staging2
SET company = TRIM(company);
```
```sql
SELECT *
FROM layoffs_staging2
WHERE industry LIKE 'Crypto%';
```
```sql
UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry LIKE 'Crypto%';
```
```sql
SELECT DISTINCT location
FROM layoffs_staging2
ORDER BY 1;
```
```sql
SELECT DISTINCT country
FROM layoffs_staging2
ORDER BY 1;
```
```sql
SELECT *
FROM layoffs_staging2
WHERE country LIKE 'United States%';
```
```sql
SELECT DISTINCT country, TRIM(TRAILING '.' FROM country)
FROM layoffs_staging2
WHERE country LIKE 'United States%';
```
```sql
UPDATE layoffs_staging2
SET country = TRIM(TRAILING '.' FROM country)
WHERE country LIKE 'United States%';
```
```sql
SELECT date,
STR_TO_DATE(date, '%m/%d/%Y')
FROM layoffs_staging2;
```
```sql
UPDATE layoffs_staging2
SET date = STR_TO_DATE(date, '%m/%d/%Y');
```
```sql
convert the data type properly
ALTER TABLE layoffs_staging2
MODIFY COLUMN date date;
```
```sql
it look like we have some null and empty rows in industry column, so let's take a look
SELECT DISTINCT industry
FROM layoffs_staging2
ORDER BY 1;
```
```sql
we need to set the blanks to nulls since those are typically easier to work with
UPDATE layoffs_staging2
SET industry = NULL
WHERE industry = '';
```
```sql
SELECT * 
FROM layoffs_staging2
WHERE company = 'Airbnb';
```
Airbnb ia a travel industry, but this one just isn.t populated
so we need to write a query that if there is another row with the same company name, it will update to the non-null industry values
```sql
SELECT *
FROM layoffs_staging2 t1
JOIN layoffs_staging2 t2
	ON t1.company = t2.company
    AND t1.location = t2.location
WHERE (t1.industry IS NULL OR t1.industry = '')
AND t2.industry IS NOT NULL;
```
now we need to populate thouse null if possible
```sql
UPDATE Layoffs_staging2 t1
JOIN layoffs_staging2 t2
	ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE t1.industry IS NULL 
AND t2.industry IS NOT NULL;
```
3. Look at Null Values
   
the null values in total_laid_off, percentage_laid_off and fund_raised_millions all look normal.
so there isn't anything i want to change with the null values

5. Remove any columns and rows we need to
```sql
SELECT * 
FROM layoffs_staging2
WHERE total_laid_off IS NULL 
AND percentage_laid_off IS NULL;
```
```sql
Delete useless data we can't really use
DELETE
FROM layoffs_staging2
WHERE total_laid_off IS NULL 
AND percentage_laid_off IS NULL;
```
delete row_num column cause we no longer need it
```sql
ALTER TABLE layoffs_staging2
DROP COLUMN row_num;
```

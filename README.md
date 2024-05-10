
# World-Layoffs Project

**-- Data Cleaning --**

select *
from layoffs;

-- Creating a copy table from the original to work and clean the data--
-- We want a table with the raw data in case something happens --

create table layoffs_staging
like layoffs;

select *
from layoffs_staging;

insert layoffs_staging
select *
from layoffs;

-- Now when we are cleaning data we usually follow a few steps
-- 1. Check for duplicates and remove any
-- 2. Standardize data and fix errors
-- 3. Look at null and blank values
-- 4. Remove any columns and rows that are not necessary


**-- 1. Remove Duplicates**

# First let's check for duplicates

select *
from layoffs_staging;

select *,
row_number() over(partition by company, location, industry, total_laid_off, 
percentage_laid_off, `date`, stage, country, funds_raised_millions) as row_num
from layoffs_staging;

with duplicate_cte as
(
	select *,
	row_number() over(partition by company, location, industry, total_laid_off, 
	percentage_laid_off, `date`, stage, country, funds_raised_millions) as row_num
	from layoffs_staging
)
select *
from duplicate_cte
where row_num > 1; 

select *
from layoffs_staging
where company = 'Casper';


-- Just DELETE is not working, have to think it through, lets see --
 
with duplicate_cte as
(
	select *,
	row_number() over(partition by company, location, industry, total_laid_off, 
	percentage_laid_off, `date`, stage, country, funds_raised_millions) as row_num
	from layoffs_staging
)
delete
from duplicate_cte
where row_num > 1; 



-- one solution, which I think is a good one. Is to create a new column and add those row numbers
-- in. Then delete where row numbers are over 2, then delete that column


CREATE TABLE `layoffs_staging2` (
  `company` text,
  `location` text,
  `industry` text,
  `total_laid_off` int DEFAULT NULL,
  `percentage_laid_off` text,
  `date` text,
  `stage` text,
  `country` text,
  `funds_raised_millions` int DEFAULT NULL,
  `row_num` INT -- new column added
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;


select *
from layoffs_staging2;

insert into layoffs_staging2
select *,
row_number() over(partition by company, location, industry, total_laid_off, 
percentage_laid_off, `date`, stage, country, funds_raised_millions) as row_num
from layoffs_staging;

-- these are the ones we want to delete where the row number is > 1 or 2or greater essentially

select *
from layoffs_staging2
where row_num > 1;

DELETE
from layoffs_staging2
where row_num > 1;

select *
from layoffs_staging2;

**-- 2. Standardize Data (finding issues and fixing them)**

select company, trim(company)
from layoffs_staging2;

update layoffs_staging2
set company = trim(company);

-- if we look at industry it looks like we have some null and empty rows, 
-- let's take a look at these

select distinct industry
from layoffs_staging2
order by 1;

-- I noticed the Crypto has different variations. 
-- We need to standardize that - let's say all to Crypto

select *
from layoffs_staging2
where industry like 'crypto%';

update layoffs_staging2
set industry = 'Crypto'
where industry like 'crypto%';

select distinct location
from layoffs_staging2
order by 1;

select distinct country
from layoffs_staging2
order by 1;

select distinct country , trim(trailing '.' from country)
from layoffs_staging2
order by 1;

update layoffs_staging2
set country = trim(trailing '.' from country)
where country like 'united states%';

-- lets fix date column as it was in text format initially 
-- we can use str_to_date to update date field

select `date`,
str_to_date(`date`, '%m/%d/%Y')
from layoffs_staging2;

update layoffs_staging2
set `date` = str_to_date(`date`, '%m/%d/%Y');

select `date`
from layoffs_staging2;

-- now we can convert the data type properly

alter table layoffs_staging2
modify column `date` date;


select *
from layoffs_staging2;

**-- 3. Look at Null Values**

select *
from layoffs_staging2
where total_laid_off is null
and percentage_laid_off is null;

select distinct industry
from layoffs_staging2;

update layoffs_staging2
set industry = null
where industry = '';


select *
from layoffs_staging2
where company = 'airbnb';

-- it looks like airbnb is a travel, but this one just isn't populated.
-- I'm sure it's the same for the others. What we can do is
-- write a query that if there is another row with the same company name, it will update it to the non-null industry values
-- makes it easy so if there were thousands we wouldn't have to manually check them all

-- we should set the blanks to nulls since those are typically easier to work with

select *
from layoffs_staging2
where industry is null
or industry = '';

-- now we need to populate those nulls if possible

select t1.industry, t2.industry
from layoffs_staging2 t1
join layoffs_staging2 t2
	on t1.company = t2.company
where (t1.industry is null or t1.industry = '')
and t2.industry is not null;

update layoffs_staging2 t1
join layoffs_staging2 t2
	on t1.company = t2.company
set t1.industry = t2.industry
where t1.industry is null
and t2.industry is not null;

select *
from layoffs_staging2;

**-- 4. remove any columns and rows we need to**

select *
from layoffs_staging2
where total_laid_off is null
and percentage_laid_off is null;

delete
from layoffs_staging2
where total_laid_off is null
and percentage_laid_off is null;

select *
from layoffs_staging2;

alter table layoffs_staging2
drop column row_num;

select *
from layoffs_staging2;


=============================================================================================================================================================================


**-- Exploratory Data Analysis (EDA) -- **

-- Here we are going to explore the data and find trends or patterns or anything interesting like outliers

Select *
from layoffs_staging2;

Select MAX(total_laid_off)
from layoffs_staging2;

Select MAX(total_laid_off), max(percentage_laid_off)
from layoffs_staging2
where percentage_laid_off =1;


-- Looking at Percentage to see how big these layoffs were

SELECT MAX(percentage_laid_off),  MIN(percentage_laid_off)
FROM layoffs_staging2
WHERE  percentage_laid_off IS NOT NULL;

-- Which companies had 1 which is basically 100 percent of they company laid off

Select *
from layoffs_staging2
where percentage_laid_off =1
order by total_laid_off desc;

-- these are mostly startups it looks like who all went out of business during this time

-- Companies with the most Total Layoffs

Select Company, sum(total_laid_off)
from layoffs_staging2
group by company
order by 2 desc;

Select min(`date`), max(`date`)
from layoffs_staging2;

Select industry, sum(total_laid_off)
from layoffs_staging2
group by industry
order by 2 desc;

-- Country with the most Total Layoffs

Select country, sum(total_laid_off)
from layoffs_staging2
group by country
order by 2 desc;

select year(`date`), sum(total_laid_off)
from layoffs_staging2
group by year(`date`)
order by 1 desc;

-- Total layoffs as per the months

select substring(`date`, 1, 7) as `month`, sum(total_laid_off)
from layoffs_staging2
where substring(`date`, 1, 7) is not null
group by `month`
order by 1 asc;

-- Rolling Total of Layoffs Per Month with CTE

with Rolling_Total as
(
select substring(`date`, 1, 7) as `month`, sum(total_laid_off) as total_off
from layoffs_staging2
where substring(`date`, 1, 7) is not null
group by `month`
order by 1 asc
)
select `month`, total_off, sum(total_off) over(order by `month`) as rolling_total
from Rolling_Total;


Select Company, year(`date`), sum(total_laid_off)
from layoffs_staging2
group by company, year(`date`)
order by 3 desc;

with company_year(company, years, total_laid_off) as 
(
Select Company, year(`date`), sum(total_laid_off)
from layoffs_staging2
group by company, year(`date`)
), Company_Year_Rank as
(select *, dense_rank() over(partition by years order by total_laid_off desc) as ranking
from company_year
where years is not null
)
select *
from Company_Year_Rank
where ranking <= 5;


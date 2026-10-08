# Final Project 
### COVID-19 Data Warehouse in PostgreSQL (Our World in Data)

Name: Anas Alsawalhi 

Operating system: windows 

Course: Databases for Analytics – Module 7

---

## 1. Initial Data Source
| Item | Detail |
|---|---|
| Dataset | Our World in Data – COVID-19 |
| File | `owid-covid-data.csv` |
| Source | https://github.com/owid/covid-19-data (folder `public/data`) |
| Access | Public, open source, no login needed |

I found the data by searching GitHub for large open datasets. I chose it because it is trusted, well documented (it has a codebook), and complex enough: ~60+ columns, many countries, daily time series, lots of NULLs.

## 2. Format of the Data
 
-**Rows** 429,435 (`SELECT COUNT(*) FROM covid_raw;`)
  
-**Columns** 67 (`SELECT COUNT(*) FROM information_schema.columns WHERE table_name = 'covid_raw';`)

## 3. Data Dictionary

The raw table `covid_raw` has 67 columns, all imported as text (`character varying`). I kept the columns needed for the analysis and converted them to proper types in 3 final tables.

### countries (one row per country)
| Column | Type | Description |
|---|---|---|
| iso_code | VARCHAR | ISO 3166-1 alpha-3 country code (primary key) |
| location | VARCHAR | Country name |
| continent | VARCHAR | Continent of the country |
| population | BIGINT | Population of the country |
| median_age | NUMERIC | Median age of the population |
| gdp_per_capita | NUMERIC | Gross domestic product per person |

### daily_stats (one row per country per day)
| Column | Type | Description |
|---|---|---|
| iso_code | VARCHAR | Country code (foreign key to countries) |
| date | DATE | Date of observation |
| total_cases | NUMERIC | Cumulative confirmed cases |
| new_cases | NUMERIC | New confirmed cases that day |
| total_deaths | NUMERIC | Cumulative deaths |
| new_deaths | NUMERIC | New deaths that day |

### daily_vaccinations (one row per country per day with vaccination data)
| Column | Type | Description |
|---|---|---|
| iso_code | VARCHAR | Country code (foreign key to countries) |
| date | DATE | Date of observation |
| total_vaccinations | NUMERIC | Total vaccine doses administered |
| people_vaccinated | NUMERIC | People with at least one dose |

## 4. Obstacles and How I Solved Them

1. **Large file.** The file has 429,435 rows and 67 columns, too big to inspect comfortably in Excel or a text editor.
   *Solution:* I loaded it into PostgreSQL first and explored it with SQL queries.

2. **Everything imported as text.** Every column in the raw table `covid_raw` was imported as `character varying`, so dates and numbers could not be used in calculations.
   *Solution:* I converted the types while creating the final tables, using `NULLIF(col::text,'')::date` and `::numeric` so empty strings became NULL.

3. **Aggregate rows mixed with countries.** Rows such as World, continents and income groups (`iso_code` starting with `OWID_`) would cause double counting.
   *Solution:* I excluded them with `WHERE iso_code NOT LIKE 'OWID%'`.

4. **Duplicate rows blocked the primary key.** Creating the primary key on `daily_stats` failed with `duplicate key (FRO, 2021-09-16)` because the raw data had duplicate country-date rows.
   *Solution:* I rebuilt the table with `SELECT DISTINCT ON (iso_code, date)` so each country has one row per day.

5. **Failed script removed the table.** After the error, `daily_stats` did not exist at all, because pgAdmin rolled back the whole script when one statement failed.
   *Solution:* I ran each statement separately (CREATE TABLE, then PRIMARY KEY, then FOREIGN KEY) and checked the result after each one.

6. **Repeated country information.** Population, median age and GDP were repeated on every daily row.
   *Solution:* I normalized the data into 3 tables (`countries`, `daily_stats`, `daily_vaccinations`) linked by primary and foreign keys.

7. **Sparse vaccination data.** Vaccination columns are mostly NULL for many dates.
   *Solution:* I moved them to their own table and kept only rows where `total_vaccinations` is not NULL (66,535 rows).

## 5. Table Structure

### Staging table
All 67 columns were imported as text (`character varying`) into `covid_raw`, then cleaned while building the final tables.

### Final tables

```sql
CREATE TABLE countries AS
SELECT DISTINCT ON (iso_code)
       iso_code, location, continent,
       NULLIF(population::text,'')::numeric::bigint AS population,
       NULLIF(median_age::text,'')::numeric         AS median_age,
       NULLIF(gdp_per_capita::text,'')::numeric     AS gdp_per_capita
FROM covid_raw
WHERE iso_code NOT LIKE 'OWID%'
ORDER BY iso_code, NULLIF(date::text,'')::date DESC;
ALTER TABLE countries ADD PRIMARY KEY (iso_code);

CREATE TABLE daily_stats AS
SELECT DISTINCT ON (iso_code, d)
       iso_code, d AS date,
       total_cases, new_cases, total_deaths, new_deaths
FROM (
  SELECT iso_code,
         NULLIF(date::text,'')::date            AS d,
         NULLIF(total_cases::text,'')::numeric  AS total_cases,
         NULLIF(new_cases::text,'')::numeric    AS new_cases,
         NULLIF(total_deaths::text,'')::numeric AS total_deaths,
         NULLIF(new_deaths::text,'')::numeric   AS new_deaths
  FROM covid_raw
  WHERE iso_code NOT LIKE 'OWID%'
) t
ORDER BY iso_code, d;
ALTER TABLE daily_stats ADD PRIMARY KEY (iso_code, date);
ALTER TABLE daily_stats ADD FOREIGN KEY (iso_code) REFERENCES countries(iso_code);

CREATE TABLE daily_vaccinations AS
SELECT DISTINCT ON (iso_code, d)
       iso_code, d AS date,
       total_vaccinations, people_vaccinated
FROM (
  SELECT iso_code,
         NULLIF(date::text,'')::date                  AS d,
         NULLIF(total_vaccinations::text,'')::numeric AS total_vaccinations,
         NULLIF(people_vaccinated::text,'')::numeric  AS people_vaccinated
  FROM covid_raw
  WHERE iso_code NOT LIKE 'OWID%'
    AND NULLIF(total_vaccinations::text,'') IS NOT NULL
) t
ORDER BY iso_code, d;
ALTER TABLE daily_vaccinations ADD PRIMARY KEY (iso_code, date);
ALTER TABLE daily_vaccinations ADD FOREIGN KEY (iso_code) REFERENCES countries(iso_code);
```

### Schema
```
countries (iso_code PK, location, continent, population, median_age, gdp_per_capita)
   │1
   ├──< daily_stats (iso_code FK, date) PK(iso_code, date)
   └──< daily_vaccinations (iso_code FK, date) PK(iso_code, date)
```

### Data types check
```sql
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_name IN ('countries','daily_stats','daily_vaccinations')
ORDER BY table_name, ordinal_position;
```
<img width="742" height="791" alt="image" src="https://github.com/user-attachments/assets/cd19512c-0a96-4ba7-b195-bf35ed8aa172" />


### Requirements check
| Requirement | Met by |
|---|---|
| At least 3 tables | countries, daily_stats, daily_vaccinations |
| One table ≥ 1000 rows | daily_stats (393,903 rows) |
| Two tables ≥ 100 rows | countries (237), daily_vaccinations (66,535) |
| Date type | `date` |
| Numeric type | `new_cases`, `total_deaths`, `population`... |
| String type | `location`, `continent`, `iso_code` |

```sql
SELECT 'countries' t, COUNT(*) FROM countries
UNION ALL SELECT 'daily_stats', COUNT(*) FROM daily_stats
UNION ALL SELECT 'daily_vaccinations', COUNT(*) FROM daily_vaccinations;
```
 <img width="747" height="549" alt="image" src="https://github.com/user-attachments/assets/9234f78a-ed6e-4375-8e81-138aebfc91b0" />



## 6. SELECT * From Each Table
```sql
SELECT * FROM countries LIMIT 10;
SELECT * FROM daily_stats LIMIT 10;
SELECT * FROM daily_vaccinations LIMIT 10;
```
<img width="752" height="651" alt="image" src="https://github.com/user-attachments/assets/c50e4536-6064-4276-83fa-b01f5f7093f6" />


## 7. Interesting Queries

### 7.1 JOIN – Top 10 countries by total deaths
```sql
SELECT c.location, c.continent, MAX(d.total_deaths) AS total_deaths
FROM daily_stats d
JOIN countries c ON c.iso_code = d.iso_code
GROUP BY c.location, c.continent
ORDER BY total_deaths DESC NULLS LAST
LIMIT 10;
```
 <img width="741" height="647" alt="image" src="https://github.com/user-attachments/assets/64355c9b-dd78-47d5-95ae-181f953898f1" />


### 7.2 GROUP BY + aggregate – Cases, deaths and death rate by continent
```sql
SELECT c.continent,
       SUM(d.new_cases)  AS total_cases,
       SUM(d.new_deaths) AS total_deaths,
       ROUND(100.0 * SUM(d.new_deaths) / NULLIF(SUM(d.new_cases), 0), 2) AS death_rate_pct
FROM daily_stats d
JOIN countries c ON c.iso_code = d.iso_code
WHERE c.continent IS NOT NULL
GROUP BY c.continent
ORDER BY total_cases DESC;
```
 <img width="740" height="561" alt="image" src="https://github.com/user-attachments/assets/97fcb18c-cef4-4ce3-9205-4838840dab9e" />


### 7.3 Three-table JOIN – Vaccination vs. death rate per country
```sql
SELECT c.location,
       MAX(v.people_vaccinated) AS people_vaccinated,
       ROUND(100.0 * MAX(v.people_vaccinated) / c.population, 1) AS pct_vaccinated,
       MAX(d.total_deaths) AS total_deaths
FROM countries c
JOIN daily_vaccinations v ON v.iso_code = c.iso_code
JOIN daily_stats d        ON d.iso_code = c.iso_code
WHERE c.population > 10000000
GROUP BY c.location, c.population
ORDER BY pct_vaccinated DESC
LIMIT 15;
```
 <img width="740" height="756" alt="image" src="https://github.com/user-attachments/assets/0848a938-a747-49d4-803c-a348e01f259b" />


### 7.4 Monthly trend of new cases worldwide
```sql
SELECT DATE_TRUNC('month', date)::date AS month, SUM(new_cases) AS cases
FROM daily_stats
GROUP BY 1
ORDER BY 1;
```
 <img width="756" height="817" alt="image" src="https://github.com/user-attachments/assets/90174343-797e-4b64-985a-10561f445de9" />


### 7.5 Cases per million by median age group
```sql
SELECT CASE WHEN c.median_age < 25 THEN 'Young (<25)'
            WHEN c.median_age < 35 THEN 'Middle (25-35)'
            ELSE 'Older (35+)' END AS age_group,
       COUNT(*) AS countries,
       ROUND(AVG(t.max_cases / (c.population / 1000000.0))) AS avg_cases_per_million
FROM countries c
JOIN (SELECT iso_code, MAX(total_cases) AS max_cases
      FROM daily_stats GROUP BY iso_code) t USING (iso_code)
WHERE c.median_age IS NOT NULL
GROUP BY age_group
ORDER BY avg_cases_per_million DESC;
```
 <img width="744" height="674" alt="image" src="https://github.com/user-attachments/assets/fcf93d3e-5803-476e-a9f8-a7429b27ae2b" />


## 8. Verifying the Data
To validate the import I (1) compared row counts with the CSV, (2) built a view, and (3) cross-checked one country against the original file.

```sql
CREATE VIEW country_summary AS
SELECT c.location, c.continent, c.population,
       MAX(d.total_cases)  AS total_cases,
       MAX(d.total_deaths) AS total_deaths
FROM countries c
JOIN daily_stats d USING (iso_code)
GROUP BY c.location, c.continent, c.population;

SELECT * FROM country_summary WHERE location = 'Jordan'

```
 <img width="612" height="637" alt="image" src="https://github.com/user-attachments/assets/1928da37-c78e-48a1-8c5a-537d34b9319c" />

``` I compared the result with the last row for Jordan in the original CSV and the numbers matched, which verifies the data.
```
Other sanity checks:

```sqlSELECT MIN(date), MAX(date) FROM daily_stats;                       -- date range is sensible

SELECT COUNT(*) FROM daily_stats WHERE new_cases < 0;               -- negative values (data corrections)
SELECT COUNT(DISTINCT iso_code) FROM countries;                     -- number of countries
```

## 9. Insights

### About the data
- The raw file has 429,435 rows and 67 columns. After removing aggregate rows (`OWID_*`) and duplicates, the final tables hold **237 countries**, **393,903 daily records** and **66,535 vaccination records**.
- Vaccination data is much sparser than case data (66,535 rows versus 393,903) and starts in 2021 (Afghanistan's first record is in February 2021). This is why it lives in its own table.
- The data covers 56 months, from January 2020 to August 2024.

### Deaths by country (7.1)
- The United States has the highest total deaths (1,193,165), followed by Brazil (702,116), India (533,623) and Russia (403,188).
- Five of the top 10 countries are in Europe (Russia, United Kingdom, Italy, Germany, France).
- These are cumulative totals and depend heavily on population size, so they are not a fair comparison between countries.

### By continent (7.2)
- Asia had the most cases (301.6 million), while Europe had the most deaths (2.10 million).
- The death rate (deaths divided by cases) ranges from 0.22% in Oceania and 0.54% in Asia up to 1.97% in both South America and Africa. Differences in testing and reporting probably explain part of this gap, because fewer tests mean fewer confirmed cases and a higher apparent rate.

### Vaccination versus deaths (7.3)
- Among countries with more than 10 million people, Cuba (96.4%), Portugal (95.6%), Chile (92.3%), Vietnam (92.2%) and China (91.9%) have the highest share of people with at least one dose.
- High vaccination does not line up with low total deaths: Peru (89.8% vaccinated) has 220,975 deaths and Brazil (88.1%) has 702,116, while Cuba has 8,530. Deaths are cumulative over the whole period, including the time before vaccines existed, and they depend on country size, so this shows a pattern at most, not cause and effect.
- Taiwan has NULL for total deaths. I kept this gap as NULL instead of replacing it with zero.

### Trend over time (7.4)
- Monthly new cases start at only 2,033 in January 2020, reach about 2 million by April 2020 and 10.2 million by October 2020.
- The highest month in the whole dataset is **January 2022 with 94,645,572 new cases**, far above every earlier month (the previous peak was 23.7 million in May 2021). This is consistent with the Omicron wave, although the data itself does not name the variant.
- A second, smaller peak appears in December 2022 (67.1 million).
- From 2023 the numbers fall sharply, to about 0.9 million in June 2023 and 47,169 in August 2024, the last month in the data. This drop probably reflects less testing and reporting, not only fewer infections.

### Age (7.5)
- Countries with a median age of 35 or more average 357,684 cases per million, versus 130,564 for the 25 to 35 group and 25,836 for countries under 25. That is about 14 times higher for the older group than for the youngest.
- This is correlation only. Older countries are often richer and test more, so they confirm more cases, which probably inflates the gap.

### Limitations
- Countries report differently, so cross-country comparisons need caution.
- Missing values were kept as NULL, not turned into zero.
- `MAX(total_*)` takes each country's last reported value, and reporting stops at different dates.

### What I learned
Most of the work in a real dataset is cleaning and typing the data. Splitting one 67-column table into three related tables with keys made the queries simpler and made the data problems (text types, duplicates) visible.

## 10. Process Summary
1. Found the dataset on GitHub (OWID) and forked it.
2. Created the database and staging table `covid_raw`.
3. Loaded the CSV with `COPY ... HEADER true` and handled NULLs.
4. Removed aggregate rows and split the data into 3 related tables with PK/FK.
5. Verified the data using counts, a view, and a cross-check with the source file.
6. Ran join and aggregate queries and summarized the insights.

---








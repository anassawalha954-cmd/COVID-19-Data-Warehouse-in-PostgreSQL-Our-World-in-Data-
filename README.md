# Final Project – Show Us Your Data!
### COVID-19 Data Warehouse in PostgreSQL (Our World in Data)

Name: Anas Alsawalhi 

Operating system: Windows · PostgreSQL <version> · pgAdmin 4

Course: Databases for Analytics – Module 7

---

## 1. Initial Data Source
| Item | Detail |
|---|---|
| Dataset | Our World in Data – COVID-19 |
| File | `owid-covid-data.csv` |
| Source | https://github.com/owid/covid-19-data (folder `public/data`) |
| My fork | <link to your fork> |
| Access | Public, open source, no login needed |

I found the data by searching GitHub for large open datasets. I chose it because it is trusted, well documented (it has a codebook), and complex enough: ~60+ columns, many countries, daily time series, lots of NULLs.

## 2. Format of the Data
- **Format:** CSV, comma-delimited, UTF-8, with a header row
- **Rows:** <N>  → `SELECT COUNT(*) FROM covid_raw;`
- **Columns:** <N> → `SELECT COUNT(*) FROM information_schema.columns WHERE table_name = 'covid_raw';`

## 3. Data Dictionary
| Column | Type | Description |
|---|---|---|
| iso_code | VARCHAR(10) | ISO 3166-1 alpha-3 country code (`OWID_...` = aggregate region) |
| continent | VARCHAR(50) | Continent of the country |
| location | VARCHAR(100) | Country or region name |
| date | DATE | Date of observation |
| total_cases | NUMERIC | Cumulative confirmed cases |
| new_cases | NUMERIC | New confirmed cases that day |
| total_deaths | NUMERIC | Cumulative deaths |
| new_deaths | NUMERIC | New deaths that day |
| total_vaccinations | NUMERIC | Total vaccine doses administered |
| people_vaccinated | NUMERIC | People with at least one dose |
| population | BIGINT | Population of the country |
| median_age | NUMERIC | Median age of the population |
| gdp_per_capita | NUMERIC | GDP per capita |
| *(remaining columns)* | | Add from `owid-covid-codebook.csv` |

## 4. Obstacles and How I Solved Them
| # | Obstacle | Solution |
|---|---|---|
| 1 | File is huge – hard to open in Excel/editors | Inspected it through PostgreSQL itself instead of an editor |
| 2 | ~60 columns – writing the CREATE TABLE by hand is error-prone | Built a raw staging table `covid_raw` first, using the codebook for types |
| 3 | Header row can break numeric columns (the lecture's TSV example had to delete it manually) | File is CSV, so I used `HEADER true` in `COPY` |
| 4 | Empty cells in numeric columns | `NULL ''` option and `NUMERIC` types instead of `INT` |
| 5 | `COPY` permission denied for file paths | Used pgAdmin Import/Export or `\copy` |
| 6 | Aggregates (World, Europe, income groups) mixed with real countries | Excluded `iso_code LIKE 'OWID%'` |
| 7 | Country info repeated on every daily row | Normalized into 3 tables with PK/FK |

## 5. Table Structure

### Staging
```sql
CREATE TABLE covid_raw (
    iso_code VARCHAR(10),
    continent VARCHAR(50),
    location VARCHAR(100),
    date DATE,
    total_cases NUMERIC,
    new_cases NUMERIC,
    total_deaths NUMERIC,
    new_deaths NUMERIC,
    total_vaccinations NUMERIC,
    people_vaccinated NUMERIC,
    population BIGINT,
    median_age NUMERIC,
    gdp_per_capita NUMERIC
    -- add the remaining columns here
);

COPY covid_raw
FROM 'C:/temp/owid-covid-data.csv'
WITH (FORMAT csv, HEADER true, NULL '');
```
> If your CSV has more columns, `COPY covid_raw (col1, col2, ...)` must list them in file order, or the table must have all columns.

### Final normalized tables
```sql
CREATE TABLE countries AS
SELECT DISTINCT ON (iso_code)
       iso_code, location, continent, population, median_age, gdp_per_capita
FROM covid_raw
WHERE iso_code NOT LIKE 'OWID%'
ORDER BY iso_code, date DESC;
ALTER TABLE countries ADD PRIMARY KEY (iso_code);

CREATE TABLE daily_stats AS
SELECT iso_code, date, total_cases, new_cases, total_deaths, new_deaths
FROM covid_raw
WHERE iso_code NOT LIKE 'OWID%';
ALTER TABLE daily_stats ADD PRIMARY KEY (iso_code, date);
ALTER TABLE daily_stats ADD FOREIGN KEY (iso_code) REFERENCES countries(iso_code);

CREATE TABLE daily_vaccinations AS
SELECT iso_code, date, total_vaccinations, people_vaccinated
FROM covid_raw
WHERE iso_code NOT LIKE 'OWID%'
  AND total_vaccinations IS NOT NULL;
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
📸 *Screenshot: `screenshots/structure.png`*

### Requirements check
| Requirement | Met by |
|---|---|
| At least 3 tables | countries, daily_stats, daily_vaccinations |
| One table ≥ 1000 rows | daily_stats (<N> rows) |
| Two tables ≥ 100 rows | countries (<N>), daily_vaccinations (<N>) |
| Date type | `date` |
| Numeric type | `new_cases`, `total_deaths`, `population`... |
| String type | `location`, `continent`, `iso_code` |

```sql
SELECT 'countries' t, COUNT(*) FROM countries
UNION ALL SELECT 'daily_stats', COUNT(*) FROM daily_stats
UNION ALL SELECT 'daily_vaccinations', COUNT(*) FROM daily_vaccinations;
```
📸 *Screenshot: <img width="747" height="549" alt="image" src="https://github.com/user-attachments/assets/9234f78a-ed6e-4375-8e81-138aebfc91b0" />



## 6. SELECT * From Each Table
```sql
SELECT * FROM countries LIMIT 10;
SELECT * FROM daily_stats LIMIT 10;
SELECT * FROM daily_vaccinations LIMIT 10;
```
📸 `screenshot <img width="752" height="651" alt="image" src="https://github.com/user-attachments/assets/c50e4536-6064-4276-83fa-b01f5f7093f6" />


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
📸 `screenshot <img width="741" height="647" alt="image" src="https://github.com/user-attachments/assets/64355c9b-dd78-47d5-95ae-181f953898f1" />


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
📸 `screenshot <img width="740" height="561" alt="image" src="https://github.com/user-attachments/assets/97fcb18c-cef4-4ce3-9205-4838840dab9e" />


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
📸 `screenshot <img width="740" height="756" alt="image" src="https://github.com/user-attachments/assets/0848a938-a747-49d4-803c-a348e01f259b" />


### 7.4 Monthly trend of new cases worldwide
```sql
SELECT DATE_TRUNC('month', date)::date AS month, SUM(new_cases) AS cases
FROM daily_stats
GROUP BY 1
ORDER BY 1;
```
📸 `screenshot <img width="756" height="817" alt="image" src="https://github.com/user-attachments/assets/90174343-797e-4b64-985a-10561f445de9" />


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
📸 `screenshot <img width="744" height="674" alt="image" src="https://github.com/user-attachments/assets/fcf93d3e-5803-476e-a9f8-a7429b27ae2b" />


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
📸 `screenshot <img width="612" height="637" alt="image" src="https://github.com/user-attachments/assets/1928da37-c78e-48a1-8c5a-537d34b9319c" />
```I compared the result with the last row for Jordan in the original CSV and the numbers matched, which verifies the data.
```
Other sanity checks:
```sql
SELECT MIN(date), MAX(date) FROM daily_stats;                       -- date range is sensible
SELECT COUNT(*) FROM daily_stats WHERE new_cases < 0;               -- negative values (data corrections)
SELECT COUNT(DISTINCT iso_code) FROM countries;                     -- number of countries
```

## 9. Insights
*(Replace with what your actual results show.)*
- **Death rate differs by continent**, which likely reflects testing capacity and reporting quality, not only severity.
- **Case waves** in 7.4 line up with major variants (Delta, Omicron).
- **Older populations** (7.5) tended to show higher cases per million, but this is correlation, not proof.
- **Vaccination data is sparse**: many countries have NULLs, which is why it lives in its own table.
- **Limits:** countries report differently, so cross-country comparisons need caution.

## 10. Process Summary
1. Found the dataset on GitHub (OWID) and forked it.
2. Created the database and staging table `covid_raw`.
3. Loaded the CSV with `COPY ... HEADER true` and handled NULLs.
4. Removed aggregate rows and split the data into 3 related tables with PK/FK.
5. Verified the data using counts, a view, and a cross-check with the source file.
6. Ran join and aggregate queries and summarized the insights.

---








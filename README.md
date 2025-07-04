# Arrest Data Analysis (2010–2019)

## Overview

This repository contains a detailed SQL analysis of arrest data from 2010 to 2019, featuring both basic and advanced queries to explore arrest patterns, trends, and insights across time, geography, demographics, and charge types.

---

## Dataset Description

The dataset contains:

- `report_id`: Unique report identifier  
- `report_type`: Booking or RFC  
- `arrest_date`: Date of arrest  
- `arrest_time`: Time of arrest  
- `area_id`, `area_name`: Geographic location  
- `age`: Age of individual arrested  
- `sex_code`: Gender code ('M' or 'F')  
- `charge_group_description`: Broad charge category  
- `arrest_type_code`: Arrest type (e.g., felony complaint filed)  
- `charge_description`: Detailed charge description  
- `address`: Location of arrest  
- `booking_date`, `booking_time`, `booking_location`: Booking details  

---

## Analysis Questions and Solutions

### Basic Questions

1. How many total arrests were made between 2010 and 2019?
```sql
SELECT 
COUNT(*) AS total_arrests
FROM ArrestData
WHERE YEAR(arrest_date) BETWEEN 2010 AND 2019
```
2. How many arrests were made each year?
```sql
SELECT YEAR(arrest_date) AS arrest_year, COUNT(*) AS total_arrests
FROM ArrestData
WHERE YEAR(arrest_date) BETWEEN 2010 AND 2019
GROUP BY YEAR(arrest_date)
ORDER BY arrest_year;
```
3. What are the top 10 most common charges?
```sql
SELECT TOP 10
charge_group_description,
COUNT(charge_group_description) AS total_arrests
FROM ArrestData
GROUP BY charge_group_description
ORDER BY total_arrests DESC
```
4. How many arrests were made for each gender?
```sql
SELECT TOP 5
area_name,
COUNT(*) AS total
FROM ArrestData
GROUP BY area_name
```
5. How many arrests were made per arrest_type_code?
```sql
SELECT 
arrest_type_code,
COUNT(*) AS total_arrests
FROM ArrestData
WHERE arrest_type_code IS NOT NULL
GROUP BY arrest_type_code
ORDER BY total_arrests DESC
```
6. What is the distribution of arrests by charge_group_description?
```sql
SELECT 
charge_group_description,
COUNT(*) AS total_arrests
FROM ArrestData
GROUP BY charge_group_description
ORDER BY total_arrests DESC
```
7. Which day of the week sees the most arrests?
```sql
SELECT DATENAME(WEEKDAY, arrest_date) AS day_of_week,
       COUNT(*) AS total_arrests
FROM ArrestData
WHERE arrest_date IS NOT NULL
GROUP BY DATENAME(WEEKDAY, arrest_date)
ORDER BY total_arrests DESC;
```
8. What hour of the day has the highest number of arrests?
```sql
SELECT DATEPART(HOUR, arrest_time) AS arrest_hour,
       COUNT(*) AS total_arrests
FROM ArrestData
WHERE arrest_time IS NOT NULL
GROUP BY DATEPART(HOUR, arrest_time)
ORDER BY total_arrests DESC;
```
9. What is the average age of arrested individuals?
```sql
SELECT
sex_code,
AVG(age) AS average_age
FROM ArrestData
WHERE age IS NOT NULL
GROUP BY sex_code
```
10. Which 5 areas had the highest number of arrests?
```sql
SELECT TOP 5
area_name,
COUNT(*) AS total
FROM ArrestData
GROUP BY area_name
```
### Advanced Questions

1. Rank the top 3 charges per area using RANK() or DENSE_RANK()
```sql
WITH RankedCharges AS (
  SELECT 
    area_name,
    charge_group_description,
    COUNT(*) AS arrest_count,
    RANK() OVER (
      PARTITION BY area_name 
      ORDER BY COUNT(*) DESC
    ) AS charge_rank
  FROM ArrestData
  WHERE area_name IS NOT NULL AND charge_group_description IS NOT NULL
  GROUP BY area_name, charge_group_description
)
SELECT *
FROM RankedCharges
WHERE charge_rank <= 3
ORDER BY area_name, charge_rank;
```
2. What is the year-over-year percentage change in arrests?
```sql
WITH YearlyArrests AS (
  SELECT 
    YEAR(arrest_date) AS arrest_year,
    COUNT(*) AS total_arrests
  FROM ArrestData
  WHERE arrest_date IS NOT NULL
  GROUP BY YEAR(arrest_date)
)
SELECT 
  arrest_year,
  total_arrests,
  LAG(total_arrests) OVER (ORDER BY arrest_year) AS previous_year_arrests,
  ROUND(
    100.0 * (total_arrests - LAG(total_arrests) OVER (ORDER BY arrest_year)) 
    / NULLIF(LAG(total_arrests) OVER (ORDER BY arrest_year), 0),
    2
  ) AS percent_change
FROM YearlyArrests
ORDER BY arrest_year;
```
3. For each charge type, what is the average time gap between arrest and booking?
```sql
SELECT 
  charge_group_description,
  AVG(DATEDIFF(MINUTE, 
      CAST(arrest_date AS DATETIME) + CAST(arrest_time AS DATETIME), 
      CAST(booking_date AS DATETIME) + CAST(booking_time AS DATETIME)
  ) / 60.0) AS avg_delay_hours
FROM ArrestData
WHERE 
  arrest_date IS NOT NULL AND 
  arrest_time IS NOT NULL AND
  booking_date IS NOT NULL AND 
  booking_time IS NOT NULL AND
  charge_group_description IS NOT NULL
GROUP BY charge_group_description
ORDER BY avg_delay_hours DESC;
```
4. What is the rolling 12-month arrest total using window functions?
```sql
WITH MonthlyArrests AS (
  SELECT 
    CAST(FORMAT(arrest_date, 'yyyy-MM-01') AS DATE) AS month_start,
    COUNT(*) AS monthly_arrests
  FROM ArrestData
  WHERE arrest_date IS NOT NULL
  GROUP BY CAST(FORMAT(arrest_date, 'yyyy-MM-01') AS DATE)
)
SELECT 
  month_start,
  monthly_arrests,
  SUM(monthly_arrests) OVER (
    ORDER BY month_start 
    ROWS BETWEEN 11 PRECEDING AND CURRENT ROW
  ) AS rolling_12_month_total
FROM MonthlyArrests
ORDER BY month_start;
```
5. Create a view showing arrest trends per year and charge group?
```sql
CREATE VIEW vw_arrest_trends_by_year_and_charge_group AS
SELECT 
  YEAR(arrest_date) AS arrest_year,
  charge_group_description,
  COUNT(*) AS total_arrests
FROM ArrestData
WHERE 
  arrest_date IS NOT NULL AND 
  charge_group_description IS NOT NULL
GROUP BY 
  YEAR(arrest_date), 
  charge_group_description;

SELECT * FROM vw_arrest_trends_by_year_and_charge_group
```
6. Write a stored procedure that returns the top charges for a given area and year?
```sql
CREATE PROCEDURE sp_top_charges_by_area_and_year
  @area_name NVARCHAR(100),
  @year INT,
  @top_n INT = 5
AS
BEGIN
  SELECT TOP (@top_n)
    charge_description,
    COUNT(*) AS arrest_count
  FROM ArrestData
  WHERE 
    area_name = @area_name AND 
    YEAR(arrest_date) = @year AND 
    charge_description IS NOT NULL
  GROUP BY charge_description
  ORDER BY arrest_count DESC;
END;

--test
EXEC sp_top_charges_by_area_and_year 
  @area_name = 'Hollywood', 
  @year = 2019,
  @top_n =5;
```
7. Detect repeat arrests for same individual (e.g., same address, age, and gender)?
```sql
SELECT 
  address,
  age,
  sex_code,
  COUNT(*) AS arrest_count
FROM ArrestData
WHERE 
  address IS NOT NULL AND
  age IS NOT NULL AND
  sex_code IS NOT NULL
GROUP BY address, age, sex_code
HAVING COUNT(*) > 1
ORDER BY arrest_count DESC;
```
8. Which age group is most arrested for each charge group?
```sql
WITH AgeGroups AS (
  SELECT 
    charge_group_description,
    CASE 
      WHEN age < 18 THEN '0-17'
      WHEN age BETWEEN 18 AND 25 THEN '18-25'
      WHEN age BETWEEN 26 AND 35 THEN '26-35'
      WHEN age BETWEEN 36 AND 45 THEN '36-45'
      WHEN age BETWEEN 46 AND 60 THEN '46-60'
      ELSE '60+' 
    END AS age_group,
    COUNT(*) AS arrest_count
  FROM ArrestData
  WHERE age IS NOT NULL AND charge_group_description IS NOT NULL
  GROUP BY 
    charge_group_description,
    CASE 
      WHEN age < 18 THEN '0-17'
      WHEN age BETWEEN 18 AND 25 THEN '18-25'
      WHEN age BETWEEN 26 AND 35 THEN '26-35'
      WHEN age BETWEEN 36 AND 45 THEN '36-45'
      WHEN age BETWEEN 46 AND 60 THEN '46-60'
      ELSE '60+' 
    END
),
RankedAgeGroups AS (
  SELECT *,
    RANK() OVER (
      PARTITION BY charge_group_description 
      ORDER BY arrest_count DESC
    ) AS rank
  FROM AgeGroups
)
SELECT charge_group_description, age_group, arrest_count
FROM RankedAgeGroups
WHERE rank = 1
ORDER BY charge_group_description;
```
9. What is the average number of arrests per area per year?
```sql
WITH YearlyArrests AS (
  SELECT 
    area_name,
    YEAR(arrest_date) AS arrest_year,
    COUNT(*) AS yearly_arrest_count
  FROM ArrestData
  WHERE 
    area_name IS NOT NULL AND 
    arrest_date IS NOT NULL
  GROUP BY 
    area_name, 
    YEAR(arrest_date)
)
SELECT 
  area_name,
  AVG(yearly_arrest_count) AS avg_arrests_per_year
FROM YearlyArrests
GROUP BY area_name
ORDER BY avg_arrests_per_year DESC;
```
10. Which charge groups are increasing or decreasing over time (trend analysis)?
```sql
WITH ChargeYearly AS (
  SELECT 
    charge_group_description,
    YEAR(arrest_date) AS arrest_year,
    COUNT(*) AS total_arrests
  FROM ArrestData
  WHERE 
    charge_group_description IS NOT NULL AND 
    arrest_date IS NOT NULL
  GROUP BY 
    charge_group_description, 
    YEAR(arrest_date)
),
WithLag AS (
  SELECT 
    charge_group_description,
    arrest_year,
    total_arrests,
    LAG(total_arrests) OVER (
      PARTITION BY charge_group_description 
      ORDER BY arrest_year
    ) AS previous_year_arrests
  FROM ChargeYearly
),
WithChange AS (
  SELECT 
    charge_group_description,
    arrest_year,
    total_arrests,
    previous_year_arrests,
    total_arrests - previous_year_arrests AS year_over_year_change,
    ROUND(
      (1.0 * (total_arrests - previous_year_arrests)) / NULLIF(previous_year_arrests, 0) * 100, 2
    ) AS percent_change
  FROM WithLag
)
SELECT *
FROM WithChange
ORDER BY charge_group_description, arrest_year;
```
### How to Use
1. Clone the repository.

2. Use the provided CREATE TABLE script to create arrest_data in your SQL Server.

3. Load your dataset.

4. Run the included SQL scripts for each question or modify them for your own analysis.

5. Connect Power BI or other BI tools for visualization.

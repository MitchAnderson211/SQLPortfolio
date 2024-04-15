# Real Estate Market Analysis for Investment Opportunities
## Project Description
This project is a detailed analysis of real estate sales data from 2002 to 2018 in Milwaukee WI, and the surrounding areas. The end-goal was to provide a holistic metric that would allow a potential investor to identify districts with the highest likelihood of investment success. The project utilizes SQL for data manipulation and analysis including data cleaning, exploratory analysis, and advanced metrics computation such as Z-score normalization.
## Data Source
This project utilized the "2002-2018 Property Sales Data" from the following link:
https://www.kaggle.com/datasets/agungpambudi/property-sales-data-real-estate-trends
## Table Creation
```sql
CREATE TABLE Properties(
	proptype VARCHAR(30),
	taxkey BIGINT,
	address VARCHAR(100),
	condoproject VARCHAR(50),
	district INT,
	nbhd INT,
	style VARCHAR(250),
	extwall VARCHAR(50),
	stories DECIMAL,
	year_built INT,
	nr_of_rms INT,
	fin_sqft INT,
	units INT,
	bdrms INT,
	fbath INT,
	hbath INT,
	lotsize INT,
	-- Source Dataset for sale_date has format of YYYY-MM, which cannot be imported as DATE data type.
	sale_date VARCHAR(10),
	sale_price INT
);
```
## Correcting Data Types
Since the sale_date column could not be imported as DATE data type due to having the format of YYYY-MM, this code accomplishes amending all dates to include a date which is sufficient as granularity to the month is all that is needed for this project's analysis. It also alters the sale_date colume to have the data type of DATE.
```sql
ALTER TABLE properties
ALTER COLUMN sale_date TYPE DATE
USING TO_DATE(sale_date || '-01', 'YYYY-MM-DD');
```
## Average Annual Price Increase by District
```sql
CREATE VIEW DistrictAnnualPriceIncreases AS
-- Identify properties with more than one sale record.
WITH RepeatedSales AS (
    SELECT
        taxkey,
        COUNT(*) AS num_sales
    FROM
        properties
    GROUP BY
        taxkey
    HAVING
        COUNT(*) > 1
),
--  Calculate the price change between sales of the same property.
PropertyPriceChanges AS (
    SELECT
        p.district,
        p.taxkey,
        EXTRACT(YEAR FROM p.sale_date) AS year,
        LEAD(p.sale_price, 1) OVER (PARTITION BY p.taxkey ORDER BY p.sale_date) - p.sale_price AS price_change,
        LEAD(p.sale_date, 1) OVER (PARTITION BY p.taxkey ORDER BY p.sale_date) AS next_sale_date,
        p.sale_date AS prev_sale_date
    FROM
        properties p
    JOIN 
        RepeatedSales rs ON p.taxkey = rs.taxkey
),
-- Calculate average annual price increase by district based on price changes.
DistrictPriceTrends AS (
    SELECT
        district,
        AVG(price_change / NULLIF(EXTRACT(YEAR FROM next_sale_date) - EXTRACT(YEAR FROM prev_sale_date), 0)) AS avg_annual_price_increase
    FROM
        PropertyPriceChanges
    GROUP BY
        district
);
```
## Average Monthly Sales by District 
```sql
CREATE VIEW DistrictAverageMonthlySales AS
-- Calculate total number of sales per month for each district.
WITH MonthlySalesVolume AS (
    SELECT
        district,
        EXTRACT(YEAR FROM sale_date) AS year,
        EXTRACT(MONTH FROM sale_date) AS month,
        COUNT(*) AS sales_count
    FROM
        properties
    GROUP BY
        district, year, month
),
-- Calculate average monthly sales per district.
AverageMonthlySales AS (
    SELECT
        district,
        AVG(sales_count) AS avg_monthly_sales
    FROM
        MonthlySalesVolume
    GROUP BY
        district
);
```
## Statistics
```sql
-- Create View for Annual Price Increase Stats (Mean and StdDev)
CREATE VIEW AnnualPriceIncreaseStats AS
SELECT
    AVG(avg_annual_price_increase) AS mean_price_increase,
    STDDEV(avg_annual_price_increase) AS stddev_price_increase
FROM
    DistrictAnnualPriceIncreases;

-- Create View for Monthly Sales Stats (Mean and StdDev)
CREATE VIEW MonthlySalesStats AS
SELECT
    AVG(avg_monthly_sales) AS mean_monthly_sales,
    STDDEV(avg_monthly_sales) AS stddev_monthly_sales
FROM
    DistrictAverageMonthlySales;
```
## Investment Score (Z-Score Normalization)
```sql
CREATE VIEW DistrictInvestmentZScore AS
SELECT
    api.district,
-- Perform Z-Scale Normalization for Annual Price Increase and Monthly Sales. Also introduce weighting (70% annual price increase, 30% average monthly sales)
    (0.7 * (api.avg_annual_price_increase - pi_stats.mean_price_increase) / pi_stats.stddev_price_increase +
     0.3 * (ms.avg_monthly_sales - ms_stats.mean_monthly_sales) / ms_stats.stddev_monthly_sales) AS investment_score
FROM
    DistrictAnnualPriceIncreases api
JOIN
    DistrictAverageMonthlySales ms ON api.district = ms.district,
    AnnualPriceIncreaseStats pi_stats,
    MonthlySalesStats ms_stats
ORDER BY
    investment_score DESC;
```
## Final Investment Score Results
```sql
SELECT 
	district,
-- Cast investment score as DECIMAL to limit to 2 decimal places.
	  cast(investment_score AS DECIMAL (10,2)) 
FROM DistrictInvestmentZScore;
```
"district"	"investment_score"
4	2.01
7	0.55
14	0.28
9	0.09
3	0.04
12	0.02
6	0.00
11	-0.03
2	-0.11
13	-0.13
5	-0.23
15	-0.32
10	-0.47
8	-0.85
1	-0.85

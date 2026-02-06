# Case Study #5 - Data Mart

<img width="540" height="540" alt="image" src="https://github.com/user-attachments/assets/5871e7c4-11a9-428f-893c-948a400efa5e" />

## Summary

Data Mart examines the impact of sustainability-driven packaging changes introduced in June 2020 across an international online supermarket. The goal is to quantify sales performance before and after the change, identify the most affected platforms, regions, segments, and customer types, and provide insights to guide future sustainability initiatives with minimal impact on sales.

## Entity Relationship Diagram
<img width="336" height="400" alt="image" src="https://github.com/user-attachments/assets/e18041bf-b36e-48f9-badf-4a5a9e7112b9" />

## Questions, Queries, and Solutions
All queries executed using PostgreSQL on [DB Fiddle](https://www.db-fiddle.com/f/jmnwogTsUE8hGqkZv9H7E8/8). Follow the link to view the schema and feel free to copy and paste the codes to see them in action.

### 1. Data Cleansing Steps

+ In a single query, perform the following operations and generate a new table in the `data_mart` schema named `clean_weekly_sales`:
+ Convert the `week_date` to a `DATE` format
+ Add a `week_number` as the second column for each `week_date` value, for example any value from the 1st of January to 7th of January will be 1, 8th to 14th will be 2 etc
+ Add a `month_number` with the calendar month for each `week_date` value as the 3rd column
+ Add a `calendar_year` column as the 4th column containing either 2018, 2019 or 2020 values
+ Add a new column called `age_band` after the original `segment` column using the following mapping on the number inside the `segment` value

| segment | age_band     |
| ------- | ------------ |
| 1       | Young Adults |
| 2       | Middle Aged  |
| 3 or 4  | Retirees     |

+ Add a new `demographic` column using the following mapping for the first letter in the `segment` values:

| segment | demographic |
| ------- | ----------- |
| C       | Couples     |
| F       | Families    |



+ Ensure all `null` string values with an "unknown" string value in the original `segment` column as well as the new `age_band` and `demographic` columns

+ Generate a new `avg_transaction` column as the `sales` value divided by transactions rounded to 2 decimal places for each record
````sql
DROP TABLE IF EXISTS clean_weekly_sales;	
CREATE TEMP TABLE clean_weekly_sales AS (	
	SELECT	
		TO_DATE(week_date, 'DD/MM/YY') AS week_date,	
		DATE_PART('week', TO_DATE(week_date, 'DD/MM/YY')) AS week_number,	
		DATE_PART('month', TO_DATE(week_date, 'DD/MM/YY')) AS month_number,	
		DATE_PART('year', TO_DATE(week_date, 'DD/MM/YY')) AS calendar_year,	
		region,	
		platform,	
		segment,	
		CASE	
			WHEN RIGHT(segment,1) = '1' THEN 'Young Adults'	
			WHEN RIGHT(segment,1) = '2' THEN 'Middle Aged'	
			WHEN RIGHT(segment,1) in ('3','4') THEN 'Retirees'	
			ELSE 'unknown'
		END AS age_band,	
		CASE	
			WHEN LEFT(segment,1) = 'C' THEN 'Couples'	
			WHEN LEFT(segment,1) = 'F' THEN 'Families'	
			ELSE 'unknown'
		END AS demographic,	
		transactions,	
		ROUND((sales::NUMERIC/transactions),2) AS avg_transaction,	
		sales	
	FROM data_mart.weekly_sales	
);	
	
SELECT *	
	FROM clean_weekly_sales	
LIMIT 10;
````
Result:
| week_date                | week_number | month_number | calendar_year | region | platform | segment | age_band     | demographic | transactions | avg_transaction | sales    |
| ------------------------ | ----------- | ------------ | ------------- | ------ | -------- | ------- | ------------ | ----------- | ------------ | --------------- | -------- |
| 2020-08-31T00:00:00.000Z | 36          | 8            | 2020          | ASIA   | Retail   | C3      | Retirees     | Couples     | 120631       | 30.31           | 3656163  |
| 2020-08-31T00:00:00.000Z | 36          | 8            | 2020          | ASIA   | Retail   | F1      | Young Adults | Families    | 31574        | 31.56           | 996575   |
| 2020-08-31T00:00:00.000Z | 36          | 8            | 2020          | USA    | Retail   | null    | unknown      | unknown     | 529151       | 31.2            | 16509610 |
| 2020-08-31T00:00:00.000Z | 36          | 8            | 2020          | EUROPE | Retail   | C1      | Young Adults | Couples     | 4517         | 31.42           | 141942   |
| 2020-08-31T00:00:00.000Z | 36          | 8            | 2020          | AFRICA | Retail   | C2      | Middle Aged  | Couples     | 58046        | 30.29           | 1758388  |
| 2020-08-31T00:00:00.000Z | 36          | 8            | 2020          | CANADA | Shopify  | F2      | Middle Aged  | Families    | 1336         | 182.54          | 243878   |
| 2020-08-31T00:00:00.000Z | 36          | 8            | 2020          | AFRICA | Shopify  | F3      | Retirees     | Families    | 2514         | 206.64          | 519502   |
| 2020-08-31T00:00:00.000Z | 36          | 8            | 2020          | ASIA   | Shopify  | F1      | Young Adults | Families    | 2158         | 172.11          | 371417   |
| 2020-08-31T00:00:00.000Z | 36          | 8            | 2020          | AFRICA | Shopify  | F2      | Middle Aged  | Families    | 318          | 155.84          | 49557    |
| 2020-08-31T00:00:00.000Z | 36          | 8            | 2020          | AFRICA | Retail   | C3      | Retirees     | Couples     | 111032       | 35.02           | 3888162  |
#

### 2. Data Exploration

**1. What day of the week is used for each `week_date` value?**
````sql
SELECT
  DISTINCT(TO_CHAR(week_date, 'day')) AS week_day	
FROM clean_weekly_sales;	
````
Result:
| week_day |
| -------- |
| monday   |
#
**2. What range of week numbers are missing from the dataset?**

Many of the answers online only responded with the missing week number for the 1st year (2018), I was able to get all of them for 2018 - 2020
````sql
WITH week_order AS (	
  SELECT	
    DISTINCT week_date,
    week_number,
    month_number,
    calendar_year  
  FROM clean_weekly_sales	
  GROUP BY week_date, week_number, month_number, calendar_year	
  ORDER BY week_date	
),	
	
all_weeks AS (	
  SELECT	
  GENERATE_SERIES('2018-03-26'::date, '2020-12-31'::date, '1 week'::interval) AS gen_week
),	
	
gen_weeks AS (	
  SELECT	
    gen_week,	
    EXTRACT(WEEK FROM gen_week) AS gen_week_number,	
    EXTRACT(MONTH FROM gen_week) AS gen_month_number,
    EXTRACT(YEAR FROM gen_week) AS gen_year_number
  FROM all_weeks	
)	
	
SELECT	
  gen_week_number AS missing_weeks,
  gen_month_number AS month,	
  gen_year_number AS year	
FROM gen_weeks	
LEFT JOIN week_order	
  ON gen_weeks.gen_week_number = week_order.week_number	
WHERE week_order.week_number IS NULL;
````
*Many other answers online only responded with the missing week number for the 1st year (2018). I was able to get all of them for 2018 - 2020.*

Result:
| missing_weeks | month | year |
| ------------- | ----- | ---- |
| 37            | 9     | 2018 |
| 38            | 9     | 2018 |
| 39            | 9     | 2018 |
| 40            | 10    | 2018 |
| 41            | 10    | 2018 |
| 42            | 10    | 2018 |
| 43            | 10    | 2018 |
| 44            | 10    | 2018 |
| 45            | 11    | 2018 |
| 46            | 11    | 2018 |
| 47            | 11    | 2018 |
| 48            | 11    | 2018 |
| 49            | 12    | 2018 |
| 50            | 12    | 2018 |
| 51            | 12    | 2018 |
| 52            | 12    | 2018 |
| 1             | 12    | 2018 |
| 2             | 1     | 2019 |
| 3             | 1     | 2019 |
| 4             | 1     | 2019 |
| 5             | 1     | 2019 |
| 6             | 2     | 2019 |
| 7             | 2     | 2019 |
| 8             | 2     | 2019 |
| 9             | 2     | 2019 |
| 10            | 3     | 2019 |
| 11            | 3     | 2019 |
| 12            | 3     | 2019 |
| 37            | 9     | 2019 |
| 38            | 9     | 2019 |
| 39            | 9     | 2019 |
| 40            | 9     | 2019 |
| 41            | 10    | 2019 |
| 42            | 10    | 2019 |
| 43            | 10    | 2019 |
| 44            | 10    | 2019 |
| 45            | 11    | 2019 |
| 46            | 11    | 2019 |
| 47            | 11    | 2019 |
| 48            | 11    | 2019 |
| 49            | 12    | 2019 |
| 50            | 12    | 2019 |
| 51            | 12    | 2019 |
| 52            | 12    | 2019 |
| 1             | 12    | 2019 |
| 2             | 1     | 2020 |
| 3             | 1     | 2020 |
| 4             | 1     | 2020 |
| 5             | 1     | 2020 |
| 6             | 2     | 2020 |
| 7             | 2     | 2020 |
| 8             | 2     | 2020 |
| 9             | 2     | 2020 |
| 10            | 3     | 2020 |
| 11            | 3     | 2020 |
| 12            | 3     | 2020 |
| 37            | 9     | 2020 |
| 38            | 9     | 2020 |
| 39            | 9     | 2020 |
| 40            | 9     | 2020 |
| 41            | 10    | 2020 |
| 42            | 10    | 2020 |
| 43            | 10    | 2020 |
| 44            | 10    | 2020 |
| 45            | 11    | 2020 |
| 46            | 11    | 2020 |
| 47            | 11    | 2020 |
| 48            | 11    | 2020 |
| 49            | 11    | 2020 |
| 50            | 12    | 2020 |
| 51            | 12    | 2020 |
| 52            | 12    | 2020 |
| 53            | 12    | 2020 |
#
**3. How many total transactions were there for each year in the dataset?**
````sql
SELECT		
  calendar_year,	
  SUM(transactions) AS total_transactions		
FROM clean_weekly_sales		
GROUP BY calendar_year		
ORDER BY calendar_year;		
````
Result:
| calendar_year | total_transactions |
| ------------- | ------------------ |
| 2018          | 346406460          |
| 2019          | 365639285          |
| 2020          | 375813651          |
#
**4. What is the total sales for each region for each month?**

*I considered March 2018 and March 2019 etc as different months in my answer (other answers online did not). If you want to see every montht, follow the query below. Otherwise, if you want all the months grouped together from each year (maybe to compare seasonality) switch the months and year in the ORDER BY line.*
````sql		
SELECT		
  region,	
  month_number,	
  calendar_year,		
  SUM(sales) AS total_sales		
FROM clean_weekly_sales		
GROUP BY region, month_number, calendar_year		
ORDER BY region, calendar_year, month_number;		
````
Result:
| region        | month_number | calendar_year | toal_sales |
| ------------- | ------------ | ------------- | ---------- |
| AFRICA        | 3            | 2018          | 130542213  |
| AFRICA        | 4            | 2018          | 650194751  |
| AFRICA        | 5            | 2018          | 522814997  |
| AFRICA        | 6            | 2018          | 519127094  |
| AFRICA        | 7            | 2018          | 674135866  |
| AFRICA        | 8            | 2018          | 539077371  |
| AFRICA        | 9            | 2018          | 135084533  |
| AFRICA        | 3            | 2019          | 141619349  |
| AFRICA        | 4            | 2019          | 700447301  |
| AFRICA        | 5            | 2019          | 553828220  |
| AFRICA        | 6            | 2019          | 546092640  |
| AFRICA        | 7            | 2019          | 711867600  |
| AFRICA        | 8            | 2019          | 564497281  |
| AFRICA        | 9            | 2019          | 141236454  |
| AFRICA        | 3            | 2020          | 295605918  |
| AFRICA        | 4            | 2020          | 561141452  |
| AFRICA        | 5            | 2020          | 570601521  |
| AFRICA        | 6            | 2020          | 702340026  |
| AFRICA        | 7            | 2020          | 574216244  |
| AFRICA        | 8            | 2020          | 706022238  |
| ASIA          | 3            | 2018          | 119180883  |
| ASIA          | 4            | 2018          | 603716301  |
| ASIA          | 5            | 2018          | 472634283  |
| ASIA          | 6            | 2018          | 462233474  |
| ASIA          | 7            | 2018          | 602910228  |
| ASIA          | 8            | 2018          | 486137188  |
| ASIA          | 9            | 2018          | 122529255  |
| ASIA          | 3            | 2019          | 129174041  |
| ASIA          | 4            | 2019          | 654973051  |
| ASIA          | 5            | 2019          | 511773780  |
| ASIA          | 6            | 2019          | 498386324  |
| ASIA          | 7            | 2019          | 635366443  |
| ASIA          | 8            | 2019          | 514795070  |
| ASIA          | 9            | 2019          | 130307552  |
| ASIA          | 3            | 2020          | 281415869  |
| ASIA          | 4            | 2020          | 545939355  |
| ASIA          | 5            | 2020          | 541877336  |
| ASIA          | 6            | 2020          | 658863091  |
| ASIA          | 7            | 2020          | 530568085  |
| ASIA          | 8            | 2020          | 662388351  |
| CANADA        | 3            | 2018          | 33815571   |
| CANADA        | 4            | 2018          | 163479820  |
| CANADA        | 5            | 2018          | 130367940  |
| CANADA        | 6            | 2018          | 130410790  |
| CANADA        | 7            | 2018          | 164198426  |
| CANADA        | 8            | 2018          | 133635800  |
| CANADA        | 9            | 2018          | 34042238   |
| CANADA        | 3            | 2019          | 36087248   |
| CANADA        | 4            | 2019          | 179830236  |
| CANADA        | 5            | 2019          | 140979946  |
| CANADA        | 6            | 2019          | 138690815  |
| CANADA        | 7            | 2019          | 173991586  |
| CANADA        | 8            | 2019          | 139428879  |
| CANADA        | 9            | 2019          | 35025721   |
| CANADA        | 3            | 2020          | 74731510   |
| CANADA        | 4            | 2020          | 141242538  |
| CANADA        | 5            | 2020          | 141030479  |
| CANADA        | 6            | 2020          | 174745093  |
| CANADA        | 7            | 2020          | 138944935  |
| CANADA        | 8            | 2020          | 174008340  |
| EUROPE        | 3            | 2018          | 8402183    |
| EUROPE        | 4            | 2018          | 44549418   |
| EUROPE        | 5            | 2018          | 36492553   |
| EUROPE        | 6            | 2018          | 38998277   |
| EUROPE        | 7            | 2018          | 50535910   |
| EUROPE        | 8            | 2018          | 39104650   |
| EUROPE        | 9            | 2018          | 9777575    |
| EUROPE        | 3            | 2019          | 8989328    |
| EUROPE        | 4            | 2019          | 46983044   |
| EUROPE        | 5            | 2019          | 36446510   |
| EUROPE        | 6            | 2019          | 36464369   |
| EUROPE        | 7            | 2019          | 47154102   |
| EUROPE        | 8            | 2019          | 36638154   |
| EUROPE        | 9            | 2019          | 9099858    |
| EUROPE        | 3            | 2020          | 17945582   |
| EUROPE        | 4            | 2020          | 35801793   |
| EUROPE        | 5            | 2020          | 36399326   |
| EUROPE        | 6            | 2020          | 47351180   |
| EUROPE        | 7            | 2020          | 39067454   |
| EUROPE        | 8            | 2020          | 46360191   |
| OCEANIA       | 3            | 2018          | 175777460  |
| OCEANIA       | 4            | 2018          | 869324594  |
| OCEANIA       | 5            | 2018          | 692610094  |
| OCEANIA       | 6            | 2018          | 687546255  |
| OCEANIA       | 7            | 2018          | 871333919  |
| OCEANIA       | 8            | 2018          | 714036679  |
| OCEANIA       | 9            | 2018          | 180310608  |
| OCEANIA       | 3            | 2019          | 192331207  |
| OCEANIA       | 4            | 2019          | 953735279  |
| OCEANIA       | 5            | 2019          | 746580473  |
| OCEANIA       | 6            | 2019          | 732354251  |
| OCEANIA       | 7            | 2019          | 934476631  |
| OCEANIA       | 8            | 2019          | 759346286  |
| OCEANIA       | 9            | 2019          | 192154910  |
| OCEANIA       | 3            | 2020          | 415174221  |
| OCEANIA       | 4            | 2020          | 776707747  |
| OCEANIA       | 5            | 2020          | 776466737  |
| OCEANIA       | 6            | 2020          | 951984238  |
| OCEANIA       | 7            | 2020          | 757648850  |
| OCEANIA       | 8            | 2020          | 958930687  |
| SOUTH AMERICA | 3            | 2018          | 16302144   |
| SOUTH AMERICA | 4            | 2018          | 80814046   |
| SOUTH AMERICA | 5            | 2018          | 63685837   |
| SOUTH AMERICA | 6            | 2018          | 63764243   |
| SOUTH AMERICA | 7            | 2018          | 81690746   |
| SOUTH AMERICA | 8            | 2018          | 66079697   |
| SOUTH AMERICA | 9            | 2018          | 16932862   |
| SOUTH AMERICA | 3            | 2019          | 17351683   |
| SOUTH AMERICA | 4            | 2019          | 87069807   |
| SOUTH AMERICA | 5            | 2019          | 67552363   |
| SOUTH AMERICA | 6            | 2019          | 67122227   |
| SOUTH AMERICA | 7            | 2019          | 84577363   |
| SOUTH AMERICA | 8            | 2019          | 68364336   |
| SOUTH AMERICA | 9            | 2019          | 17242721   |
| SOUTH AMERICA | 3            | 2020          | 37369282   |
| SOUTH AMERICA | 4            | 2020          | 70567678   |
| SOUTH AMERICA | 5            | 2020          | 70153609   |
| SOUTH AMERICA | 6            | 2020          | 87360985   |
| SOUTH AMERICA | 7            | 2020          | 69314667   |
| SOUTH AMERICA | 8            | 2020          | 86722019   |
| USA           | 3            | 2018          | 52734998   |
| USA           | 4            | 2018          | 260725717  |
| USA           | 5            | 2018          | 210050720  |
| USA           | 6            | 2018          | 206372070  |
| USA           | 7            | 2018          | 262393377  |
| USA           | 8            | 2018          | 212470882  |
| USA           | 9            | 2018          | 54294291   |
| USA           | 3            | 2019          | 55764198   |
| USA           | 4            | 2019          | 277108603  |
| USA           | 5            | 2019          | 220370520  |
| USA           | 6            | 2019          | 219743295  |
| USA           | 7            | 2019          | 274203066  |
| USA           | 8            | 2019          | 222170302  |
| USA           | 9            | 2019          | 56238077   |
| USA           | 3            | 2020          | 116853847  |
| USA           | 4            | 2020          | 221952003  |
| USA           | 5            | 2020          | 225545881  |
| USA           | 6            | 2020          | 277763625  |
| USA           | 7            | 2020          | 223735311  |
| USA           | 8            | 2020          | 277361606  |
#	
**5. What is the total count of transactions for each platform**
````sql
SELECT		
  platform,	
  SUM(transactions) AS total_transactions		
FROM clean_weekly_sales		
GROUP BY platform		
ORDER BY platform;		
````
Result:
| platform | total_transactions |
| -------- | ------------------ |
| Retail   | 1081934227         |
| Shopify  | 5925169            |
#		
**6. What is the percentage of sales for Retail vs Shopify for each month?**
````sql
WITH sales_ratio AS (		
SELECT		
month_number,	
calendar_year,	
platform,	
SUM(sales) AS monthly_sales	
FROM clean_weekly_sales		
GROUP BY month_number, calendar_year, platform		
)		
		
SELECT		
  calendar_year,		
  month_number,		
  ROUND(100 * MAX(	
    CASE WHEN platform = 'Retail' THEN monthly_sales END)
    /SUM(monthly_sales),2) AS retail_ratio,	
  ROUND(100 * MAX(		
    CASE WHEN platform = 'Shopify' THEN monthly_sales END)
    /SUM(monthly_sales),2) AS shopify_ratio	
FROM sales_ratio		
GROUP BY calendar_year, month_number		
ORDER BY calendar_year, month_number;		
````
Result:
| calendar_year | month_number | retail_ratio | shopify_ratio |
| ------------- | ------------ | ------------ | ------------- |
| 2018          | 3            | 97.92        | 2.08          |
| 2018          | 4            | 97.93        | 2.07          |
| 2018          | 5            | 97.73        | 2.27          |
| 2018          | 6            | 97.76        | 2.24          |
| 2018          | 7            | 97.75        | 2.25          |
| 2018          | 8            | 97.71        | 2.29          |
| 2018          | 9            | 97.68        | 2.32          |
| 2019          | 3            | 97.71        | 2.29          |
| 2019          | 4            | 97.8         | 2.2           |
| 2019          | 5            | 97.52        | 2.48          |
| 2019          | 6            | 97.42        | 2.58          |
| 2019          | 7            | 97.35        | 2.65          |
| 2019          | 8            | 97.21        | 2.79          |
| 2019          | 9            | 97.09        | 2.91          |
| 2020          | 3            | 97.3         | 2.7           |
| 2020          | 4            | 96.96        | 3.04          |
| 2020          | 5            | 96.71        | 3.29          |
| 2020          | 6            | 96.8         | 3.2           |
| 2020          | 7            | 96.67        | 3.33          |
| 2020          | 8            | 96.51        | 3.49          |
#		
**7. What is the percentage of sales by demographic for each year in the dataset?**
````sql
WITH sales_demo AS (		
  SELECT		
    calendar_year,	
    demographic,	
    SUM(sales) AS yearly_sales	
  FROM clean_weekly_sales		
  GROUP BY calendar_year, demographic		
)		
		
SELECT		
  calendar_year,		
  ROUND(100 * MAX(	
    CASE WHEN demographic = 'Couples' THEN yearly_sales END)
    /SUM(yearly_sales),2) AS couples_ratio,	
  ROUND(100 * MAX(		
    CASE WHEN demographic = 'Families' THEN yearly_sales END)
    /SUM(yearly_sales),2) AS families_ratio,	
  ROUND(100 * MAX(		
    CASE WHEN demographic = 'unknown' THEN yearly_sales END)
    /SUM(yearly_sales),2) AS unknown_ratio	
FROM sales_demo		
GROUP BY calendar_year		
ORDER BY calendar_year;		
````
Result:
| calendar_year | couples_ratio | families_ratio | unknown_ratio |
| ------------- | ------------- | -------------- | ------------- |
| 2018          | 26.38         | 31.99          | 41.63         |
| 2019          | 27.28         | 32.47          | 40.25         |
| 2020          | 28.72         | 32.73          | 38.55         |
#		
**8. Which `age_band` and `demographic` values contribute the most to Retail sales?**
````sql		
SELECT		
  age_band,		
  demographic,		
  SUM(sales) as retail_sales,		
  ROUND(100 * SUM(sales)/SUM(SUM(sales)) OVER (), 1) AS contribution_perc		
FROM clean_weekly_sales		
WHERE platform = 'Retail'		
GROUP BY age_band, demographic		
ORDER BY retail_sales DESC;		
````
Result:
| age_band    | demographic | retail_sales | contribution_perc |
| ----------- | ----------- | ------------ | ----------------- |
| unknown     | unknown     | 16067285533  | 40.5              |
| RETIREES    | Families    | 6634686916   | 16.7              |
| RETIREES    | Couples     | 6370580014   | 16.1              |
| MIDDLE AGED | Families    | 4354091554   | 11                |
| YOUNG ADULT | Couples     | 2602922797   | 6.6               |
| MIDDLE AGED | Couples     | 1854160330   | 4.7               |
| YOUNG ADULT | Families    | 1770889293   | 4.5               |
#	
**9. Can we use the `avg_transaction` column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?**	
````sql
SELECT		
  calendar_year,	
  platform,		
  SUM(sales)/SUM(transactions) AS avg_transaction		
FROM clean_weekly_sales		
GROUP BY calendar_year, platform		
ORDER BY calendar_year, platform;
````
Result:
| calendar_year | platform | avg_transaction |
| ------------- | -------- | --------------- |
| 2018          | Retail   | 36              |
| 2018          | Shopify  | 192             |
| 2019          | Retail   | 36              |
| 2019          | Shopify  | 183             |
| 2020          | Retail   | 36              |
| 2020          | Shopify  | 179             |

*No, you do not want to use the `avg_transaction` column as it would be the average of an average using the sales week. Instead, we want to recalculate the same way, but with `sales` and `transactions` grouped by `platform` instead of by sales week.*
#

### 3. Before & After Analysis

This technique is usually used when we inspect an important event and want to inspect the impact before and after a certain point in time.
Taking the `week_date` value of 2020-06-15 as the baseline week where the Data Mart sustainable packaging changes came into effect.
We would include all `week_date` values for 2020-06-15 as the start of the period after the change and the previous `week_date` values would be before

Using this analysis approach - answer the following questions:

**1. What is the total sales for the 4 weeks before and after 2020-06-15? What is the growth or reduction rate in actual values and percentage of sales?**

*To get week_number:*
````sql
SELECT		
	DISTINCT week_date,	
	week_number,		
	month_number,		
	calendar_year		
FROM clean_weekly_sales		
WHERE week_date = '2020-06-15';		
````
| week_date                | week_number | month_number | calendar_year |
| ------------------------ | ----------- | ------------ | ------------- |
| 2020-06-15T00:00:00.000Z | 25          | 6            | 2020          |

*Now we can run this query:*
````sql
WITH week_totals AS (		
	SELECT		
		week_date,	
		week_number,	
		SUM(sales) AS total_sales	
	FROM clean_weekly_sales		
	WHERE calendar_year = 2020		
		AND week_number BETWEEN (25 - 4) AND (25 + 3)		
	GROUP BY week_date, week_number		
),
		
before_after AS (		
	SELECT		
		SUM(CASE	
				WHEN week_number BETWEEN 21 AND 24 THEN total_sales
			END) AS before_change,		
		SUM(CASE	
				WHEN week_number BETWEEN 25 AND 28 THEN total_sales
			END) AS after_change		
	FROM week_totals		
)		
		
SELECT		
	*,	
	after_change - before_change AS sales_variance,		
	ROUND(100 * (after_change - before_change) /
		before_change, 2) AS percent_change		
FROM before_after;		
````
Result:
| before_change | after_change | sales_variance | percent_change |
| ------------- | ------------ | -------------- | -------------- |
| 2345878357    | 2318994169   | \-26884188     | \-1.15         |
#		
**2. What about the entire 12 weeks before and after?**
````sql
WITH week_totals AS (		
	SELECT		
		week_date,	
		week_number,	
		SUM(sales) AS total_sales	
	FROM clean_weekly_sales		
	WHERE calendar_year = 2020		
		AND week_number BETWEEN (25 - 12) AND (25 + 11)		
	GROUP BY week_date, week_number			
),

before_after AS (		
	SELECT		
		SUM(CASE	
				WHEN week_number BETWEEN 13 AND 24 THEN total_sales
			END) AS before_change,		
		SUM(CASE	
				WHEN week_number BETWEEN 25 AND 36 THEN total_sales
			END) AS after_change		
	FROM week_totals		
)		
		
SELECT		
	*,	
	after_change - before_change AS sales_variance,		
	ROUND(100 * (after_change - before_change) /
		before_change, 2) AS percent_change		
FROM before_after;		
````
Result:
| before_change | after_change | sales_variance | percent_change |
| ------------- | ------------ | -------------- | -------------- |
| 7126273147    | 6973947753   | \-152325394    | \-2.14         |
#		
**3. How do the sale metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?**

*12 week period before and after 6/15:*
````sql		
WITH week_totals AS (		
	SELECT		
		week_date,	
		week_number,	
		calendar_year,	
		SUM(sales) AS total_sales		
	FROM clean_weekly_sales		
	WHERE week_number BETWEEN (25 - 12) AND (25 + 11)		
	GROUP BY week_date, week_number, calendar_year		
),
	
before_after AS (		
	SELECT		
		calendar_year,	
		SUM(CASE	
				WHEN week_number BETWEEN 13 AND 24 THEN total_sales
			END) AS before_change,		
		SUM(CASE	
				WHEN week_number BETWEEN 25 AND 36 THEN total_sales
			END) AS after_change		
	FROM week_totals		
	GROUP BY calendar_year		
)		
		
SELECT		
	*,	
	after_change - before_change AS sales_variance,		
	ROUND(100 * (after_change - before_change) /
		before_change, 2) AS percent_change		
FROM before_after;		
````
Result for 12 week period:
| calendar_year | before_change | after_change | sales_variance | percent_change |
| ------------- | ------------- | ------------ | -------------- | -------------- |
| 2018          | 6396562317    | 6500818510   | 104256193      | 1.63           |
| 2019          | 6883386397    | 6862646103   | \-20740294     | \-0.3          |
| 2020          | 7126273147    | 6973947753   | \-152325394    | \-2.14         |
#		
*4 week period before and after 6/15:*
````sql
WITH week_totals AS (		
	SELECT		
		week_date,	
		week_number,	
		calendar_year,	
		SUM(sales) AS total_sales	
	FROM clean_weekly_sales		
	WHERE week_number BETWEEN (25 - 4) AND (25 + 3)		
	GROUP BY week_date, week_number, calendar_year		
),
	
before_after AS (		
	SELECT		
		calendar_year,	
		SUM(CASE	
				WHEN week_number BETWEEN 21 AND 24 THEN total_sales
			END) AS before_change,		
		SUM(CASE	
				WHEN week_number BETWEEN 25 AND 28 THEN total_sales
			END) AS after_change		
	FROM week_totals		
	GROUP BY calendar_year		
)		
		
SELECT		
	*,	
	after_change - before_change AS sales_variance,		
	ROUND(100 * (after_change - before_change) /
		before_change, 2) AS percent_change		
FROM before_after;
````
Result for 4 week period:
| calendar_year | before_change | after_change | sales_variance | percent_change |
| ------------- | ------------- | ------------ | -------------- | -------------- |
| 2018          | 2125140809    | 2129242914   | 4102105        | 0.19           |
| 2019          | 2249989796    | 2252326390   | 2336594        | 0.1            |
| 2020          | 2345878357    | 2318994169   | \-26884188     | \-1.15         |
#

### 4. Bonus Question

Which areas of the business have the highest negative impact in sales metrics performance in 2020 for the 12 week before and after period?

* `region`
* `platform`
* `age_band`
* `demographic`
* `customer_type`

Do you have any further recommendations for Danny’s team at Data Mart or any interesting insights based off this analysis?

**`region`**
````sql
WITH week_totals AS (	
	SELECT	
		week_date,
		week_number,
		region,
		SUM(sales) AS total_sales
	FROM clean_weekly_sales	
	WHERE calendar_year = 2020	
		AND week_number BETWEEN (25 - 12) AND (25 + 11)	
	GROUP BY week_date, week_number, region	
),

before_after AS (	
	SELECT	
		region,
		SUM(CASE
				WHEN week_number BETWEEN 13 AND 24 THEN total_sales
			END) AS before_change,	
		SUM(CASE
				WHEN week_number BETWEEN 25 AND 36 THEN total_sales
			END) AS after_change	
	FROM week_totals	
	GROUP BY region	
)	
	
SELECT	
	*,
	after_change - before_change AS sales_variance,	
	ROUND(100 * (after_change - before_change) /
		before_change, 2) AS percent_change	
FROM before_after	
ORDER BY percent_change ASC;	
````
Result:
| region        | before_change | after_change | sales_variance | percent_change |
| ------------- | ------------- | ------------ | -------------- | -------------- |
| ASIA          | 1637244466    | 1583807621   | \-53436845     | \-3.26         |
| OCEANIA       | 2354116790    | 2282795690   | \-71321100     | \-3.03         |
| SOUTH AMERICA | 213036207     | 208452033    | \-4584174      | \-2.15         |
| CANADA        | 426438454     | 418264441    | \-8174013      | \-1.92         |
| USA           | 677013558     | 666198715    | \-10814843     | \-1.6          |
| AFRICA        | 1709537105    | 1700390294   | \-9146811      | \-0.54         |
| EUROPE        | 108886567     | 114038959    | 5152392        | 4.73           |

*Insight: All regions, except Europe, were negatively impacted. Asia the most, however, Europe increased by more than Asia decreased.*
#	
**`platform`**
````sql	
WITH week_totals AS (	
	SELECT	
		week_date,
		week_number,
		platform,
		SUM(sales) AS total_sales
	FROM clean_weekly_sales	
	WHERE calendar_year = 2020	
		AND week_number BETWEEN (25 - 12) AND (25 + 11)	
	GROUP BY week_date, week_number, platform	
),

before_after AS (	
	SELECT	
		platform,
		SUM(CASE
				WHEN week_number BETWEEN 13 AND 24 THEN total_sales
			END) AS before_change,	
		SUM(CASE
				WHEN week_number BETWEEN 25 AND 36 THEN total_sales
			END) AS after_change	
	FROM week_totals	
	GROUP BY platform	
)	
	
SELECT	
	*,
	after_change - before_change AS sales_variance,	
	ROUND(100 * (after_change - before_change) /
		before_change, 2) AS percent_change	
FROM before_after	
ORDER BY percent_change ASC;	
````
Result:
| platform | before_change | after_change | sales_variance | percent_change |
| -------- | ------------- | ------------ | -------------- | -------------- |
| Retail   | 6906861113    | 6738777279   | \-168083834    | \-2.43         |
| Shopify  | 219412034     | 235170474    | 15758440       | 7.18           |

*Insight: Retail decreased, but Shopify increased by far more than the decrease.*
#	
**`age_band`**
````sql	
WITH week_totals AS (	
	SELECT	
		week_date,
		week_number,
		age_band,
		SUM(sales) AS total_sales
	FROM clean_weekly_sales	
	WHERE calendar_year = 2020	
		AND week_number BETWEEN (25 - 12) AND (25 + 11)	
	GROUP BY week_date, week_number, age_band	
),

before_after AS (	
	SELECT	
		age_band,
		SUM(CASE
				WHEN week_number BETWEEN 13 AND 24 THEN total_sales
			END) AS before_change,	
		SUM(CASE
				WHEN week_number BETWEEN 25 AND 36 THEN total_sales
			END) AS after_change	
	FROM week_totals	
	GROUP BY age_band	
)	
	
SELECT	
	*,
	after_change - before_change AS sales_variance,	
	ROUND(100 * (after_change - before_change) /
		before_change, 2) AS percent_change	
FROM before_after	
ORDER BY percent_change ASC;	
````
Result:
| age_band     | before_change | after_change | sales_variance | percent_change |
| ------------ | ------------- | ------------ | -------------- | -------------- |
| unknown      | 2764354464    | 2671961443   | \-92393021     | \-3.34         |
| Middle Aged  | 1164847640    | 1141853348   | \-22994292     | \-1.97         |
| Retirees     | 2395264515    | 2365714994   | \-29549521     | \-1.23         |
| Young Adults | 801806528     | 794417968    | \-7388560      | \-0.92         |

*Insight: Every demographic decreased. We need more data on the unkown category since it was the largest decrease, but middle aged was the largest known decrease.*
#		
**`demographic`**
````sql	
WITH week_totals AS (	
	SELECT	
		week_date,
		week_number,
		demographic,
		SUM(sales) AS total_sales
	FROM clean_weekly_sales	
	WHERE calendar_year = 2020	
		AND week_number BETWEEN (25 - 12) AND (25 + 11)	
	GROUP BY week_date, week_number, demographic	
),

before_after AS (	
	SELECT	
		demographic,
		SUM(CASE
				WHEN week_number BETWEEN 13 AND 24 THEN total_sales
			END) AS before_change,	
		SUM(CASE
				WHEN week_number BETWEEN 25 AND 36 THEN total_sales
			END) AS after_change	
	FROM week_totals	
	GROUP BY demographic	
)	
	
SELECT	
	*,
	after_change - before_change AS sales_variance,	
	ROUND(100 * (after_change - before_change) /
		before_change, 2) AS percent_change	
FROM before_after	
ORDER BY percent_change ASC;	
````
Result:
| demographic | before_change | after_change | sales_variance | percent_change |
| ----------- | ------------- | ------------ | -------------- | -------------- |
| unknown     | 2764354464    | 2671961443   | \-92393021     | \-3.34         |
| Families    | 2328329040    | 2286009025   | \-42320015     | \-1.82         |
| Couples     | 2033589643    | 2015977285   | \-17612358     | \-0.87         |

*Insight: Again, we need more data on the unknown category, but families were the largest known decrease.*
#		
**`customer_type`**
````sql
WITH week_totals AS (	
	SELECT	
		week_date,
		week_number,
		customer_type,
		SUM(sales) AS total_sales
	FROM clean_weekly_sales	
	WHERE calendar_year = 2020	
		AND week_number BETWEEN (25 - 12) AND (25 + 11)	
	GROUP BY week_date, week_number, customer_type	
),

before_after AS (	
	SELECT	
		customer_type,
		SUM(CASE
				WHEN week_number BETWEEN 13 AND 24 THEN total_sales
			END) AS before_change,	
		SUM(CASE
				WHEN week_number BETWEEN 25 AND 36 THEN total_sales
			END) AS after_change	
	FROM week_totals	
	GROUP BY customer_type	
)	
	
SELECT	
	*,
	after_change - before_change AS sales_variance,	
	ROUND(100 * (after_change - before_change) /
		before_change, 2) AS percent_change	
FROM before_after	
ORDER BY percent_change ASC;	
````
Result:
| customer_type | before_change | after_change | sales_variance | percent_change |
| ------------- | ------------- | ------------ | -------------- | -------------- |
| Guest         | 2573436301    | 2496233635   | \-77202666     | \-3            |
| Existing      | 3690116427    | 3606243454   | \-83872973     | \-2.27         |
| New           | 862720419     | 871470664    | 8750245        | 1.01           |

*Insight: Guests saw the largest decrease, while new customers actually increased.*
#
**Summary**

Decreases: Where to reconsider sustainable packaging	
* Asia	
* Retail	
* Middle Aged/unknown	
* Family/unknown	
* Guest	
	
Increases: Where to focus sustainability efforts	
* Europe	
* Shopify	
* Young Adults	
* Couples	
* New	
	
**More specifically, new Shopify customers in Europe are the best to focus on.**

#
	
**Further Analysis:**
````sql	
WITH week_totals AS (	
	SELECT	
		week_date,
		week_number,
		region,
		platform,
		age_band,
		demographic,
		customer_type,
		SUM(sales) AS total_sales
	FROM clean_weekly_sales	
	WHERE calendar_year = 2020	
	AND week_number BETWEEN (25 - 12) AND (25 + 11)	
	GROUP BY week_date, week_number, region, platform, age_band, demographic, customer_type	
),

before_after AS (	
	SELECT	
		SUM(CASE
				WHEN week_number BETWEEN 13 AND 24 THEN total_sales
			END) AS before_change,	
		SUM(CASE
				WHEN week_number BETWEEN 25 AND 36 THEN total_sales
			END) AS after_change	
	FROM week_totals	
	WHERE platform = 'Shopify'	
		AND region = 'EUROPE'	
		AND customer_type = 'New'	
	GROUP BY region, platform, age_band, demographic, customer_type	
)	
	
SELECT	
	*,
	after_change - before_change AS sales_variance,	
	ROUND(100 * (after_change - before_change) /
		before_change, 2) AS percent_change	
FROM before_after	
ORDER BY percent_change DESC;	
````
Result:
| region | platform | age_band     | demographic | customer_type | before_change | after_change | sales_variance | percent_change |
| ------ | -------- | ------------ | ----------- | ------------- | ------------- | ------------ | -------------- | -------------- |
| EUROPE | Shopify  | unknown      | unknown     | New           | 27321         | 42930        | 15609          | 57.13          |
| EUROPE | Shopify  | Young Adults | Couples     | New           | 20466         | 26738        | 6272           | 30.65          |
| EUROPE | Shopify  | Retirees     | Couples     | New           | 33426         | 38389        | 4963           | 14.85          |
| EUROPE | Shopify  | Middle Aged  | Couples     | New           | 35446         | 34283        | \-1163         | \-3.28         |
| EUROPE | Shopify  | Middle Aged  | Families    | New           | 27104         | 25920        | \-1184         | \-4.37         |
| EUROPE | Shopify  | Young Adults | Families    | New           | 15863         | 11426        | \-4437         | \-27.97        |
| EUROPE | Shopify  | Retirees     | Families    | New           | 7292          | 4834         | \-2458         | \-33.71        |

*Insight: It would appear of new Shopify European customers, families are not the demographic to target and young couples are the most likely to appreciate sustainability efforts.*
#		
	
**Region ASIA:**
````sql	
WITH week_totals AS (	
	SELECT	
		week_date,
		week_number,
		region,
		platform,
		age_band,
		demographic,
		customer_type,
		SUM(sales) AS total_sales
	FROM clean_weekly_sales	
	WHERE calendar_year = 2020	
		AND week_number BETWEEN (25 - 12) AND (25 + 11)	
	GROUP BY week_date, week_number, region, platform, age_band, demographic, customer_type	
),

before_after AS (	
	SELECT
		SUM(CASE
				WHEN week_number BETWEEN 13 AND 24 THEN total_sales
			END) AS before_change,	
		SUM(CASE
				WHEN week_number BETWEEN 25 AND 36 THEN total_sales
			END) AS after_change	
	FROM week_totals	
	WHERE region = 'ASIA'	
	GROUP BY region, platform, age_band, demographic, customer_type	
)	
	
SELECT	
	*,
	after_change - before_change AS sales_variance,	
	ROUND(100 * (after_change - before_change) /
		before_change, 2) AS percent_change	
FROM before_after	
ORDER BY percent_change ASC;	
````
Result:
| region | platform | age_band     | demographic | customer_type | before_change | after_change | sales_variance | percent_change |
| ------ | -------- | ------------ | ----------- | ------------- | ------------- | ------------ | -------------- | -------------- |
| ASIA   | Retail   | unknown      | unknown     | Existing      | 17658530      | 15616404     | \-2042126      | \-11.56        |
| ASIA   | Retail   | unknown      | unknown     | New           | 26844660      | 24879292     | \-1965368      | \-7.32         |
| ASIA   | Retail   | Retirees     | Couples     | Existing      | 200058271     | 190527375    | \-9530896      | \-4.76         |
| ASIA   | Retail   | unknown      | unknown     | Guest         | 605059313     | 576625191    | \-28434122     | \-4.7          |
| ASIA   | Retail   | Middle Aged  | Families    | Existing      | 131698799     | 125714950    | \-5983849      | \-4.54         |
| ASIA   | Retail   | Young Adults | Families    | Existing      | 55758921      | 53795714     | \-1963207      | \-3.52         |
| ASIA   | Retail   | Retirees     | Families    | Existing      | 244092189     | 237915427    | \-6176762      | \-2.53         |
| ASIA   | Retail   | Middle Aged  | Families    | New           | 25111268      | 24503537     | \-607731       | \-2.42         |
| ASIA   | Retail   | Middle Aged  | Couples     | Existing      | 59833638      | 58490268     | \-1343370      | \-2.25         |
| ASIA   | Retail   | Young Adults | Couples     | Existing      | 73893538      | 73013310     | \-880228       | \-1.19         |
| ASIA   | Retail   | Young Adults | Families    | New           | 11227782      | 11101134     | \-126648       | \-1.13         |
| ASIA   | Retail   | Retirees     | Couples     | New           | 59324635      | 59244521     | \-80114        | \-0.14         |
| ASIA   | Retail   | Middle Aged  | Couples     | New           | 26286614      | 26435301     | 148687         | 0.57           |
| ASIA   | Retail   | Retirees     | Families    | New           | 30872857      | 31111070     | 238213         | 0.77           |
| ASIA   | Shopify  | Middle Aged  | Families    | New           | 343043        | 352751       | 9708           | 2.83           |
| ASIA   | Retail   | Young Adults | Couples     | New           | 31849635      | 32939975     | 1090340        | 3.42           |
| ASIA   | Shopify  | Middle Aged  | Couples     | New           | 616425        | 651204       | 34779          | 5.64           |
| ASIA   | Shopify  | Retirees     | Couples     | Existing      | 4909398       | 5188939      | 279541         | 5.69           |
| ASIA   | Shopify  | Middle Aged  | Couples     | Existing      | 4342258       | 4648779      | 306521         | 7.06           |
| ASIA   | Shopify  | Young Adults | Families    | Existing      | 3916458       | 4228004      | 311546         | 7.95           |
| ASIA   | Shopify  | Retirees     | Couples     | New           | 507015        | 551975       | 44960          | 8.87           |
| ASIA   | Shopify  | Retirees     | Families    | Existing      | 3831458       | 4204826      | 373368         | 9.74           |
| ASIA   | Shopify  | unknown      | unknown     | New           | 498203        | 547406       | 49203          | 9.88           |
| ASIA   | Shopify  | Middle Aged  | Families    | Existing      | 5682304       | 6353872      | 671568         | 11.82          |
| ASIA   | Shopify  | Young Adults | Couples     | Existing      | 2138836       | 2406768      | 267932         | 12.53          |
| ASIA   | Shopify  | Young Adults | Families    | New           | 254328        | 286203       | 31875          | 12.53          |
| ASIA   | Shopify  | unknown      | unknown     | Guest         | 9563388       | 11178672     | 1615284        | 16.89          |
| ASIA   | Shopify  | Young Adults | Couples     | New           | 316078        | 372251       | 56173          | 17.77          |
| ASIA   | Shopify  | Retirees     | Families    | New           | 152417        | 184681       | 32264          | 21.17          |
| ASIA   | Shopify  | unknown      | unknown     | Existing      | 602207        | 737821       | 135614         | 22.52          |

*Insight: In the largest decrease region of Asia, only retail decreased. Efforts could still be focused on	
in the Shopify market of Asia, where every single category increased. Young Adult, New, and Couples were	
the largest of 3 categories to increase for retail.*
	
*Existing Asian retail customers were the largest decrease.*

# Case Study #8 - Fresh Segments

<img width="540" height="540" alt="image" src="https://github.com/user-attachments/assets/9fc9ea63-7a81-4c6a-9494-e932a92a9751" />

## Summary

Fresh Segments focuses on analyzing customer interest and engagement data to better understand how different audience segments interact with digital content. The business aims to use this behavioral data to optimize marketing strategies and improve targeted campaigns.

The objective is to explore and transform large-scale segmentation data to identify high-performing interest categories, remove irrelevant or low-value segments, and generate insights that enable more effective, data-driven marketing decisions.

**Table 1: Interest Metrics**

This table contains information about aggregated interest metrics for a specific major client of Fresh Segments which makes up a large proportion of their customer base. Each record in this table represents the performance of a specific `interest_id` based on the client’s customer base interest measured through clicks and interactions with specific targeted advertising content.

**Table 2: Interest Map**

This mapping table links the `interest_id` with their relevant interest information. You will need to join this table onto the previous `interest_details` table to obtain the `interest_name` as well as any details about the summary information.

## Questions, Queries, and Solutions
All queries executed using PostgreSQL on [DB Fiddle](https://www.db-fiddle.com/f/iRdsT76vaus813crPP8Ma4/10). Follow the link to view the schema and feel free to copy and paste the codes to see them in action.

### A. Data Exploration and Cleansing

**1. Update the `fresh_segments.interest_metrics` table by modifying the `month_year` column to be a date data type with the start of the month**
````sql
SET		
  SEARCH_PATH = fresh_segments;	
ALTER TABLE		
  interest_metrics	
ALTER COLUMN		
  month_year TYPE DATE USING TO_DATE(month_year, 'MM-YYYY');
	
SELECT * FROM interest_metrics		
LIMIT 10;		
````
Result:
| _month | _year | month_year               | interest_id | composition | index_value | ranking | percentile_ranking |
| ------ | ----- | ------------------------ | ----------- | ----------- | ----------- | ------- | ------------------ |
| 7      | 2018  | 2018-07-01T00:00:00.000Z | 32486       | 11.89       | 6.19        | 1       | 99.86              |
| 7      | 2018  | 2018-07-01T00:00:00.000Z | 6106        | 9.93        | 5.31        | 2       | 99.73              |
| 7      | 2018  | 2018-07-01T00:00:00.000Z | 18923       | 10.85       | 5.29        | 3       | 99.59              |
| 7      | 2018  | 2018-07-01T00:00:00.000Z | 6344        | 10.32       | 5.1         | 4       | 99.45              |
| 7      | 2018  | 2018-07-01T00:00:00.000Z | 100         | 10.77       | 5.04        | 5       | 99.31              |
| 7      | 2018  | 2018-07-01T00:00:00.000Z | 69          | 10.82       | 5.03        | 6       | 99.18              |
| 7      | 2018  | 2018-07-01T00:00:00.000Z | 79          | 11.21       | 4.97        | 7       | 99.04              |
| 7      | 2018  | 2018-07-01T00:00:00.000Z | 6111        | 10.71       | 4.83        | 8       | 98.9               |
| 7      | 2018  | 2018-07-01T00:00:00.000Z | 6214        | 9.71        | 4.83        | 8       | 98.9               |
| 7      | 2018  | 2018-07-01T00:00:00.000Z | 19422       | 10.11       | 4.81        | 10      | 98.63              |
#
**2. What is count of records in the `fresh_segments.interest_metrics` for each month_year value sorted in chronological order (earliest to latest) with the null values appearing first?**
````sql
SELECT		
  month_year AS date,	
  COUNT(*) AS records_count		
FROM interest_metrics		
GROUP BY month_year		
ORDER BY month_year ASC NULLS FIRST;		
````
Result:
| date                     | records_count |
| ------------------------ | ------------- |
| null                     | 1194          |
| 2018-07-01T00:00:00.000Z | 729           |
| 2018-08-01T00:00:00.000Z | 767           |
| 2018-09-01T00:00:00.000Z | 780           |
| 2018-10-01T00:00:00.000Z | 857           |
| 2018-11-01T00:00:00.000Z | 928           |
| 2018-12-01T00:00:00.000Z | 995           |
| 2019-01-01T00:00:00.000Z | 973           |
| 2019-02-01T00:00:00.000Z | 1121          |
| 2019-03-01T00:00:00.000Z | 1136          |
| 2019-04-01T00:00:00.000Z | 1099          |
| 2019-05-01T00:00:00.000Z | 857           |
| 2019-06-01T00:00:00.000Z | 824           |
| 2019-07-01T00:00:00.000Z | 864           |
| 2019-08-01T00:00:00.000Z | 1149          |
#		
**3. What do you think we should do with these null values in the `fresh_segments.interest_metrics`**

If `month_year` and `interest_id` columns are nulls, then we can just drop these values, or exclude them because we cannot join them to other tables and cannot understand what the other values, like composition, index, ranking in the rows are about. For example, we see the composition or index but do not know which `interest_id` it belongs to.		
		
**4. How many `interest_id` values exist in the `fresh_segments.interest_metrics` table but not in the `fresh_segments.interest_map` table? What about the other way around?**
````sql
SELECT		
  COUNT(DISTINCT interest_id) AS records_count	
FROM interest_metrics		
WHERE interest_id :: int NOT IN (	
  SELECT		
    id	
  FROM interest_map		
);
````
Result:
| records_count |
| ------------- |
| 0             |

````sql
SELECT		
  COUNT(id) AS records_count	
FROM interest_map		
WHERE	id NOT IN (	
  SELECT		
    DISTINCT interest_id :: int	
  FROM interest_metrics		
  WHERE interest_id IS NOT NULL		
);		
````
Result:
| records_count |
| ------------- |
| 7             |
#		
**5. Summarize the id values in the `fresh_segments.interest_map` by its total record count in this table**
````sql
SELECT		
  COUNT(DISTINCT id) AS records_count	
FROM interest_map;		
````
Result:
| records_count |
| ------------- |
| 1209          |
#		
**6. What sort of table join should we perform for our analysis and why? Check your logic by checking the rows where `interest_id` = 21246 in your joined output and include all columns from `fresh_segments.interest_metrics` and all columns from `fresh_segments.interest_map` except from the id column.**
````sql	
SELECT		
  DISTINCT interest_id :: int,	
  interest_name,		
  interest_summary,		
  created_at,		
  last_modified,		
  _month,		
  _year,		
  month_year,		
  composition,		
  index_value,		
  ranking,		
  percentile_ranking		
FROM interest_map AS map		
LEFT JOIN interest_metrics AS metrics		
  ON map.id = metrics.interest_id :: int		
WHERE interest_id = '21246'		
GROUP BY
  interest_name,
  id,
  interest_summary,
  created_at,
  last_modified,
  _month,
  _year,
  month_year,
  interest_id,
  composition,
  index_value,
  ranking,
  percentile_ranking		
ORDER BY _month NULLS FIRST;		
````
Result:
| interest_id | interest_name                    | interest_summary                                      | created_at               | last_modified            | _month | _year | month_year               | composition | index_value | ranking | percentile_ranking |
| ----------- | -------------------------------- | ----------------------------------------------------- | ------------------------ | ------------------------ | ------ | ----- | ------------------------ | ----------- | ----------- | ------- | ------------------ |
| 21246       | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11T17:50:04.000Z | 2018-06-11T17:50:04.000Z | null   | null  | null                     | 1.61        | 0.68        | 1191    | 0.25               |
| 21246       | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11T17:50:04.000Z | 2018-06-11T17:50:04.000Z | 1      | 2019  | 2019-01-01T00:00:00.000Z | 2.05        | 0.76        | 954     | 1.95               |
| 21246       | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11T17:50:04.000Z | 2018-06-11T17:50:04.000Z | 10     | 2018  | 2018-10-01T00:00:00.000Z | 1.74        | 0.58        | 855     | 0.23               |
| 21246       | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11T17:50:04.000Z | 2018-06-11T17:50:04.000Z | 11     | 2018  | 2018-11-01T00:00:00.000Z | 2.25        | 0.78        | 908     | 2.16               |
| 21246       | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11T17:50:04.000Z | 2018-06-11T17:50:04.000Z | 12     | 2018  | 2018-12-01T00:00:00.000Z | 1.97        | 0.7         | 983     | 1.21               |
| 21246       | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11T17:50:04.000Z | 2018-06-11T17:50:04.000Z | 2      | 2019  | 2019-02-01T00:00:00.000Z | 1.84        | 0.68        | 1109    | 1.07               |
| 21246       | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11T17:50:04.000Z | 2018-06-11T17:50:04.000Z | 3      | 2019  | 2019-03-01T00:00:00.000Z | 1.75        | 0.67        | 1123    | 1.14               |
| 21246       | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11T17:50:04.000Z | 2018-06-11T17:50:04.000Z | 4      | 2019  | 2019-04-01T00:00:00.000Z | 1.58        | 0.63        | 1092    | 0.64               |
| 21246       | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11T17:50:04.000Z | 2018-06-11T17:50:04.000Z | 7      | 2018  | 2018-07-01T00:00:00.000Z | 2.26        | 0.65        | 722     | 0.96               |
| 21246       | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11T17:50:04.000Z | 2018-06-11T17:50:04.000Z | 8      | 2018  | 2018-08-01T00:00:00.000Z | 2.13        | 0.59        | 765     | 0.26               |
| 21246       | Readers of El Salvadoran Content | People reading news from El Salvadoran media sources. | 2018-06-11T17:50:04.000Z | 2018-06-11T17:50:04.000Z | 9      | 2018  | 2018-09-01T00:00:00.000Z | 2.06        | 0.61        | 774     | 0.77               |
#		
**7. Are there any records in your joined table where the `month_year value` is before the `created_at` value from the `fresh_segments.interest_map` table? Do you think these values are valid and why?**
````sql
WITH joined_table AS (		
  SELECT		
    DISTINCT interest_id :: int,	
    interest_name,		
    interest_summary,		
    created_at,		
    last_modified,		
    _month,		
    _year,		
    month_year,		
    composition,		
    index_value,		
    ranking,		
    percentile_ranking		
  FROM interest_map AS map		
  LEFT JOIN interest_metrics AS metrics		
    ON map.id = metrics.interest_id :: int		
  GROUP BY interest_name,
    id,
    interest_summary,
    created_at,
    last_modified,
    _month,
    _year,
    month_year,
    interest_id,
    composition,
    index_value,
    ranking,
    percentile_ranking		
)
	
SELECT		
  COUNT(*)	
FROM joined_table		
WHERE month_year < created_at;		
````
Result:
| count |
| ----- |
| 188   |
		
*We have 188 rows where the `month_year` value is before the created_at value from the `fresh_segments.interest_map` table. I think these values are valid because months are the same, and the value in the `month_year` column has the first day of the month but we do not know the real day of the month as we created this column by combining month and year only.*
#		
### B. Interest Analysis		
**1. Which interests have been present in all `month_year` dates in our dataset?**

*To get a count of total number of interests:*
````sql	
WITH months_cte AS (		
  SELECT		
    interest_id :: int,	
    COUNT(DISTINCT month_year) AS total_months	
  FROM interest_metrics		
  WHERE interest_id IS NOT NULL		
  GROUP BY interest_id		
)		
		
SELECT		
  COUNT(*)	
FROM months_cte		
WHERE total_months = 14;		
````
Result:
| count |
| ----- |
| 480   |
		
*To get a list of those 480 interests in every month:*
````sql	
WITH months_cte AS (		
  SELECT		
    interest_id :: int,	
    COUNT(DISTINCT month_year) AS total_months	
  FROM interest_metrics		
  WHERE interest_id IS NOT NULL		
  GROUP BY interest_id		
)		
		
SELECT		
  interest_name,		
  total_months		
FROM months_cte AS cte		
INNER JOIN interest_map AS map		
  ON cte.interest_id :: int = map.id		
WHERE total_months = 14		
GROUP BY interest_name, total_months
LIMIT 10;		
````
Result:
| interest_name                                     | total_months |
| ------------------------------------------------- | ------------ |
| Accounting & CPA Continuing Education Researchers | 14           |
| Affordable Hotel Bookers                          | 14           |
| Aftermarket Accessories Shoppers                  | 14           |
| Alabama Trip Planners                             | 14           |
| Alaskan Cruise Planners                           | 14           |
| Alzheimer and Dementia Researchers                | 14           |
| Anesthesiologists                                 | 14           |
| Apartment Furniture Shoppers                      | 14           |
| Apartment Hunters                                 | 14           |
| Apple Fans                                        | 14           |
#				
**2. Using this same `total_months` measure - calculate the cumulative percentage of all records starting at 14 months - which `total_months` value passes the 90% cumulative percentage value?**
````sql
WITH months_cte AS (		
  SELECT		
    interest_id :: int,	
    COUNT(DISTINCT month_year) AS total_months	
  FROM fresh_segments.interest_metrics		
  WHERE interest_id IS NOT NULL		
  GROUP BY interest_id		
),

cumulative_perc_cte AS (		
  SELECT	
    total_months,
    COUNT(*) AS number_of_ids,
    ROUND(100 * SUM(COUNT(*)) OVER (ORDER BY total_months DESC)/
      SUM(COUNT(*)) OVER (), 2) AS cumulative_perc
  FROM months_cte		
  GROUP BY total_months		
  ORDER BY total_months DESC		
)

SELECT		
  total_months,	
  number_of_ids,		
  cumulative_perc		
FROM cumulative_perc_cte		
WHERE cumulative_perc >=90;
````
Result:
| total_months | number_of_ids | cumulative_perc |
| ------------ | ------------- | --------------- |
| 6            | 33            | 90.85           |
| 5            | 38            | 94.01           |
| 4            | 32            | 96.67           |
| 3            | 15            | 97.92           |
| 2            | 12            | 98.92           |
| 1            | 13            | 100             |
#

# Case Study #3 - Foodie-Fi

<img width="540" height="540" alt="image" src="https://github.com/user-attachments/assets/e9065fc0-a741-4469-8ec4-2bb30d6a37c2" />

## Summary
Foodie-Fi is a subscription-based streaming platform launched in 2020 that provides unlimited access to food-related video content through monthly and annual plans. The business was built with a data-driven approach to support growth and product decisions.

The objective is to analyze subscription data to understand customer behavior, including retention, churn, and plan selection, in order to inform strategic and investment decisions.

**Table 1: plans**

Customers can choose which plans to join Foodie-Fi when they first sign up.
Basic plan customers have limited access and can only stream their videos and is only available monthly at $9.90
Pro plan customers have no watch time limits and are able to download videos for offline viewing. Pro plans start at $19.90 a month or $199 for an annual subscription.
Customers can sign up to an initial 7 day free trial will automatically continue with the pro monthly subscription plan unless they cancel, downgrade to basic or upgrade to an annual pro plan at any point during the trial.
When customers cancel their Foodie-Fi service - they will have a churn plan record with a null price but their plan will continue until the end of the billing period.

**Table 2: subscriptions**

Customer subscriptions show the exact date where their specific `plan_id` starts.
If customers downgrade from a pro plan or cancel their subscription - the higher plan will remain in place until the period is over - the `start_date` in the `subscriptions` table will reflect the date that the actual plan changes.
When customers upgrade their account from a basic plan to a pro or annual pro plan - the higher plan will take effect straightaway.
When customers churn - they will keep their access until the end of their current billing period but the `start_date` will be technically the day they decided to cancel their service.

## Entity Relationship Diagram
<img width="698" height="290" alt="image" src="https://github.com/user-attachments/assets/e4f66189-6639-49e2-8f3f-6454f3b8d9d1" />

## Questions, Queries, and Solutions
All queries executed using PostgreSQL on [DB Fiddle](https://www.db-fiddle.com/f/rHJhRrXy5hbVBNJ6F6b9gJ/16). Follow the link to view the schema and feel free to copy and paste the codes to see them in action.

### A. Customer Journey
Based off the 8 sample customers provided in the sample from the `subscriptions` table, write a brief description about each customer’s onboarding journey.

````sql
SELECT	
  customer_id,
  plans.plan_id,	
  plan_name,	
  start_date	
FROM foodie_fi.subscriptions	
INNER JOIN foodie_fi.plans	
  ON subscriptions.plan_id = plans.plan_id	
WHERE customer_id IN (1, 2, 11, 13, 15, 16, 18, 19)	
ORDER BY customer_id, start_date;	
````
Result:
| customer_id | plan_id | plan_name     | start_date               |
| ----------- | ------- | ------------- | ------------------------ |
| 1           | 0       | trial         | 2020-08-01T00:00:00.000Z |
| 1           | 1       | basic monthly | 2020-08-08T00:00:00.000Z |
| 2           | 0       | trial         | 2020-09-20T00:00:00.000Z |
| 2           | 3       | pro annual    | 2020-09-27T00:00:00.000Z |
| 11          | 0       | trial         | 2020-11-19T00:00:00.000Z |
| 11          | 4       | churn         | 2020-11-26T00:00:00.000Z |
| 13          | 0       | trial         | 2020-12-15T00:00:00.000Z |
| 13          | 1       | basic monthly | 2020-12-22T00:00:00.000Z |
| 13          | 2       | pro monthly   | 2021-03-29T00:00:00.000Z |
| 15          | 0       | trial         | 2020-03-17T00:00:00.000Z |
| 15          | 2       | pro monthly   | 2020-03-24T00:00:00.000Z |
| 15          | 4       | churn         | 2020-04-29T00:00:00.000Z |
| 16          | 0       | trial         | 2020-05-31T00:00:00.000Z |
| 16          | 1       | basic monthly | 2020-06-07T00:00:00.000Z |
| 16          | 3       | pro annual    | 2020-10-21T00:00:00.000Z |
| 18          | 0       | trial         | 2020-07-06T00:00:00.000Z |
| 18          | 2       | pro monthly   | 2020-07-13T00:00:00.000Z |
| 19          | 0       | trial         | 2020-06-22T00:00:00.000Z |
| 19          | 2       | pro monthly   | 2020-06-29T00:00:00.000Z |
| 19          | 3       | pro annual    | 2020-08-29T00:00:00.000Z |

*Analysis: All customers started with a free trial and then upgraded.
Only 1 customer from the sample cancelled their subscription before the trial ended.
1 customer cancelled their pro nonthly subscription.
All customers that didn't cancel, except for 1, ended up on a pro monthly or annual subscription.*
#
### B. Data Analysis Questions

**1. How many customers has Foodie-Fi ever had?**
````sql
SELECT		
  COUNT(DISTINCT customer_id) AS total_customers	
FROM foodie_fi.subscriptions;		
````
Result:
| total_customers |
| --------------- |
| 1000            |

#
**2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value**
````sql
SELECT		
  date_part('month', start_date) AS month,	
  COUNT(plan_id) AS total_trials		
FROM foodie_fi.subscriptions		
WHERE plan_id = 0		
GROUP BY month		
ORDER BY month;		
````
Result:
| month | total_trials |
| ----- | ------------ |
| 1     | 88           |
| 2     | 68           |
| 3     | 94           |
| 4     | 81           |
| 5     | 88           |
| 6     | 79           |
| 7     | 89           |
| 8     | 88           |
| 9     | 87           |
| 10    | 79           |
| 11    | 75           |
| 12    | 84           |
#
**3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name**
````sql
SELECT		
  s.plan_id,	
  p.plan_name,		
  COUNT(s.plan_id) AS count		
FROM foodie_fi.subscriptions s		
INNER JOIN foodie_fi.plans p		
  ON s.plan_id = p.plan_id		
WHERE date_part('year', start_date) > 2020		
GROUP BY s.plan_id, p.plan_name		
ORDER BY s.plan_id;		
````
Result:
| plan_id | plan_name     | count |
| ------- | ------------- | ----- |
| 1       | basic monthly | 8     |
| 2       | pro monthly   | 60    |
| 3       | pro annual    | 63    |
| 4       | churn         | 71    |
#
**4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?**
````sql
SELECT		
  COUNT(plan_id) AS total_churned,	
  ROUND(100.0 * COUNT(plan_id)/(	
    SELECT
      COUNT(distinct customer_id)		
    FROM foodie_fi.subscriptions
  ),1) AS percent_of_customers		
FROM foodie_fi.subscriptions		
WHERE plan_id = 4;		
````		
It is important to include the decimal point on the '100.0' as leaving it off resulted in a rounding error producing the wrong number		
Result:
| total_churned | percent_of_customers |
| ------------- | -------------------- |
| 307           | 30.7                 |
#
**5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?**
````sql
WITH next_plan AS (		
  SELECT		
    customer_id,	
    plan_id,	
    start_date,	
    LEAD(plan_id) OVER(PARTITION BY customer_id	ORDER BY start_date) AS next_plan_id		
  FROM foodie_fi.subscriptions
)		
		
SELECT		
  COUNT(next_plan_id) AS churned_trials,	
  ROUND(100.0 * COUNT(next_plan_id)/(
    SELECT
      COUNT(DISTINCT customer_id)		
    FROM foodie_fi.subscriptions
  )) AS churn_percentage		
FROM next_plan		
WHERE plan_id = 0 AND next_plan_id = 4;		
````
Result:
| churned_trials | churn_percentage |
| -------------- | ---------------- |
| 92             | 9                |
#
**6. What is the number and percentage of customer plans after their initial free trial?**
````sql
WITH next_plan AS (		
  SELECT		
    customer_id,	
    plan_id,	
    start_date,	
    LEAD(plan_id) OVER(PARTITION BY customer_id	ORDER BY start_date) AS next_plan_id		
  FROM foodie_fi.subscriptions		
)
	
SELECT		
  plans.plan_id,	
  plan_name,		
  COUNT(next_plan_id) AS post_trial_plan,		
  ROUND(100.0 * COUNT(next_plan_id)/(
    SELECT
      COUNT(DISTINCT customer_id)		
    FROM foodie_fi.subscriptions
  ),1) AS plan_percentage		
FROM next_plan		
INNER JOIN foodie_fi.plans		
  ON next_plan.next_plan_id = plans.plan_id		
WHERE next_plan.plan_id = 0		
GROUP BY plans.plan_id, plan_name		
ORDER BY plans.plan_id;		
````
Result:
| plan_id | plan_name     | post_trial_plan | plan_percentage |
| ------- | ------------- | --------------- | --------------- |
| 1       | basic monthly | 546             | 54.6            |
| 2       | pro monthly   | 325             | 32.5            |
| 3       | pro annual    | 37              | 3.7             |
| 4       | churn         | 92              | 9.2             |
#
**7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?**
````sql
WITH next_plan AS (		
  SELECT		
    customer_id,	
    plan_id,	
    start_date,	
    LEAD(plan_id) OVER(PARTITION BY customer_id	ORDER BY start_date) AS next_plan_id,		
    LEAD(start_date) OVER(PARTITION BY customer_id	ORDER BY start_date) AS next_start_date		
  FROM foodie_fi.subscriptions		
  WHERE start_date <= '2020-12-31'		
)
	
SELECT		
  p.plan_id,	
  p.plan_name,		
  COUNT(np.plan_id) AS plan_count,		
  ROUND(100.0 * COUNT(np.plan_id)/(
    SELECT
      COUNT(DISTINCT customer_id)		
    FROM next_plan
  ),1) AS plan_percentage		
FROM next_plan AS np		
INNER JOIN foodie_fi.plans AS p		
  ON np.plan_id = p.plan_id		
WHERE np.next_start_date > '2020-12-31' OR np.next_start_date IS NULL		
GROUP BY p.plan_id, p.plan_name		
ORDER BY p.plan_id;		
````
Result:
| plan_id | plan_name     | plan_count | plan_percentage |
| ------- | ------------- | ---------- | --------------- |
| 0       | trial         | 19         | 1.9             |
| 1       | basic monthly | 224        | 22.4            |
| 2       | pro monthly   | 326        | 32.6            |
| 3       | pro annual    | 195        | 19.5            |
| 4       | churn         | 236        | 23.6            |
#
**8. How many customers have upgraded to an annual plan in 2020?**
````sql
SELECT		
  COUNT(DISTINCT customer_id) AS annual_upgrades		
FROM foodie_fi.subscriptions		
WHERE plan_id = 3		
  AND start_date<='2020-12-31';		
````
Result:
| annual_upgrades |
| --------------- |
| 195             |
#
**9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?**
````sql
WITH trial_plans AS (		
  SELECT		
    customer_id,	
    start_date AS trial_date	
  FROM foodie_fi.subscriptions		
  WHERE plan_id = 0		
),		
annual_plans AS (		
  SELECT		
    customer_id,	
    start_date AS annual_date	
  FROM foodie_fi.subscriptions		
  WHERE plan_id = 3		
)		
		
SELECT		
  ROUND(AVG( ap.annual_date - tp.trial_date),0) AS avg_days_to_upgrade	
FROM trial_plans AS tp		
INNER JOIN annual_plans AS ap		
  ON tp.customer_id = ap.customer_id;		
````
Result:
| avg_days_to_upgrade |
| ------------------- |
| 105                 |
#
**10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)**
WITH_BUCKET - drops items into ranges, below the 0 is the lower range, 365 is the upper range, and 12 is the number of buckets:		
WIDTH_BUCKET(annual.annual_date - trial.trial_date, 0, 365, 12) AS avg_days_to_upgrade		
Below, the "||" is used to concentrate multiple parts into a singular string		
````sql	
WITH trial_plans AS (		
  SELECT		
    customer_id,	
    start_date AS trial_date	
  FROM foodie_fi.subscriptions		
  WHERE plan_id = 0		
),		
annual_plans AS (		
  SELECT		
    customer_id,	
    start_date AS annual_date	
  FROM foodie_fi.subscriptions		
  WHERE plan_id = 3		
),		
buckets AS (		
  SELECT		
    WIDTH_BUCKET(annual.annual_date - trial.trial_date, 0, 365, 12) AS avg_days_to_upgrade	
  FROM annual_plans AS annual		
  INNER JOIN trial_plans AS trial		
    ON annual.customer_id = trial.customer_id		
)		
		
SELECT		
  ((avg_days_to_upgrade - 1)*30 || ' - ' || avg_days_to_upgrade*30 || ' days') AS bucket,	
  COUNT(*) AS customer_count		
FROM buckets		
GROUP BY avg_days_to_upgrade		
ORDER BY avg_days_to_upgrade;		
````
Result:
| bucket         | customer_count |
| -------------- | -------------- |
| 0 - 30 days    | 49             |
| 30 - 60 days   | 24             |
| 60 - 90 days   | 35             |
| 90 - 120 days  | 35             |
| 120 - 150 days | 43             |
| 150 - 180 days | 37             |
| 180 - 210 days | 24             |
| 210 - 240 days | 4              |
| 240 - 270 days | 4              |
| 270 - 300 days | 1              |
| 300 - 330 days | 1              |
| 330 - 360 days | 1              |
#
**11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?**
````sql
WITH next_plan AS (		
  SELECT		
    customer_id,	
    plan_id,	
    start_date,	
    LEAD(plan_id) OVER (PARTITION BY customer_id ORDER BY start_date) as next_plan_id	
  FROM foodie_fi.subscriptions		
  WHERE DATE_PART ('year', start_date) = 2020		
)		
		
SELECT		
  COUNT(customer_id) AS downgrade_count	
FROM next_plan		
WHERE plan_id = 2		
  AND next_plan_id = 1;
````
Result:
| downgrade_count |
| --------------- |
| 0               |
#

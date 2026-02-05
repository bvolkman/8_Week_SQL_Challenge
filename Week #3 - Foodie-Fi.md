# Case Study #3 - Foodie-Fi

<img width="540" height="540" alt="image" src="https://github.com/user-attachments/assets/e9065fc0-a741-4469-8ec4-2bb30d6a37c2" />

## Summary
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
All queries executed using PostgreSQL on [DB Fiddle](https://www.db-fiddle.com/f/rHJhRrXy5hbVBNJ6F6b9gJ/16). Follow the link to view the schema.

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

#
**8. How many customers have upgraded to an annual plan in 2020?**
````sql
SELECT		
  COUNT(DISTINCT customer_id) AS annual_upgrades		
FROM foodie_fi.subscriptions		
WHERE plan_id = 3		
  AND start_date<='2020-12-31';		
````		
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

#

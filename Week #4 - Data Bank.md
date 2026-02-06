# Case Study #4 - Data Bank

<img width="540" height="540" alt="image" src="https://github.com/user-attachments/assets/8dfbc2d6-3d69-4712-bb26-ee7d115ce401" />

## Summary
Data Bank is a digital-only neo-bank that combines traditional banking services with secure, distributed data storage linked to customer account balances. As a hybrid of finance and data services, the platform introduces unique challenges in managing customer growth and storage allocation.

The objective is to analyze customer and account data to track growth, calculate key metrics, and forecast future data storage needs, enabling Data Bank to plan infrastructure and scale the business effectively.

**Table 1: Regions**

Contains information about Data Bank’s geographic regions, which are used to organize and allocate customer nodes.

**Table 2: Customer Nodes**

Tracks which node each customer is assigned to, including the region and the time period for each allocation. This table helps analyze customer distribution and node movement over time.

**Table 3: Customer Transactions**

Records all customer banking activity, including deposits, withdrawals, and purchases. This data is used to calculate account balances, customer value, and data storage entitlements.

## Entity Relationship Diagram
<img width="796" height="342" alt="image" src="https://github.com/user-attachments/assets/224fa25a-69d0-4a36-aa6e-3cdda408876f" />

## Questions, Queries, and Solutions
All queries executed using PostgreSQL on [DB Fiddle](https://www.db-fiddle.com/f/2GtQz4wZtuNNu7zXH5HtV4/3). Follow the link to view the schema and feel free to copy and paste the codes to see them in action.

### A. Customer Nodes Exploration

**1. How many unique nodes are there on the Data Bank system?**
````sql
SELECT	
  COUNT(DISTINCT node_id) AS unique_nodes
FROM data_bank.customer_nodes;	
````
Result:
| unique_nodes |
| ------------ |
| 5            |
#
**2. What is the number of nodes per region?**
````sql
SELECT	
  region_name,
  COUNT(DISTINCT node_id) AS nodes	
FROM data_bank.customer_nodes AS c	
INNER JOIN data_bank.regions AS r	
  ON c.region_id = r.region_id	
GROUP BY region_name;	
````
Result:
| region_name | nodes |
| ----------- | ----- |
| Africa      | 5     |
| America     | 5     |
| Asia        | 5     |
| Australia   | 5     |
| Europe      | 5     |
#
**3. How many customers are allocated to each region?**
````sql
SELECT	
  r.region_id,
  region_name,	
  COUNT(DISTINCT customer_id) AS total_customers	
FROM data_bank.customer_nodes AS c	
INNER JOIN data_bank.regions AS r	
  ON c.region_id = r.region_id	
GROUP BY region_name, r.region_id	
ORDER BY r.region_id;	
````
Result:
| region_id | region_name | total_customers |
| --------- | ----------- | --------------- |
| 1         | Australia   | 110             |
| 2         | America     | 105             |
| 3         | Africa      | 102             |
| 4         | Asia        | 95              |
| 5         | Europe      | 88              |	
#
**4. How many days on average are customers reallocated to a different node?**
````sql
WITH dates AS (	
  SELECT	
    customer_id,
    node_id,
    start_date,
    end_date,
    LAG(node_id) OVER (PARTITION BY customer_id ORDER BY start_date) AS prev_node
  FROM data_bank.customer_nodes	
  WHERE EXTRACT(YEAR FROM end_date) != '9999'	
  ORDER BY customer_id, start_date	
)	
	
SELECT	
  ROUND(AVG(end_date - start_date),1) AS avg_days
FROM dates	
WHERE node_id != prev_node;	
````
Result:
| avg_days |
| -------- |
| 14.6     |	
#	
**5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?**
````sql
WITH dates AS (	
  SELECT	
    customer_id,
    region_id,
    node_id,
    start_date,
    end_date,
    LAG(node_id) OVER (PARTITION BY customer_id ORDER BY start_date) AS prev_node
  FROM data_bank.customer_nodes	
  WHERE EXTRACT(YEAR FROM end_date) != '9999'	
  ORDER BY customer_id, start_date	
),	
percentile AS (	
  SELECT	
    regions.region_name,
    PERCENTILE_CONT(0.5) WITHIN GROUP(ORDER BY end_date - start_date) AS "50th_perc",
    PERCENTILE_CONT(0.8) WITHIN GROUP(ORDER BY end_date - start_date) AS "80th_perc",	
    PERCENTILE_CONT(0.95) WITHIN GROUP(ORDER BY end_date - start_date) AS "95th_perc"	
  FROM dates	
  INNER JOIN data_bank.regions	
    ON dates.region_id = regions.region_id	
  WHERE node_id != prev_node	
  GROUP BY regions.region_name	
)
	
SELECT	
  region_name,
  CEIL("50th_perc") AS median,	
  CEIL("80th_perc") AS "80th_percentile",	
  CEIL("95th_perc") AS "95th_percentile"	
FROM percentile;
````
Result:
| region_name | median | 80th_percentile | 95th_percentile |
| ----------- | ------ | --------------- | --------------- |
| Africa      | 15     | 23              | 28              |
| America     | 15     | 23              | 27              |
| Asia        | 14     | 23              | 27              |
| Australia   | 16     | 23              | 28              |
| Europe      | 15     | 24              | 28              |	
#

### B. Customer Transactions

**1. What is the unique count and total amount for each transaction type?**
````sql
SELECT		
  txn_type AS transaction_type,	
  COUNT(customer_id) AS transaction_count,		
  SUM(txn_amount) AS total_transaction		
FROM data_bank.customer_transactions AS ct		
GROUP BY txn_type;
````
Result:
| transaction_type | transaction_count | total_transaction |
| ---------------- | ----------------- | ----------------- |
| purchase         | 1617              | 806537            |
| deposit          | 2671              | 1359168           |
| withdrawal       | 1580              | 793003            |
#
**2. What is the average total historical deposit counts and amounts for all customers?**
````sql
WITH txn_count AS (		
SELECT		
  customer_id,	
  COUNT(txn_type) AS total_deposits,	
  AVG(txn_amount) AS avg_amount	
FROM data_bank.customer_transactions		
WHERE txn_type = 'deposit'		
GROUP BY customer_id		
)		
		
SELECT		
  ROUND(AVG(total_deposits),0) AS avg_deposits,	
  ROUND(AVG(avg_amount),0) AS avg_total_amount		
FROM txn_count;		
````
Result:
| avg_deposits | avg_total_amount |
| ------------ | ---------------- |
| 5            | 509              |
#		
**3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?**
````sql
WITH month_txn AS (		
  SELECT		
    customer_id,	
    SUM(CASE
          WHEN txn_type = 'deposit' THEN 1
          ELSE 0
        END) AS deposit_count,	
    SUM(CASE
          WHEN txn_type = 'purchase' THEN 1
          ELSE 0
        END) AS purchase_count,	
    SUM(CASE
          WHEN txn_type = 'withdrawal' THEN 1
          ELSE 0
        END) AS withdrawal_count,	
    DATE_PART('month', txn_date) AS mnth	
  FROM data_bank.customer_transactions		
  GROUP BY customer_id, DATE_PART('month', txn_date)		
)		
		
SELECT		
  mnth AS month,	
  COUNT(DISTINCT customer_id) AS customer_count	
FROM month_txn		
WHERE deposit_count > 1		
  AND (purchase_count >= 1 OR withdrawal_count >= 1)		
GROUP BY mnth		
ORDER BY mnth;		
````
Result:
| month | customer_count |
| ----- | -------------- |
| 1     | 168            |
| 2     | 181            |
| 3     | 192            |
| 4     | 70             |
#		
**4. What is the closing balance for each customer at the end of the month?**	
*I need to return to this one since I couldn't figure out how to include months with no transactions.*	
````sql
WITH monthly_txn AS (		
  SELECT		
    customer_id,	
    date_part('month', txn_date) AS txn_month,	
    SUM(CASE	
          WHEN txn_type = 'deposit' THEN txn_amount	
          ELSE -txn_amount	
        END) AS total_txn		
  FROM data_bank.customer_transactions		
  GROUP BY customer_id, txn_month		
)		
		
SELECT		
  customer_id,	
  txn_month,		
  total_txn,		
  SUM(total_txn) OVER (PARTITION BY customer_id ORDER BY txn_month) AS closing_balance	
FROM monthly_txn		
WHERE customer_id < 4;		
````
Result:
| customer_id | txn_month | total_txn | closing_balance |
| ----------- | --------- | --------- | --------------- |
| 1           | 1         | 312       | 312             |
| 1           | 3         | \-952     | \-640           |
| 2           | 1         | 549       | 549             |
| 2           | 3         | 61        | 610             |
| 3           | 1         | 144       | 144             |
| 3           | 2         | \-965     | \-821           |
| 3           | 3         | \-401     | \-1222          |
| 3           | 4         | 493       | \-729           |
#	
**5. What is the percentage of customers who increase their closing balance by more than 5%?**
````sql
WITH balances AS (		
  SELECT		
    customer_id,	
    txn_date,	
    CASE		
      WHEN txn_type = 'deposit' THEN txn_amount	
      ELSE -txn_amount	
    END	AS txn_change		
  FROM data_bank.customer_transactions		
  ORDER BY customer_id, txn_date		
),		
		
first_txn AS (		
  SELECT		
    customer_id,	
    MIN(txn_date) AS first_txn_date,	
    MIN(CASE		
          WHEN txn_type = 'deposit' THEN txn_amount	
          ELSE -txn_amount	
        END) AS opening_bal		
  FROM data_bank.customer_transactions		
  GROUP BY customer_id		
),		
		
bal_change AS (		
  SELECT		
    b.customer_id,	
    f.opening_bal,		
    SUM(b.txn_change) AS closing_bal,		
    ROUND(100.0*(SUM(b.txn_change) - f.opening_bal)/
      f.opening_bal, 1) AS perc_change		
  FROM balances as b		
  INNER JOIN first_txn AS f		
    ON b.customer_id = f.customer_id	
  GROUP BY b.customer_id, f.opening_bal		
)		
		
SELECT		
  COUNT(CASE WHEN perc_change > 5 THEN 1 END) AS over_5_perc,	
  COUNT(DISTINCT customer_id) AS total_customers,		
  ROUND(100.0*COUNT(CASE WHEN perc_change > 5 THEN 1 END)/
    COUNT(DISTINCT customer_id), 1) AS percent_of_customers	
FROM bal_change;
````
Result:
| over_5_perc | total_customers | percent_of_customers |
| ----------- | --------------- | -------------------- |
| 199         | 500             | 39.8                 |
#	

# Case Study #7 - Balanced Tree Clothing Co.

<img width="540" height="540" alt="image" src="https://github.com/user-attachments/assets/b8df7195-688c-4cbc-9fa7-1d9b6e70b076" />

## Summary

Balanced Tree Clothing Company prides themselves on providing an optimised range of clothing and lifestyle wear for the modern adventurer! Danny, the CEO of this trendy fashion company has asked you to assist the team’s merchandising teams analyse their sales performance and generate a basic financial report to share with the wider business.

**Table 1: Product Details**

`balanced_tree.product_details` includes all information about the entire range that Balanced Clothing sells in their store.

**Table 2: Product Sales**

`balanced_tree.sales` contains product level information for all the transactions made for Balanced Tree including quantity, price, percentage discount, member status, a transaction ID and also the transaction timestamp.

**Table 3: Product Hierarcy & Product Price**

These tables are used only for the bonus question where we will use them to recreate the `balanced_tree.product_details` table.

## Questions, Queries, and Solutions
All queries executed using PostgreSQL on [DB Fiddle](https://www.db-fiddle.com/f/dkhULDEjGib3K58MvDjYJr/8). Follow the link to view the schema and feel free to copy and paste the codes to see them in action.

### A. High Level Sales Analysis

**1. What was the total quantity sold for all products?**
````sql
SELECT	
  s.prod_id,
  pd.product_name,	
  SUM(s.qty) AS total_sold	
FROM	
balanced_tree.product_details pd	
INNER JOIN balanced_tree.sales s	
  ON pd.product_id = s.prod_id	
GROUP BY s.prod_id, pd.product_name;	
````
Result:
| prod_id | product_name                     | total_sold |
| ------- | -------------------------------- | ---------- |
| 2a2353  | Blue Polo Shirt - Mens           | 3819       |
| 2feb6b  | Pink Fluro Polkadot Socks - Mens | 3770       |
| 5d267b  | White Tee Shirt - Mens           | 3800       |
| 72f5d4  | Indigo Rain Jacket - Womens      | 3757       |
| 9ec847  | Grey Fashion Jacket - Womens     | 3876       |
| b9a74d  | White Striped Socks - Mens       | 3655       |
| c4a632  | Navy Oversized Jeans - Womens    | 3856       |
| c8d436  | Teal Button Up Shirt - Mens      | 3646       |
| d5e9a6  | Khaki Suit Jacket - Womens       | 3752       |
| e31d39  | Cream Relaxed Jeans - Womens     | 3707       |
| e83aa3  | Black Straight Jeans - Womens    | 3786       |
| f084eb  | Navy Solid Socks - Mens          | 3792       |
#
**2. What is the total generated revenue for all products before discounts?**
````sql
SELECT	
  s.prod_id,
  pd.product_name,	
  SUM(s.qty) AS total_sold,	
  pd.price,	
  SUM(s.qty)*pd.price AS total_revenue	
FROM	
balanced_tree.product_details pd	
INNER JOIN balanced_tree.sales s	
  ON pd.product_id = s.prod_id	
GROUP BY s.prod_id, pd.product_name, pd.price;	
````
Result:
| prod_id | product_name                     | total_sold | price | total_revenue |
| ------- | -------------------------------- | ---------- | ----- | ------------- |
| 2a2353  | Blue Polo Shirt - Mens           | 3819       | 57    | 217683        |
| 2feb6b  | Pink Fluro Polkadot Socks - Mens | 3770       | 29    | 109330        |
| 5d267b  | White Tee Shirt - Mens           | 3800       | 40    | 152000        |
| 72f5d4  | Indigo Rain Jacket - Womens      | 3757       | 19    | 71383         |
| 9ec847  | Grey Fashion Jacket - Womens     | 3876       | 54    | 209304        |
| b9a74d  | White Striped Socks - Mens       | 3655       | 17    | 62135         |
| c4a632  | Navy Oversized Jeans - Womens    | 3856       | 13    | 50128         |
| c8d436  | Teal Button Up Shirt - Mens      | 3646       | 10    | 36460         |
| d5e9a6  | Khaki Suit Jacket - Womens       | 3752       | 23    | 86296         |
| e31d39  | Cream Relaxed Jeans - Womens     | 3707       | 10    | 37070         |
| e83aa3  | Black Straight Jeans - Womens    | 3786       | 32    | 121152        |
| f084eb  | Navy Solid Socks - Mens          | 3792       | 36    | 136512        |
#	
**3. What was the total discount amount for all products?**
````sql
SELECT	
  pd.product_name,
  SUM(s.qty * s.price * s.discount/100) AS total_discount	
FROM balanced_tree.sales s	
INNER JOIN balanced_tree.product_details pd	
  ON s.prod_id = pd.product_id	
GROUP BY pd.product_name;
````
Result:
| product_name                     | total_discount |
| -------------------------------- | -------------- |
| White Tee Shirt - Mens           | 17968          |
| Navy Solid Socks - Mens          | 16059          |
| Grey Fashion Jacket - Womens     | 24781          |
| Navy Oversized Jeans - Womens    | 5538           |
| Pink Fluro Polkadot Socks - Mens | 12344          |
| Khaki Suit Jacket - Womens       | 9660           |
| Black Straight Jeans - Womens    | 14156          |
| White Striped Socks - Mens       | 6877           |
| Blue Polo Shirt - Mens           | 26189          |
| Indigo Rain Jacket - Womens      | 8010           |
| Cream Relaxed Jeans - Womens     | 3979           |
| Teal Button Up Shirt - Mens      | 3925           |
#

### B. Transaction Analysis

**1. How many unique transactions were there?**
````sql
SELECT	
  COUNT(DISTINCT txn_id) AS total_txn
FROM balanced_tree.sales;	
````
Result:
| total_txn |
| --------- |
| 2500      |
#
**2. What is the average unique products purchased in each transaction?**
````sql
WITH unique_count AS(	
  SELECT	
    COUNT(DISTINCT prod_id) AS unique_products
  FROM balanced_tree.sales	
  GROUP BY txn_id	
)

SELECT	
  ROUND(AVG(unique_products),0) AS avg_unique_prod
FROM unique_count;	
````
Result:
| avg_unique_prod |
| --------------- |
| 6               |
#	
**3. What are the 25th, 50th and 75th percentile values for the revenue per transaction?**
````sql
WITH revenue_cte AS(	
  SELECT	
    txn_id,
    SUM(qty * price) AS revenue
  FROM balanced_tree.sales	
  GROUP BY txn_id	
)

SELECT	
  PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY revenue) AS perc_25th,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue) AS perc_50th,	
  PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY revenue) AS perc_75th	
FROM revenue_cte;	
````
Result:
| perc_25th | perc_50th | perc_75th |
| --------- | --------- | --------- |
| 375.75    | 509.5     | 647       |
#	
**4. What is the average discount value per transaction?**
````sql
WITH cte_discounts AS (		
  SELECT	
    txn_id,
    SUM(qty * price * discount/100) AS total_discount
  FROM balanced_tree.sales	
  GROUP BY txn_id	
)

SELECT		
  ROUND(AVG(total_discount),2) AS avg_discount	
FROM cte_discounts;		
````
Result:
| avg_discount |
| ------------ |
| 59.79        |
#	
**5. What is the percentage split of all transactions for members vs non-members?**
````sql
SELECT	
  ROUND(100 * COUNT(DISTINCT
    CASE WHEN member = 't' THEN txn_id END)/
    COUNT(DISTINCT txn_id), 2) AS perc_members,
  100 - ROUND(100 * COUNT(
    DISTINCT CASE WHEN member = 't' THEN txn_id END)/
    COUNT(DISTINCT txn_id),2) AS perc_nonmembers	
FROM balanced_tree.sales;	
````
Result:
| perc_members | perc_nonmembers |
| ------------ | --------------- |
| 60           | 40              |
#	
**6. What is the average revenue for member transactions and non-member transactions?**
````sql
WITH member_rev AS (	
  SELECT	
    member,
    txn_id,
    SUM(qty * price) AS revenue
  FROM balanced_tree.sales	
  GROUP BY member, txn_id	
)

SELECT	
  member,
  ROUND(AVG(revenue), 2) AS avg_revenue
FROM member_rev	
GROUP BY member;
````
Result:
| member | avg_revenue |
| ------ | ----------- |
| FALSE  | 515.04      |
| TRUE   | 516.27      |
#

### C. Product Analysis

**1. What are the top 3 products by total revenue before discount?**
````sql
SELECT		
  p.product_id,	
  p.product_name,		
  SUM(s.qty * s.price) AS total_revenue		
FROM balanced_tree.sales s		
INNER JOIN balanced_tree.product_details p		
  ON s.prod_id = p.product_id		
GROUP BY p.product_id, p.product_name		
ORDER by total_revenue DESC		
LIMIT 3;		
````
Result:

#
**2. What is the total quantity, revenue and discount for each segment?**
````sql
SELECT		
  p.segment_id,	
  p.segment_name,		
  SUM(s.qty) AS total_quantity,		
  SUM(s.qty * s.price) AS total_revenue,		
  SUM(s.qty * s.price * s.discount/100) AS total_discount		
FROM balanced_tree.sales s		
INNER JOIN balanced_tree.product_details p		
  ON s.prod_id = p.product_id		
GROUP BY p.segment_id, p.segment_name;		
````
Result:

#
**3. What is the top selling product for each segment?**
````sql
WITH product_total AS (		
  SELECT		
    s.prod_id,	
    p.product_name,	
    p.segment_id,	
    p.segment_name,	
    SUM(s.qty) AS total_qty,	
    RANK () OVER (
      PARTITION BY p.segment_id ORDER BY SUM(s.qty) DESC) AS ranking	
  FROM balanced_tree.sales s		
  INNER JOIN balanced_tree.product_details p		
    ON s.prod_id = p.product_id		
  GROUP BY s.prod_id, p.product_name, p.segment_id, p.segment_name		
)		
		
SELECT		
  segment_name,	
  product_name,		
  total_qty		
FROM product_total		
WHERE ranking = 1;		
````
Result:

#
**4. What is the total quantity, revenue and discount for each category?**
````sql
SELECT		
  p.category_name,	
  p.category_id,		
  SUM(s.qty) AS quantity_sold,		
  SUM(s.qty * s.price) AS total_revenue,		
  SUM(s.qty * s.price * s.discount/100) AS total_discount		
FROM balanced_tree.sales s		
INNER JOIN balanced_tree.product_details p		
  ON s.prod_id = p.product_id		
GROUP BY p.category_name, p.category_id;		
````
Result:

#
**5. What is the top selling product for each category?**
````sql
WITH product_cte AS (		
  SELECT		
    s.prod_id,	
    p.product_name,	
    p.category_name,	
    SUM(s.qty) AS total_sold,	
    RANK () OVER (
      PARTITION BY p.category_name ORDER BY SUM(s.qty) DESC) AS ranking	
  FROM balanced_tree.sales s		
  INNER JOIN balanced_tree.product_details p		
    ON s.prod_id = p.product_id		
  GROUP BY s.prod_id, p.product_name, p.category_name		
)		
		
SELECT		
  category_name,		
  prod_id,		
  product_name,		
  total_sold		
FROM product_cte		
WHERE ranking = 1;		
````
Result:

#
**6. What is the percentage split of revenue by product for each segment?**
````sql
WITH product_rev AS (		
  SELECT		
    p.segment_name,	
    p.product_name,	
    p.product_id,	
    SUM(s.qty * s.price) AS prod_revenue	
  FROM balanced_tree.sales s		
  INNER JOIN balanced_tree.product_details p		
    ON s.prod_id = p.product_id		
  GROUP BY p.segment_name, p.product_name, p.product_id		
)		
		
SELECT		
  segment_name,	
  product_name,		
  ROUND(100 * prod_revenue/SUM(prod_revenue) OVER (
    PARTITION BY segment_name),2) AS perc_revenue		
FROM product_rev		
ORDER BY segment_name, perc_revenue DESC;		
````
Result:


*This question was interpreted as each segment being it's own section and the products within adding up to 100% instead of as a percentage over all segments.*
#		
**7. What is the percentage split of revenue by segment for each category?**
````sql
WITH product_rev AS (
  SELECT		
    p.segment_name,	
    p.category_name,	
    SUM(s.qty * s.price) AS cat_revenue	
  FROM balanced_tree.sales s		
  INNER JOIN balanced_tree.product_details p		
    ON s.prod_id = p.product_id		
  GROUP BY p.segment_name, p.category_name		
)		
		
SELECT		
  category_name,	
  segment_name,		
  ROUND(100 * cat_revenue/SUM(cat_revenue) OVER (
    PARTITION BY category_name),2) AS perc_revenue		
FROM product_rev		
ORDER BY category_name, perc_revenue DESC;		
````
Result:


*This question was interpreted as each category being it's own section and the segments within adding up to 100% instead of as a percentage over all categories.*
#
**8. What is the percentage split of total revenue by category?**
````sql
WITH cat_rev AS (		
  SELECT		
    p.category_name,	
    SUM(s.qty * s.price) AS cat_revenue	
  FROM balanced_tree.sales s		
  INNER JOIN balanced_tree.product_details p		
    ON s.prod_id = p.product_id		
  GROUP BY p.category_name		
)		
		
SELECT		
  category_name,	
  ROUND(100 * cat_revenue/SUM(cat_revenue) OVER (),2) AS perc_revenue		
FROM cat_rev		
ORDER BY perc_revenue DESC;		
````
Result:

#
**9. What is the total transaction “penetration” for each product? (hint: penetration = number of transactions where at least 1 quantity of a product was purchased divided by total number of transactions)**
````sql
WITH txn_cte AS (		
  SELECT		
    p.product_name,	
    p.product_id,	
    s.txn_id,	
    CASE WHEN s.qty > 0 THEN 1 END AS prod_txn	
  FROM balanced_tree.sales s		
  INNER JOIN balanced_tree.product_details p		
    ON s.prod_id = p.product_id		
),

prod_cte AS (		
  SELECT		
    COUNT(DISTINCT txn_id) AS total_txn	
  FROM balanced_tree.sales		
)

SELECT		
  t.product_name,	
  t.product_id,		
  ROUND(100.00 * SUM(t.prod_txn) / p.total_txn,2) AS penetration		
FROM txn_cte t		
CROSS JOIN prod_cte p		
GROUP BY t.product_name, t.product_id, p.total_txn		
ORDER BY penetration DESC;		
````
Result:

#	
**10. What is the most common combination of at least 1 quantity of any 3 products in a 1 single transaction?**

*"SET SEARCH_PATH = schema name;" allows you to not have to reference the schema in the queries later, so this would be: SET SEARCH_PATH = balanced_tree;		
All the join statements create every combo of 3 products across all txns, excluding duplicates (!=) and ensuring a different order is not counted (A, B, C and B, A, C) with the <.*
````sql
SET		
  SEARCH_PATH = balanced_tree;	
SELECT		
  product_1,	
  product_2,		
  product_3,		
  combo_count		
FROM		
  (	
    with product_cte AS (		
      SELECT		
        txn_id,	
        product_name	
      FROM sales AS s	
      JOIN product_details AS pd	
        ON s.prod_id = pd.product_id	
    )		
    SELECT		
      p.product_name AS product_1,	
      p1.product_name AS product_2,	
      p2.product_name AS product_3,	
      COUNT(*) AS combo_count,	
      RANK() OVER(ORDER BY COUNT(*) DESC) AS rank	  
    FROM product_cte AS p		
    JOIN product_cte AS p1		
      ON p.txn_id = p1.txn_id		
      AND p.product_name != p1.product_name		
      AND p.product_name < p1.product_name		
    JOIN product_cte AS p2		
      ON p.txn_id = p2.txn_id		
      AND p.product_name != p2.product_name		
      AND p.product_name < p2.product_name		
      AND p1.product_name != p2.product_name		
      AND p1.product_name < p2.product_name		
    GROUP BY p.product_name, p1.product_name, p2.product_name		
  ) pp -- this is just naming the subquery
WHERE rank = 1;
````
Result:

#

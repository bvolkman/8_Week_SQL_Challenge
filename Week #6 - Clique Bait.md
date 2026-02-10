# Case Study #6 - Clique Bait

<img width="540" height="540" alt="image" src="https://github.com/user-attachments/assets/27734bf7-3f04-4650-9eab-3483a6af32b9" />

## Summary

Clique Bait is not like your regular online seafood store - the founder and CEO Danny, was also a part of a digital data analytics team and wanted to expand his knowledge into the seafood industry!

In this case study - you are required to support Danny’s vision and analyse his dataset and come up with creative solutions to calculate funnel fallout rates for the Clique Bait online store.

**Table 1: Users**

Customers who visit the Clique Bait website are tagged via their `cookie_id`.

**Table 2: Events**

Customer visits are logged in this `events` table at a `cookie_id` level and the `event_type` and `page_id` values can be used to join onto relevant satellite tables to obtain further information about each event. The `sequence_number` is used to order the events within each visit.

**Table 3: Event Identifier**

The `event_identifier` table shows the types of events which are captured by Clique Bait’s digital data systems.

**Table 4: Campaign Identifier**

This table shows information for the 3 campaigns that Clique Bait has ran on their website so far in 2020.

**Table 5: Page Hierarchy**

This table lists all of the pages on the Clique Bait website which are tagged and have data passing through from user interaction events.

## Questions, Queries, and Solutions
All queries executed using PostgreSQL on [DB Fiddle](https://www.db-fiddle.com/f/jmnwogTsUE8hGqkZv9H7E8/17). Follow the link to view the schema and feel free to copy and paste the codes to see them in action.

### 1. Enterprise Relationship Diagram

Create an ERD for all the Clique Bait datasets using this link https://dbdiagram.io/d.
````
Table event_identifier {	
event_type integer	
event_name varchar(13)	
}	
	
Table campaign_identifier {	
campaign_id integer [primary key]	
products varchar(3)	
campaign_name varchar(33)	
start_date timestamp	
end_date timestamp	
}	
	
Table page_hierarchy {	
page_id integer [primary key]	
page_name varchar(14)	
product_category varchar(9)	
product_id integer	
}	
	
Table users {	
user_id integer	
cookie_id varchar(6)	
start_date timestamp	
}	
	
Table events {	
visit_id varchar(6)	
cookie_id varchar(6)	
page_id integer	
event_type integer	
sequence_number integer	
event_time timestamp	
}	
	
Ref: users.cookie_id > events.cookie_id	
	
Ref: events.page_id > page_hierarchy.page_id	
	
Ref: events.event_type > event_identifier.event_type
````
Result:
<img width="1806" height="926" alt="image" src="https://github.com/user-attachments/assets/b67b3315-260c-43c5-861c-668be30fa4ba" />
#

### 2. Digital Analysis

**1. How many users are there?**
````sql
SELECT		
	COUNT(DISTINCT user_id) AS user_count	
FROM clique_bait.users;		
````
Result:
| user_count |
| ---------- |
| 500        |
#		
**2. How many cookies does each user have on average?**
````sql
WITH cookie_count AS (		
	SELECT		
		user_id,	
		COUNT(cookie_id) AS cookie_id_count	
	FROM clique_bait.users		
	GROUP BY user_id		
)		
		
SELECT		
	ROUND(AVG(cookie_id_count),0) AS avg_cookie_count	
FROM cookie_count;
````
Result:
| avg_cookie_count |
| ---------------- |
| 4                |
#		
**3. What is the unique number of visits by all users per month?**
````sql	
SELECT		
	EXTRACT(MONTH FROM event_time) AS month,	
	COUNT(DISTINCT visit_id) AS total_visits	
FROM clique_bait.events		
GROUP BY month		
ORDER BY month;		
````
Result:
| month | total_visits |
| ----- | ------------ |
| 1     | 876          |
| 2     | 1488         |
| 3     | 916          |
| 4     | 248          |
| 5     | 36           |
#
**4. What is the number of events for each event type?**
````sql		
SELECT		
	ev.event_type,	
	evid.event_name,		
	COUNT(ev.event_type) AS event_count			
FROM clique_bait.events ev		
INNER JOIN clique_bait.event_identifier evid		
	ON ev.event_type = evid.event_type		
GROUP BY ev.event_type, evid.event_name		
ORDER BY ev.event_type;		
````
Result:
| event_type | event_name    | event_count |
| ---------- | ------------- | ----------- |
| 1          | Page View     | 20928       |
| 2          | Add to Cart   | 8451        |
| 3          | Purchase      | 1777        |
| 4          | Ad Impression | 876         |
| 5          | Ad Click      | 702         |
#		
**5. What is the percentage of visits which have a purchase event?**
````sql		
SELECT		
	ROUND(100.00 * COUNT(DISTINCT visit_id)/(
		SELECT
			COUNT(DISTINCT visit_id)
		FROM clique_bait.events
	), 2) AS perc_pruchase		
FROM clique_bait.events		
WHERE event_type = 3;		
````
Result:
| perc_pruchase |
| ------------- |
| 49.86         |
#		
**6. What is the percentage of visits which view the checkout page but do not have a purchase event?**
````sql		
WITH checkout_purch AS (		
	SELECT
		visit_id,	
		MAX(CASE
				WHEN event_type = 1 AND page_id = 12 THEN 1 ELSE 0
			END) AS checkout,	
		MAX(CASE
				WHEN event_type = 3 THEN 1 ELSE 0
			END) AS purchase	
	FROM clique_bait.events		
	GROUP BY visit_id		
)		
		
SELECT		
	ROUND(100 * (1-(SUM(purchase)::numeric/SUM(checkout))), 2) AS checkout_no_purch	
FROM checkout_purch;		
````		
Result:
| checkout_no_purch |
| ----------------- |
| 15.5              |
#		
**7. What are the top 3 pages by number of views?**
````sql	
SELECT		
	e.page_id,	
	ph.page_name,		
	COUNT(e.event_type) AS views		
FROM clique_bait.events e		
INNER JOIN clique_bait.page_hierarchy ph		
	ON e.page_id = ph.page_id		
WHERE event_type = 1		
GROUP BY e.page_id, ph.page_name		
ORDER BY views DESC		
LIMIT 3;		
````
Result:
| page_id | page_name    | views |
| ------- | ------------ | ----- |
| 2       | All Products | 3174  |
| 12      | Checkout     | 2103  |
| 1       | Home Page    | 1782  |
#		
**8. What is the number of views and cart adds for each product category?**
````sql	
SELECT		
	ph.product_category,		
	SUM(CASE
			WHEN e.event_type = 1 THEN 1 ELSE 0
		END) AS total_views,		
	SUM(CASE
			WHEN e.event_type = 2 THEN 1 ELSE 0
		END) AS total_adds		
FROM clique_bait.events e		
JOIN clique_bait.page_hierarchy ph		
	ON e.page_id = ph.page_id		
WHERE ph.product_category IS NOT NULL		
GROUP BY ph.product_category		
ORDER BY total_views DESC;		
````
Result:
| product_category | total_views | total_adds |
| ---------------- | ----------- | ---------- |
| Shellfish        | 6204        | 3792       |
| Fish             | 4633        | 2789       |
| Luxury           | 3032        | 1870       |
#		
**9. What are the top 3 products by purchases?**
````sql		
SELECT		
	ph.product_id,	
	ph.product_category,	
	ph.page_name,	
	COUNT(e.visit_id) AS purchases		
FROM clique_bait.events e		
JOIN clique_bait.page_hierarchy ph		
	ON e.page_id = ph.page_id		
WHERE e.event_type = 2		
	AND e.visit_id IN (		
		SELECT
			visit_id
		FROM clique_bait.events	
		WHERE event_type = 3
		)	
GROUP BY ph.product_id, ph.product_category, ph.page_name		
ORDER BY purchases DESC
LIMIT 3;
````
Result:
| product_id | product_category | page_name | purchases |
| ---------- | ---------------- | --------- | --------- |
| 7          | Shellfish        | Lobster   | 754       |
| 9          | Shellfish        | Oyster    | 726       |
| 8          | Shellfish        | Crab      | 719       |
#

### 3. Product Funnel Analysis

Using a single SQL query - create a new output table which has the following details:

* How many times was each product viewed?
* How many times was each product added to cart?
* How many times was each product added to a cart but not purchased (abandoned)?
* How many times was each product purchased?
````sql
WITH product_page_events AS (	
	SELECT	
		e.visit_id,
		ph.product_id,
		ph.page_name AS product_name,
		ph.product_category,
		SUM(CASE
				WHEN e.event_type = 1 THEN 1 ELSE 0
			END) AS page_views,
		SUM(CASE
				WHEN e.event_type = 2 THEN 1 ELSE 0
			END) AS cart_adds
	FROM clique_bait.events e	
	JOIN clique_bait.page_hierarchy ph	
		ON e.page_id = ph.page_id	
	AND product_id IS NOT NULL	
	GROUP BY e.visit_id, ph.product_id, ph.product_category, ph.page_name	
),

purchase_events AS (	
	SELECT	
		DISTINCT visit_id
	FROM clique_bait.events	
	WHERE event_type = 3	
),

combined AS (	
	SELECT	
		ppe.visit_id,
		ppe.product_id,
		ppe.product_name,
		ppe.product_category,
		ppe.page_views,
		ppe.cart_adds,
		CASE
			WHEN pe.visit_id IS NOT NULL THEN 1 ELSE 0
		END AS purchase
	FROM product_page_events AS ppe	
	LEFT JOIN purchase_events AS pe	
		ON ppe.visit_id = pe.visit_id	
),

product_summary AS (	
	SELECT	
		product_id,
		product_name,
		product_category,
		SUM(page_views) AS total_views,
		SUM(cart_adds) AS total_cart_adds,
		SUM(CASE
				WHEN cart_adds = 1 AND purchase = 1 THEN 1 ELSE 0
			END) AS purchased,
		SUM(CASE
				WHEN cart_adds = 1 AND purchase = 0 THEN 1 ELSE 0
			END) AS abandoned
	FROM combined	
	GROUP BY product_id, product_name, product_category	
)

SELECT *	
	FROM product_summary	
ORDER BY product_id;
````
Result:
| product_id | product_name   | product_category | total_views | total_cart_adds | purchased | abandoned |
| ---------- | -------------- | ---------------- | ----------- | --------------- | --------- | --------- |
| 1          | Salmon         | Fish             | 1559        | 938             | 711       | 227       |
| 2          | Kingfish       | Fish             | 1559        | 920             | 707       | 213       |
| 3          | Tuna           | Fish             | 1515        | 931             | 697       | 234       |
| 4          | Russian Caviar | Luxury           | 1563        | 946             | 697       | 249       |
| 5          | Black Truffle  | Luxury           | 1469        | 924             | 707       | 217       |
| 6          | Abalone        | Shellfish        | 1525        | 932             | 699       | 233       |
| 7          | Lobster        | Shellfish        | 1547        | 968             | 754       | 214       |
| 8          | Crab           | Shellfish        | 1564        | 949             | 719       | 230       |
| 9          | Oyster         | Shellfish        | 1568        | 943             | 726       | 217       |
#
Additionally, create another table which further aggregates the data for the above points but this time for each product category instead of individual products.
````sql
WITH product_page_events AS (	
	SELECT	
		e.visit_id,
		ph.product_id,
		ph.page_name AS product_name,
		ph.product_category,
		SUM(CASE
				WHEN e.event_type = 1 THEN 1 ELSE 0
			END) AS page_views,
		SUM(CASE
				WHEN e.event_type = 2 THEN 1 ELSE 0
			END) AS cart_adds
	FROM clique_bait.events e	
	JOIN clique_bait.page_hierarchy ph	
		ON e.page_id = ph.page_id	
	AND product_id IS NOT NULL	
	GROUP BY e.visit_id, ph.product_id, ph.product_category, ph.page_name	
),

purchase_events AS (	
	SELECT	
		DISTINCT visit_id
	FROM clique_bait.events	
	WHERE event_type = 3	
),

combined AS (	
	SELECT	
		ppe.visit_id,
		ppe.product_id,
		ppe.product_name,
		ppe.product_category,
		ppe.page_views,
		ppe.cart_adds,
		CASE
			WHEN pe.visit_id IS NOT NULL THEN 1 ELSE 0
		END AS purchase
	FROM product_page_events AS ppe	
	LEFT JOIN purchase_events AS pe	
		ON ppe.visit_id = pe.visit_id	
),

product_summary AS (	
	SELECT	
		product_category,
		SUM(page_views) AS total_views,
		SUM(cart_adds) AS total_cart_adds,
		SUM(CASE
				WHEN cart_adds = 1 AND purchase = 1 THEN 1 ELSE 0
			END) AS purchased,
		SUM(CASE
				WHEN cart_adds = 1 AND purchase = 0 THEN 1 ELSE 0
			END) AS abandoned
	FROM combined	
	GROUP BY product_category	
)
	
SELECT *	
	FROM product_summary	
ORDER BY product_category;	
````
Result:
| product_category | total_views | total_cart_adds | purchased | abandoned |
| ---------------- | ----------- | --------------- | --------- | --------- |
| Fish             | 4633        | 2789            | 2115      | 674       |
| Luxury           | 3032        | 1870            | 1404      | 466       |
| Shellfish        | 6204        | 3792            | 2898      | 894       |
#
Use your 2 new output tables - answer the following questions:

**1. Which product had the most views, cart adds and purchases?**

| Views          | Cart Adds     | Purchases     |
| -------------- | ------------- | ------------- |
| Oysters (1568) | Lobster (968) | Lobster (754) |

**2. Which product was most likely to be abandoned?** Russian Caviar (249)

**3. Which product had the highest view to purchase percentage?** Lobster (48.74)

*Change the original SELECT statment in the first table to the following:*
````sql
SELECT	
	*,
	ROUND( 100.00 * purchased/total_views, 2) AS viewed_to_purchased	
FROM product_summary	
ORDER BY product_id;
````
Result:
| product_id | product_name   | product_category | total_views | total_cart_adds | purchased | abandoned | viewed_to_purchased |
| ---------- | -------------- | ---------------- | ----------- | --------------- | --------- | --------- | ------------------- |
| 1          | Salmon         | Fish             | 1559        | 938             | 711       | 227       | 45.61               |
| 2          | Kingfish       | Fish             | 1559        | 920             | 707       | 213       | 45.35               |
| 3          | Tuna           | Fish             | 1515        | 931             | 697       | 234       | 46.01               |
| 4          | Russian Caviar | Luxury           | 1563        | 946             | 697       | 249       | 44.59               |
| 5          | Black Truffle  | Luxury           | 1469        | 924             | 707       | 217       | 48.13               |
| 6          | Abalone        | Shellfish        | 1525        | 932             | 699       | 233       | 45.84               |
| 7          | Lobster        | Shellfish        | 1547        | 968             | 754       | 214       | 48.74               |

**4. What is the average conversion rate from view to cart add?** 60.95

**5. What is the average conversion rate from cart add to purchase?** 75.93

*Again, change the SELECT statement to the following:*
````sql
SELECT	
	ROUND( 100.00 * AVG(total_cart_adds/total_views), 2) AS viewed_to_cart,	
	ROUND( 100.00 * AVG(purchased/total_cart_adds), 2) AS cart_to_purchase	
FROM product_summary;
````
Result:
| viewed_to_cart | cart_to_purchase |
| -------------- | ---------------- |
| 60.95          | 75.93            |
#

### 4. Campaigns Analysis

Generate a table that has 1 single row for every unique `visit_id` record and has the following columns:

* `user_id`
* `visit_id`
* `visit_start_time`: the earliest `event_time` for each visit
* `page_views`: count of page views for each visit
* `cart_adds`: count of product cart add events for each visit
* `purchase`: 1/0 flag if a purchase event exists for each visit
* `campaign_name`: map the visit to a campaign if the `visit_start_time` falls between the `start_date` and `end_date`
* `impression`: count of ad impressions for each visit
* `click`: count of ad clicks for each visit
* (Optional column) `cart_products`: a comma separated text value with products added to the cart sorted by the order they were added to the cart (hint: use the `sequence_number`)
````sql
SELECT	
	u.user_id,
	e.visit_id,	
	MIN(e.event_time) AS visit_start_time,	
	SUM(CASE
			WHEN e.event_type = 1 THEN 1 ELSE 0
		END) AS page_views,	
	SUM(CASE
			WHEN e.event_type = 2 THEN 1 ELSE 0
		END) AS cart_adds,	
	SUM(CASE
			WHEN e.event_type = 3 THEN 1 ELSE 0
		END) AS purchase,	
	c.campaign_name,	
	SUM(CASE
			WHEN e.event_type = 4 THEN 1 ELSE 0
		END) AS impression,	
	SUM(CASE
			WHEN e.event_type = 5 THEN 1 ELSE 0
		END) AS clicks,	
	STRING_AGG(
		CASE
			WHEN p.product_id IS NOT NULL AND e.event_type = 2 THEN p.page_name
			ELSE NULL
		END, ', ' ORDER BY e.sequence_number) AS cart_products	
FROM clique_bait.events e	
INNER JOIN clique_bait.users u	
	ON e.cookie_id = u.cookie_id	
LEFT JOIN clique_bait.campaign_identifier c	
	ON e.event_time BETWEEN c.start_date AND c.end_date	
LEFT JOIN clique_bait.page_hierarchy p	
	ON e.page_id = p.page_id	
GROUP BY u.user_id, e.visit_id, c.campaign_name	
ORDER BY u.user_id, e.visit_id	
LIMIT 10;	
````
Result *(Limited to 10 rows)*:
| user_id | visit_id | visit_start_time         | page_views | cart_adds | purchase | campaign_name                     | impression | clicks | cart_products                                                               |
| ------- | -------- | ------------------------ | ---------- | --------- | -------- | --------------------------------- | ---------- | ------ | --------------------------------------------------------------------------- |
| 1       | 02a5d5   | 2020-02-26T16:57:26.260Z | 4          | 0         | 0        | Half Off - Treat Your Shellf(ish) | 0          | 0      | null                                                                        |
| 1       | 0826dc   | 2020-02-26T05:58:37.918Z | 1          | 0         | 0        | Half Off - Treat Your Shellf(ish) | 0          | 0      | null                                                                        |
| 1       | 0fc437   | 2020-02-04T17:49:49.602Z | 10         | 6         | 1        | Half Off - Treat Your Shellf(ish) | 1          | 1      | Tuna, Russian Caviar, Black Truffle, Abalone, Crab, Oyster                  |
| 1       | 30b94d   | 2020-03-15T13:12:54.023Z | 9          | 7         | 1        | Half Off - Treat Your Shellf(ish) | 1          | 1      | Salmon, Kingfish, Tuna, Russian Caviar, Abalone, Lobster, Crab              |
| 1       | 41355d   | 2020-03-25T00:11:17.860Z | 6          | 1         | 0        | Half Off - Treat Your Shellf(ish) | 0          | 0      | Lobster                                                                     |
| 1       | ccf365   | 2020-02-04T19:16:09.182Z | 7          | 3         | 1        | Half Off - Treat Your Shellf(ish) | 0          | 0      | Lobster, Crab, Oyster                                                       |
| 1       | eaffde   | 2020-03-25T20:06:32.342Z | 10         | 8         | 1        | Half Off - Treat Your Shellf(ish) | 1          | 1      | Salmon, Tuna, Russian Caviar, Black Truffle, Abalone, Lobster, Crab, Oyster |
| 1       | f7c798   | 2020-03-15T02:23:26.312Z | 9          | 3         | 1        | Half Off - Treat Your Shellf(ish) | 0          | 0      | Russian Caviar, Crab, Oyster                                                |
| 2       | 0635fb   | 2020-02-16T06:42:42.735Z | 9          | 4         | 1        | Half Off - Treat Your Shellf(ish) | 0          | 0      | Salmon, Kingfish, Abalone, Crab                                             |
| 2       | 1f1198   | 2020-02-01T21:51:55.078Z | 1          | 0         | 0        | Half Off - Treat Your Shellf(ish) | 0          | 0      | null                                                                        |

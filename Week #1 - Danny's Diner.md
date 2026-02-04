# Case Study #1 - Danny's Diner

<img width="540" height="540" alt="image" src="https://github.com/user-attachments/assets/be4faa06-f4d4-4166-9e4d-6f1e1e652c77" />

## Summary and Problem Statement
This week’s challenge is to analyze customer data from Danny’s Diner to solve the problem of understanding customer visit patterns, spending habits, and favorite menu items. The goal is to generate insights that enable better customer personalization and support decisions about expanding the loyalty program.

### Entity Relationship Diagram
<img width="1080" height="440" alt="image" src="https://github.com/user-attachments/assets/9eec5183-128b-49d6-ba08-65133b517935" />

## Questions, Queries, and Solutions
All queries executed using PostgreSQL on [DB Fiddle](https://www.db-fiddle.com/f/2rM8RAnq7h5LLDTzZiRWcd/138). Follow the link to view the schema.

**1. What is the total amount each customer spent at the restaurant?**
````sql  
SELECT		
	sales.customer_id,	
SUM(menu.price) AS total_sales		
FROM dannys_diner.sales		
INNER JOIN dannys_diner.menu		
	ON sales.product_id = menu.product_id		
GROUP BY sales.customer_id
ORDER BY sales.customer_id ASC;
````

Result:
| customer_id | total_sales |
| ----------- | ----------- |
| A           | 76          |
| B           | 74          |
| C           | 36          |

#

**2. How many days has each customer visited the restaurant?**
````sql
SELECT		
	customer_id,	
	COUNT(DISTINCT order_date) AS days_visited		
FROM dannys_diner.sales		
GROUP BY customer_id;
````

Result:
| customer_id | days_visited |
| ----------- | ------------ |
| A           | 4            |
| B           | 6            |
| C           | 2            |

#

**3. What was the first item from the menu purchased by each customer?**
````sql
WITH customer_visits AS (			
	SELECT			
		RANK () OVER (PARTITION BY customer_id ORDER BY order_date ASC) AS visit,			
		customer_id,			
		order_date,			
		product_id			
	FROM dannys_diner.sales			
)
		
SELECT			
	customer_id,		
	product_name			
FROM customer_visits			
INNER JOIN dannys_diner.menu			
	ON menu.product_id = customer_visits.product_id			
WHERE visit = 1			
GROUP BY customer_id, product_name			
ORDER BY customer_id;			
````

Result:
| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

#

**4. What is the most purchased item on the menu and how many times was it purchased by all customers?**
````sql
SELECT	
	menu.product_name,
	COUNT (sales.product_id) AS most_purchased_item	
FROM dannys_diner.sales	
INNER JOIN dannys_diner.menu	
	ON menu.product_id = sales.product_id
GROUP BY menu.product_name	
ORDER BY most_purchased_item DESC	
LIMIT 1;	
````

Result:
| product_name | most_purchased_item |
| ------------ | ------------------- |
| ramen        | 8                   |

#

**5. Which item was the most popular for each customer?**
````sql
WITH customer_favs AS (	
	SELECT	
		sales.customer_id,	
		menu.product_name,	
		COUNT (sales.product_id) AS product_count,	
		RANK () OVER (PARTITION BY sales.customer_id ORDER BY COUNT(sales.product_id) DESC) AS product_rank	
	FROM dannys_diner.sales	
	INNER JOIN dannys_diner.menu	
		ON sales.product_id = menu.product_id	
	GROUP BY sales.customer_id, menu.product_name	
)

SELECT	
	customer_id,
	product_name,	
	product_count	
FROM customer_favs	
WHERE product_rank = 1;	
````

Result:
| customer_id | product_name | product_count |
| ----------- | ------------ | ------------- |
| A           | ramen        | 3             |
| B           | ramen        | 2             |
| B           | curry        | 2             |
| B           | sushi        | 2             |
| C           | ramen        | 3             |

#

**6. Which item was purchased first by the customer after they became a member?**
````sql
With member_orders AS (	
	SELECT	
		sales.customer_id,	
		sales.product_id,	
		ROW_NUMBER () OVER (PARTITION BY sales.customer_id ORDER BY sales.order_date) AS member_first_order	
	FROM dannys_diner.sales	
	INNER JOIN dannys_diner.members	
		ON sales.customer_id = members.customer_id	
	AND sales.order_date > members.join_date	
)

SELECT	
	mo.customer_id,
	m.product_name	
FROM member_orders mo	
INNER JOIN dannys_diner.menu m	
	ON m.product_id = mo.product_id	
WHERE member_first_order = 1	
ORDER BY customer_id ASC;	
````

Result:
| customer_id | product_name |
| ----------- | ------------ |
| A           | ramen        |
| B           | sushi        |

#

**7. Which item was purchased just before the customer became a member?**
````sql
With prior_member_orders AS (	
	SELECT	
		sales.customer_id,	
		sales.product_id,	
		ROW_NUMBER () OVER (PARTITION BY sales.customer_id ORDER BY sales.order_date DESC) AS member_order_rank	
	FROM dannys_diner.sales	
	INNER JOIN dannys_diner.members	
		ON sales.customer_id = members.customer_id	
	AND sales.order_date < members.join_date	
)
	
SELECT	
	pmo.customer_id,
	m.product_name	
FROM prior_member_orders pmo	
INNER JOIN dannys_diner.menu m	
	ON m.product_id = pmo.product_id	
WHERE member_order_rank = 1	
ORDER BY customer_id ASC;	
````

Result:
| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| B           | sushi        |

#

**8. What is the total items and amount spent for each member before they became a member?**
````sql
SELECT	
	sales.customer_id,
	COUNT (sales.product_id) AS total_orders,	
	SUM (menu.price) AS total_spent	
FROM dannys_diner.sales	
INNER JOIN dannys_diner.members	
	ON sales.customer_id = members.customer_id
	AND sales.order_date < members.join_date
INNER JOIN dannys_diner.menu	
	ON sales.product_id = menu.product_id
GROUP BY sales.customer_id	
ORDER BY sales.customer_id ASC;	
````

Result:
| customer_id | total_orders | total_spent |
| ----------- | ------------ | ----------- |
| A           | 2            | 25          |
| B           | 3            | 40          |

#

**9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?**
````sql
WITH product_points AS(	
	SELECT	
		product_id,	
		price,	
		CASE	
			WHEN product_id = 1 THEN price * 20
			ELSE price * 10
			END as points
	FROM dannys_diner.menu	
)

SELECT	
	sales.customer_id,
	SUM (pp.points) AS total_points	
FROM dannys_diner.sales	
INNER JOIN product_points pp	
	ON sales.product_id = pp.product_id	
GROUP BY sales.customer_id	
ORDER BY sales.customer_id ASC;	
````

Result:
| customer_id | total_points |
| ----------- | ------------ |
| A           | 860          |
| B           | 940          |
| C           | 360          |

#

**10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?**
````sql
WITH dates_cte AS(	
	SELECT	
		customer_id,	
		join_date,	
		join_date + 6 AS bonus_end	
	FROM dannys_diner.members	
)	
	
SELECT	
	sales.customer_id,
	SUM (	
		CASE	
			WHEN sales.product_id = 1 THEN menu.price * 20
			WHEN sales.order_date BETWEEN dates_cte.join_date AND dates_cte.bonus_end THEN menu.price * 20
			ELSE menu.price * 10 END) AS point_total	
FROM dannys_diner.sales	
INNER JOIN dannys_diner.menu	
	ON sales.product_id = menu.product_id	
INNER JOIN dates_cte	
	ON dates_cte.customer_id = sales.customer_id	
WHERE EXTRACT (MONTH FROM sales.order_date) = 1	
	AND sales.order_date >= dates_cte.join_date	
GROUP BY sales.customer_id;	
````

Result:
| customer_id | point_total |
| ----------- | ----------- |
| B           | 320         |
| A           | 1020        |

#

**Bonus Question - Join All The Things**
The following questions are related creating basic data tables that Danny and his team can use to quickly derive insights without needing to join the underlying tables using SQL.
Recreate the following table output using the available data:
| customer_id | order_date | product_name | price | member |
| ----------- | ---------- | ------------ | ----- | ------ |
| A           | 2021-01-01 | curry        | 15    | N      |
| A           | 2021-01-01 | sushi        | 10    | N      |
| A           | 2021-01-07 | curry        | 15    | Y      |
| A           | 2021-01-10 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| B           | 2021-01-01 | curry        | 15    | N      |
| B           | 2021-01-02 | curry        | 15    | N      |
| B           | 2021-01-04 | sushi        | 10    | N      |
| B           | 2021-01-11 | sushi        | 10    | Y      |
| B           | 2021-01-16 | ramen        | 12    | Y      |
| B           | 2021-02-01 | ramen        | 12    | Y      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-07 | ramen        | 12    | N      |

Query:
````sql
SELECT	
	sales.customer_id,
	sales.order_date,	
	menu.product_name,	
	menu.price,	
	CASE	
		WHEN sales.order_date < members.join_date THEN 'N'
		WHEN sales.order_date >= members.join_date THEN 'Y'
		ELSE 'N'
	END AS member	
FROM dannys_diner.sales	
INNER JOIN dannys_diner.menu	
	ON sales.product_id = menu.product_id	
LEFT JOIN dannys_diner.members	
	ON members.customer_id = sales.customer_id	
ORDER BY sales.customer_id, sales.order_date ASC;	
````

Result:
| customer_id | order_date               | product_name | price | member |
| ----------- | ------------------------ | ------------ | ----- | ------ |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      |
| A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      |
| A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      |
| B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |
| C           | 2021-01-07T00:00:00.000Z | ramen        | 12    | N      |

#

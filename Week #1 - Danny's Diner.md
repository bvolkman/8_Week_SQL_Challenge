# Case Study #1 - Danny's Diner

<img width="540" height="540" alt="image" src="https://github.com/user-attachments/assets/be4faa06-f4d4-4166-9e4d-6f1e1e652c77" />

## Summary and Problem Statement
This week’s challenge is to analyze customer data from Danny’s Diner to solve the problem of understanding customer visit patterns, spending habits, and favorite menu items. The goal is to generate insights that enable better customer personalization and support decisions about expanding the loyalty program.

### Entity Relationship Diagram
<img width="1080" height="440" alt="image" src="https://github.com/user-attachments/assets/9eec5183-128b-49d6-ba08-65133b517935" />

## Questions, Queries, and Solutions
All queries executed using PostgreSQL on [DB Fiddle](https://www.db-fiddle.com/f/2rM8RAnq7h5LLDTzZiRWcd/138). Also, follow the linnk to view the schema.

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


**2. How many days has each customer visited the restaurant?**

**3. What was the first item from the menu purchased by each customer?**

**4. What is the most purchased item on the menu and how many times was it purchased by all customers?**

**5. Which item was the most popular for each customer?**

**6. Which item was purchased first by the customer after they became a member?**

**7. Which item was purchased just before the customer became a member?**

**8. What is the total items and amount spent for each member before they became a member?**

**9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?**

**10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?**

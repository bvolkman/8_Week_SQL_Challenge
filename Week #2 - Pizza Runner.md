# Case Study #2 - Pizza Runner

<img width="540" height="540" alt="image" src="https://github.com/user-attachments/assets/06efa8cd-6af7-41dc-bad3-3b933d4a9ac8" />

## Summary
**Table 1: runners** 

The `runners` table shows the `registration_date` for each new runner.

**Table 2: customer_orders** 

Customer pizza orders are captured in the `customer_orders` table with 1 row for each individual pizza that is part of the order.
The `pizza_id` relates to the type of pizza which was ordered whilst the `exclusions` are the `ingredient_id` values which should be removed from the pizza and the `extras` are the `ingredient_id` values which need to be added to the pizza.
Note that customers can order multiple pizzas in a single order with varying `exclusions` and `extras` values even if the pizza is the same type!
The `exclusions` and `extras` columns will need to be cleaned up before using them in your queries.

**Table 3: runner_orders** 

After each orders are received through the system - they are assigned to a runner - however not all orders are fully completed and can be cancelled by the restaurant or the customer.
The `pickup_time` is the timestamp at which the runner arrives at the Pizza Runner headquarters to pick up the freshly cooked pizzas. The `distance` and `duration` fields are related to how far and long the runner had to travel to deliver the order to the respective customer.
There are some known data issues with this table so be careful when using this in your queries - make sure to check the data types for each column in the schema SQL!

**Table 4: pizza_names**

At the moment - Pizza Runner only has 2 pizzas available the Meat Lovers or Vegetarian!

**Table 5: pizza_recipes**

Each `pizza_id` has a standard set of `toppings` which are used as part of the pizza recipe.

**Table 6: pizza_toppings**

This table contains all of the `topping_name` values with their corresponding `topping_id` value.

## Entity Relationship Diagram
<img width="1752" height="904" alt="image" src="https://github.com/user-attachments/assets/44b92d39-15fa-4f4d-80b1-41bf9570fcd5" />

## Questions, Queries, and Solutions
All queries executed using PostgreSQL on [DB Fiddle](https://www.db-fiddle.com/f/7VcQKQwsS3CTkGRFG7vu98/65). Follow the link to view the schema and feel free to copy and paste the codes to see them in action.

### A. Pizza Metrics

**1. How many pizzas were ordered?**
````sql
SELECT	
	COUNT(*) AS total_orders
FROM pizza_runner.customer_orders;
````

Result:
| total_orders |
| ------------ |
| 14           |

#

**2. How many unique customer orders were made?**
````sql
SELECT			
	COUNT (DISTINCT order_id) AS unique_orders		
FROM pizza_runner.customer_orders;		
````

Result:
| unique_orders |
| ------------- |
| 10            |

#

**3. How many successful orders were delivered by each runner?**
````sql
SELECT			
  runner_id,		
  COUNT (order_id) AS successful_orders			
FROM pizza_runner.runner_orders			
WHERE cancellation IS NULL			
  OR cancellation NOT IN ('Restaurant Cancellation', 'Customer Cancellation')			
GROUP BY runner_id			
ORDER BY runner_id;			
````

Result:
| runner_id | successful_orders |
| --------- | ----------------- |
| 1         | 4                 |
| 2         | 3                 |
| 3         | 1                 |

#

**4. How many of each type of pizza was delivered?**
````sql
SELECT			
  pizza_names.pizza_name,			
  COUNT (customer_orders.pizza_id) AS pizza_type_total			
FROM pizza_runner.customer_orders			
INNER JOIN pizza_runner.pizza_names			
  ON customer_orders.pizza_id = pizza_names.pizza_id			
INNER JOIN pizza_runner.runner_orders			
  ON runner_orders.order_id = customer_orders.order_id			
WHERE cancellation IS NULL			
  OR cancellation NOT IN ('Restaurant Cancellation', 'Customer Cancellation')			
GROUP BY pizza_name;
````

Result:
| pizza_name | pizza_type_total |
| ---------- | ---------------- |
| Meatlovers | 9                |
| Vegetarian | 3                |

#

**5. How many Vegetarian and Meatlovers were ordered by each customer?**
*First we need to clean up the `customer_orders` and `runner_orders` tables. We need consistency with the NULL values. Currently in both tables you will see values of NULL, 'null', and ''.*
		
Cleaning the `customer_orders` table:
````sql
DROP TABLE IF EXISTS updated_customer_orders;			
CREATE TEMP TABLE updated_customer_orders AS (			
  SELECT			
    order_id,		
    customer_id,			
    pizza_id,			
    CASE			
      WHEN exclusions IS NULL		
        OR exclusions LIKE 'null' THEN ''			
      ELSE exclusions			
    END AS exclusions,			
    CASE
      WHEN extras IS NULL
        OR extras LIKE 'null' THEN ''			
      ELSE extras			
    END AS extras,			
    order_time			
  FROM pizza_runner.customer_orders			
);			
			
SELECT * FROM updated_customer_orders;			
````
Updated table:
| order_id | customer_id | pizza_id | exclusions | extras | order_time               |
| -------- | ----------- | -------- | ---------- | ------ | ------------------------ |
| 1        | 101         | 1        |            |        | 2020-01-01T18:05:02.000Z |
| 2        | 101         | 1        |            |        | 2020-01-01T19:00:52.000Z |
| 3        | 102         | 1        |            |        | 2020-01-02T23:51:23.000Z |
| 3        | 102         | 2        |            |        | 2020-01-02T23:51:23.000Z |
| 4        | 103         | 1        | 4          |        | 2020-01-04T13:23:46.000Z |
| 4        | 103         | 1        | 4          |        | 2020-01-04T13:23:46.000Z |
| 4        | 103         | 2        | 4          |        | 2020-01-04T13:23:46.000Z |
| 5        | 104         | 1        |            | 1      | 2020-01-08T21:00:29.000Z |
| 6        | 101         | 2        |            |        | 2020-01-08T21:03:13.000Z |
| 7        | 105         | 2        |            | 1      | 2020-01-08T21:20:29.000Z |
| 8        | 102         | 1        |            |        | 2020-01-09T23:54:33.000Z |
| 9        | 103         | 1        | 4          | 1, 5   | 2020-01-10T11:22:59.000Z |
| 10       | 104         | 1        |            |        | 2020-01-11T18:34:49.000Z |
| 10       | 104         | 1        | 2, 6       | 1, 4   | 2020-01-11T18:34:49.000Z |


Cleaning the `runner_orders` table:
````sql
DROP TABLE IF EXISTS updated_runner_orders;			
CREATE TEMP TABLE updated_runner_orders AS (			
  SELECT
    order_id,		
    runner_id,		
    CASE		
      WHEN pickup_time LIKE 'null' THEN null	
      ELSE pickup_time	
    END AS pickup_time,	
    NULLIF(regexp_replace(distance, '[^0-9.]','','g'), '')::numeric AS distance,		
    NULLIF(regexp_replace(duration, '[^0-9.]','','g'), '')::numeric AS duration,		
    CASE		
      WHEN cancellation IN ('null', 'NaN', '') THEN ''	
      WHEN cancellation IS NULL THEN ''	
    ELSE cancellation	
    END AS cancellation		
  FROM pizza_runner.runner_orders
);
			
SELECT * FROM updated_runner_orders;			
````

Update table:
| order_id | runner_id | pickup_time         | distance | duration | cancellation            |
| -------- | --------- | ------------------- | -------- | -------- | ----------------------- |
| 1        | 1         | 2020-01-01 18:15:34 | 20       | 32       |                         |
| 2        | 1         | 2020-01-01 19:10:54 | 20       | 27       |                         |
| 3        | 1         | 2020-01-03 0:12:37  | 13.4     | 20       |                         |
| 4        | 2         | 2020-01-04 13:53:03 | 23.4     | 40       |                         |
| 5        | 3         | 2020-01-08 21:10:57 | 10       | 15       |                         |
| 6        | 3         | null                | null     | null     | Restaurant Cancellation |
| 7        | 2         | 2020-01-08 21:30:45 | 25       | 25       |                         |
| 8        | 2         | 2020-01-10 0:15:02  | 23.4     | 15       |                         |
| 9        | 2         | null                | null     | null     | Customer Cancellation   |
| 10       | 1         | 2020-01-11 18:50:20 | 10       | 10       |                         |

Now that we have clean, consistent data we can answer the question.
````sql
SELECT			
  customer_id,		
  COUNT(CASE WHEN pizza_id = 1 THEN 1 END) AS meat_lovers,			
  COUNT(CASE WHEN pizza_id = 2 THEN 1 END) AS vegetarian			
FROM updated_customer_orders			
GROUP BY customer_id;			
````

Result:
| customer_id | meat_lovers | vegetarian |
| ----------- | ----------- | ---------- |
| 101         | 2           | 1          |
| 103         | 3           | 1          |
| 104         | 3           | 0          |
| 105         | 0           | 1          |
| 102         | 2           | 1          |

#

**6. What was the maximum number of pizzas delivered in a single order?**
````sql
SELECT
  MAX(order_count) AS max_count			
FROM (			
  SELECT			
    customer_orders.order_id,		
    COUNT(customer_orders.order_id) AS order_count			
  FROM pizza_runner.customer_orders			
  INNER JOIN updated_runner_orders			
    ON customer_orders.order_id = updated_runner_orders.order_id			
  WHERE updated_runner_orders.cancellation = ''			
  GROUP BY customer_orders.order_id
  ) AS pizza_count;
````

Result:
| max_count |
| --------- |
| 3         |

#
			
**7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?**
````sql
SELECT			
  co.customer_id,		
  SUM(CASE			
        WHEN co.exclusions = '' AND co.extras = '' THEN 1
        ELSE 0
      END) AS no_changes,		
  SUM(CASE			
        WHEN co.exclusions = '' AND co.extras = '' THEN 0
        ELSE 1
      END) AS pizza_changes		
FROM updated_customer_orders AS co			
INNER JOIN updated_runner_orders AS ro			
  ON co.order_id = ro.order_id			
WHERE ro.cancellation = ''			
GROUP BY co.customer_id			
ORDER BY co.customer_id;
````

Result:
| customer_id | no_changes | pizza_changes |
| ----------- | ---------- | ------------- |
| 101         | 2          | 0             |
| 102         | 3          | 0             |
| 103         | 0          | 3             |
| 104         | 1          | 2             |
| 105         | 0          | 1             |

#
			
**8. How many pizzas were delivered that had both exclusions and extras?**
````sql
SELECT			
  SUM(CASE		
        WHEN co.exclusions = '' OR co.extras = '' THEN 0
        ELSE 1
      END) AS extras_and_exclusions		
FROM updated_customer_orders AS co			
INNER JOIN updated_runner_orders AS ro			
  ON co.order_id = ro.order_id			
WHERE ro.cancellation = '';			
````

Result:
| extras_and_exclusions |
| --------------------- |
| 1                     |

#

**9. What was the total volume of pizzas ordered for each hour of the day?**
````sql
SELECT			
  DATE_PART('hour', order_time::TIMESTAMP) AS hour,
  COUNT(*) AS pizza_count			
FROM updated_customer_orders			
GROUP BY hour			
ORDER BY hour;			
````

Result:
| hour | pizza_count |
| ---- | ----------- |
| 11   | 1           |
| 13   | 3           |
| 18   | 3           |
| 19   | 1           |
| 21   | 3           |
| 23   | 3           |

#
			
**10. What was the volume of orders for each day of the week?**
````sql
SELECT			
  TO_CHAR(order_time, 'Day') AS day,		
  COUNT (*) AS pizza_count			
FROM updated_customer_orders			
GROUP BY day			
ORDER BY day;
````
Result:
| day       | pizza_count |
| --------- | ----------- |
| Friday    | 1           |
| Saturday  | 5           |
| Thursday  | 3           |
| Wednesday | 5           |

#

### B. Runner and Customer Experience

**1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)**
````sql
WITH runner_signups AS (		
  SELECT		
    runner_id,	
    registration_date,	
    registration_date - ((registration_date - '2021-01-01') % 7) AS start_of_week	
  FROM pizza_runner.runners		
)
		
SELECT		
  start_of_week,	
  COUNT(runner_id) AS signups		
FROM runner_signups		
GROUP BY start_of_week		
ORDER BY start_of_week;		
````
Result:
| start_of_week            | signups |
| ------------------------ | ------- |
| 2021-01-01T00:00:00.000Z | 2       |
| 2021-01-08T00:00:00.000Z | 1       |
| 2021-01-15T00:00:00.000Z | 1       |

#

**2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?**
````sql
WITH runner_pickups AS (		
  SELECT		
    ro.runner_id,	
    ro.order_id,	
    co.order_time,	
    ro.pickup_time,	
    (pickup_time - order_time) AS time_to_pickup	
  FROM updated_runner_orders AS ro		
  INNER JOIN updated_customer_orders AS co		
    ON ro.order_id = co.order_id		
  WHERE ro.pickup_time IS NOT null		
)
		
SELECT		
  runner_id,	
  date_part('minutes', AVG(time_to_pickup)) AS avg_runner_arrival		
FROM runner_pickups		
GROUP BY runner_id		
ORDER BY runner_id;		
````
Result:
| runner_id | avg_runner_arrival |
| --------- | ------------------ |
| 1         | 15                 |
| 2         | 23                 |
| 3         | 10                 |

#
**3. Is there any relationship between the number of pizzas and how long the order takes to prepare?**
````sql
WITH order_count AS (		
  SELECT		
    order_id,	
    order_time,	
    COUNT(pizza_id) AS pizza_order_count	
  FROM updated_customer_orders		
  GROUP BY order_id, order_time		
),		
prep_time AS (		
  SELECT		
    oc.pizza_order_count,	
    (ro.pickup_time - oc.order_time) AS time_to_pickup	
  FROM order_count AS oc		
  INNER JOIN updated_runner_orders AS ro		
    ON oc.order_id = ro.order_id		
  WHERE ro.cancellation IS NOT NULL		
)
	
SELECT		
  pizza_order_count,	
  date_part('minutes',AVG(time_to_pickup)) AS avg_order_time		
FROM prep_time		
GROUP BY pizza_order_count		
ORDER BY pizza_order_count;		
````
Result:
| pizza_order_count | avg_order_time |
| ----------------- | -------------- |
| 1                 | 12             |
| 2                 | 18             |
| 3                 | 29             |

#
**4. What was the average distance travelled for each runner?**
````sql
SELECT		
  runner_id,	
  ROUND(AVG(distance),2) AS avg_distance		
FROM updated_runner_orders		
GROUP BY runner_id		
ORDER BY runner_id;		
````
Result:
| runner_id | avg_distance |
| --------- | ------------ |
| 1         | 15.85        |
| 2         | 23.93        |
| 3         | 10           |

#
**5. What was the difference between the longest and shortest delivery times for all orders?**
````sql
SELECT		
  MIN(duration),	
  MAX(duration),		
  MAX(duration) - MIN(duration) as delivery_range		
FROM updated_runner_orders;		
		
/*Or if delivery time is defined as from the order time, not just the duration, I would use the following query:*/		
		
WITH delivery AS (		
  SELECT		
    co.order_id,	
    date_part('minutes', (ro.pickup_time - co.order_time)) + ro.duration AS delivery_time	
  FROM updated_customer_orders AS co		
  INNER JOIN updated_runner_orders AS ro		
    ON co.order_id = ro.order_id		
)

SELECT		
  MIN(delivery_time),	
  MAX(delivery_time),		
  MAX(delivery_time) - MIN(delivery_time) AS delivery_range		
FROM delivery;		
````
Result for query 1:
| min | max | delivery_range |
| --- | --- | -------------- |
| 10  | 40  | 30             |

Result for query 2 (starting from the order time):
| min | max | delivery_range |
| --- | --- | -------------- |
| 25  | 69  | 44             |

#
**6. What was the average speed for each runner for each delivery and do you notice any trend for these values?**
````sql
WITH runner_speed AS (		
  SELECT
    runner_id,	
    order_id,	
    60/duration*distance AS speed	
  FROM updated_runner_orders		
  WHERE cancellation IS NOT NULL		
)
		
SELECT		
  runner_id,	
  ROUND(AVG(speed),2) AS avg_speed		
FROM runner_speed		
GROUP BY runner_id		
ORDER BY runner_id;		
````
Result:
| runner_id | avg_speed |
| --------- | --------- |
| 1         | 45.54     |
| 2         | 62.9      |
| 3         | 40        |

#
**7. What is the successful delivery percentage for each runner?**
````sql
SELECT		
  runner_id,	
  ROUND(100 * COUNT(pickup_time) / COUNT(order_id),2) AS delivery_rate		
FROM updated_runner_orders		
GROUP BY runner_id		
ORDER BY runner_id;		
````
Result:
| runner_id | delivery_rate |
| --------- | ------------- |
| 1         | 100           |
| 2         | 75            |
| 3         | 50            |

#

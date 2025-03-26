# SQL Project: Data Analysis for Zomato - A Food Delivery Company

## Project Overview

This project showcases SQL problem-solving skills through the analysis of Zomato's data, a leading food delivery company in India. It involves setting up a structured database, importing data, and applying complex SQL queries to address key business challenges and extract valuable insights.

## Project Structure

- **Database Setup:** Creation of the `zomato_db` database and the required tables.
- **Data Import:** Inserting sample data into the tables.
- **Business Problems:** Solving 20 specific business problems using SQL queries.

## ER Diagram

![ERD](https://github.com/kkhannah/Zomato-SQL-Project/blob/main/ER%20Diagram/ER%20diagram.png)

## Database Setup and Data Import

```sql

CREATE DATABASE zomato_db;

```

```sql

DROP TABLE IF EXISTS deliveries;
DROP TABLE IF EXISTS Orders;
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS restaurants;
DROP TABLE IF EXISTS riders;

-- Creating Tables

CREATE TABLE restaurants (
    restaurant_id SERIAL PRIMARY KEY,
    restaurant_name VARCHAR(100) NOT NULL,
    city VARCHAR(50),
    opening_hours VARCHAR(50)
);


CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    reg_date DATE
);


CREATE TABLE riders (
    rider_id SERIAL PRIMARY KEY,
    rider_name VARCHAR(100) NOT NULL,
    sign_up DATE
);


CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    restaurant_id INT,
    order_item VARCHAR(255),
    order_date DATE NOT NULL,
    order_time TIME NOT NULL,
    order_status VARCHAR(20) DEFAULT 'Pending',
    total_amount DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (restaurant_id) REFERENCES restaurants(restaurant_id)
);


CREATE TABLE deliveries (
    delivery_id SERIAL PRIMARY KEY,
    order_id INT,
    delivery_status VARCHAR(20) DEFAULT 'Pending',
    delivery_time TIME,
    rider_id INT,
    FOREIGN KEY (order_id) REFERENCES Orders(order_id),
    FOREIGN KEY (rider_id) REFERENCES riders(rider_id)
);

-- Data Import Hierarcy

-- First import to customers;
-- 2nd Import to restaurants;
-- 3rd Import to orders;
-- 4th Import to riders;
-- 5th Import to deliveries;

```

## Business Problems

### 1. Write a query to find the top 5 most frequently ordered dishes by customer called "Arjun Mehta" in the last 1 year.

```sql

WITH dish_cte AS

	(SELECT 
		c.customer_id,
		c.customer_name,
		o.order_item as dishes,
		COUNT(*) as total_orders,
		DENSE_RANK() OVER (ORDER BY COUNT(*) DESC) as rank
	FROM orders as o
	JOIN
	customers as c
	ON c.customer_id = o.customer_id
	WHERE 
		o.order_date >= CURRENT_DATE - INTERVAL '1 Year'
		AND 
		c.customer_name = 'Arjun Mehta'
	GROUP BY c.customer_id, c.customer_name, dishes)
	
SELECT 
	customer_name,
	dishes,
	total_orders
FROM dish_cte
WHERE rank <= 5;

```

### 2. Popular Time Slots
-- Identify the time slots during which the most orders are placed, based on 2-hour intervals.

**Approach 1:**

```sql

-- Approach 1

SELECT
    CASE
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 0 AND 1 THEN '00:00 - 02:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 2 AND 3 THEN '02:00 - 04:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 4 AND 5 THEN '04:00 - 06:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 6 AND 7 THEN '06:00 - 08:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 8 AND 9 THEN '08:00 - 10:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 10 AND 11 THEN '10:00 - 12:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 12 AND 13 THEN '12:00 - 14:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 14 AND 15 THEN '14:00 - 16:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 16 AND 17 THEN '16:00 - 18:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 18 AND 19 THEN '18:00 - 20:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 20 AND 21 THEN '20:00 - 22:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 22 AND 23 THEN '22:00 - 00:00'
    END AS time_slot,
    COUNT(order_id) AS order_count
FROM Orders
GROUP BY time_slot
ORDER BY order_count DESC;

```

**Approach 2:**

```sql

SELECT 
	FLOOR(EXTRACT(HOUR FROM order_time)/2)*2 as start_time,
	FLOOR(EXTRACT(HOUR FROM order_time)/2)*2 + 2 as end_time,
	COUNT(*) as total_orders
FROM orders
GROUP BY start_time, end_time
ORDER BY total_orders DESC;

```

### 3. Order Value Analysis
-- Find the average order value per customer who has placed more than 750 orders.
-- Return customer_name, and aov(average order value).

```sql

SELECT 
	c.customer_name,
	ROUND(AVG(o.total_amount),2) as aov
FROM orders as o
	JOIN customers as c
	ON c.customer_id = o.customer_id
GROUP BY o.customer_id,c.customer_name
HAVING  COUNT(*) > 750;

```

### 4. High-Value Customers
-- List the customers who have spent more than 100K in total on food orders.
-- return customer_name and customer_id.

```sql

SELECT
	c.customer_id,
	c.customer_name,
	SUM(o.total_amount) as total_sum
FROM orders as o
	JOIN customers as c
	ON c.customer_id = o.customer_id
GROUP BY c.customer_id,c.customer_name
HAVING  SUM(o.total_amount)> 100000
ORDER BY total_sum DESC;

```

### 5. Orders Without Delivery
-- Write a query to find orders that were placed but not delivered.
-- Return each restuarant name and number of non-delivered orders.

```sql

SELECT 
	r.restaurant_name,
	COUNT(o.order_id) as cnt_not_delivered_orders
FROM orders as o
LEFT JOIN 
restaurants as r
ON r.restaurant_id = o.restaurant_id
LEFT JOIN
deliveries as d
ON d.order_id = o.order_id
WHERE d.delivery_status <> 'Delivered' OR d.delivery_id IS NULL
GROUP BY r.restaurant_name
ORDER BY cnt_not_delivered_orders DESC;

```


### 6. Restaurant Revenue Ranking: 
-- Rank restaurants by their total revenue from the last year, including their name, 
-- total revenue, and rank within their city.

```sql

SELECT 
	r.city,
	r.restaurant_name,
	SUM(o.total_amount) as revenue,
	DENSE_RANK() OVER(PARTITION BY r.city ORDER BY SUM(o.total_amount) DESC) as rank
FROM orders as o
JOIN 
restaurants as r
ON r.restaurant_id = o.restaurant_id
WHERE o.order_date >= CURRENT_DATE - INTERVAL '1 year'
GROUP BY r.city, r.restaurant_name;

```

### 7. Most Popular Dish by City: 
-- Identify the most popular dish in each city based on the number of orders.

```sql

WITH dish_cte 
AS
(
	SELECT 
		r.city,
		o.order_item as dish,
		COUNT(order_id) as total_orders,
		RANK() OVER(PARTITION BY r.city ORDER BY COUNT(order_id) DESC) as rank
	FROM orders as o
	JOIN 
	restaurants as r
	ON r.restaurant_id = o.restaurant_id
	GROUP BY r.city, dish
) 
SELECT
	city, dish, total_orders
FROM dish_cte
WHERE rank=1;

```

### 8. Customer Churn: 
-- Find customers who haven’t placed an order in 2024 but did in 2023.

```sql

SELECT 
	DISTINCT customer_id 
FROM orders
WHERE EXTRACT(YEAR FROM order_date) = 2023
	AND
customer_id NOT IN (SELECT DISTINCT customer_id FROM orders
					WHERE EXTRACT(YEAR FROM order_date) = 2024);
```

### 9. Cancellation Rate Comparison: 
-- Calculate and compare the order cancellation rate for each restaurant between the 
-- current year and the previous year.

```sql

WITH cancel_ratio_23 AS (
    SELECT 
        o.restaurant_id,
        COUNT(o.order_id) AS total_orders,
        COUNT(CASE WHEN d.delivery_id IS NULL OR d.delivery_status <> 'Delivered' THEN 1 END) AS not_delivered
    FROM orders AS o
    LEFT JOIN deliveries AS d
    ON o.order_id = d.order_id
    WHERE EXTRACT(YEAR FROM o.order_date) = 2023
    GROUP BY o.restaurant_id
),
cancel_ratio_24 AS (
    SELECT 
        o.restaurant_id,
        COUNT(o.order_id) AS total_orders,
        COUNT(CASE WHEN d.delivery_id IS NULL OR d.delivery_status <> 'Delivered' THEN 1 END) AS not_delivered
    FROM orders AS o
    LEFT JOIN deliveries AS d
    ON o.order_id = d.order_id
    WHERE EXTRACT(YEAR FROM o.order_date) = 2024
    GROUP BY o.restaurant_id
),
last_year_data AS (
    SELECT 
        restaurant_id,
        total_orders,
        not_delivered,
        ROUND((not_delivered::numeric / total_orders::numeric) * 100, 2) AS cancel_ratio
    FROM cancel_ratio_23
),
current_year_data AS (
    SELECT 
        restaurant_id,
        total_orders,
        not_delivered,
        ROUND((not_delivered::numeric / total_orders::numeric) * 100, 2) AS cancel_ratio
    FROM cancel_ratio_24
)	

SELECT 
    c.restaurant_id AS restaurant_id_24,
    l.restaurant_id AS restaurant_id_23,
    c.cancel_ratio AS current_year_cancel_ratio,
    l.cancel_ratio AS last_year_cancel_ratio
FROM current_year_data AS c
FULL JOIN last_year_data AS l
ON c.restaurant_id = l.restaurant_id;

```

### 10. Rider Average Delivery Time: 
-- Determine each rider's average delivery time.

```sql

SELECT 
    d.rider_id,
	ROUND(AVG(EXTRACT(EPOCH FROM (d.delivery_time - o.order_time + 
	CASE WHEN d.delivery_time < o.order_time THEN INTERVAL '1 day' ELSE
	INTERVAL '0 day' END))/60),2) as avg_delivery_time
FROM orders AS o
JOIN deliveries AS d
ON o.order_id = d.order_id
WHERE d.delivery_status = 'Delivered'
GROUP BY d.rider_id
ORDER BY d.rider_id;

```

### 11. Monthly Restaurant Growth Ratio: 
-- Calculate each restaurant's growth ratio based on the total number of delivered orders since it's opening

```sql

WITH growth_ratio
AS
(
	SELECT 
		o.restaurant_id,
		EXTRACT(YEAR FROM o.order_date) as year,
		EXTRACT(MONTH FROM o.order_date) as month,
		COUNT(o.order_id) as cr_month_orders,
		LAG(COUNT(o.order_id), 1) OVER(PARTITION BY o.restaurant_id ORDER BY EXTRACT(YEAR FROM o.order_date), EXTRACT(MONTH FROM o.order_date)) as prev_month_orders
	FROM orders as o
	JOIN
	deliveries as d
	ON o.order_id = d.order_id
	WHERE d.delivery_status = 'Delivered'
	GROUP BY o.restaurant_id, year, month
)
SELECT
	restaurant_id,
	year,
	month,
	prev_month_orders,
	cr_month_orders,
	ROUND((cr_month_orders::numeric-prev_month_orders::numeric)/prev_month_orders::numeric * 100 ,2) as growth_ratio
FROM growth_ratio;

```

### 12. Customer Segmentation: 
-- Customer Segmentation: Segment customers into 'Gold' or 'Silver' groups based on their total spending 
-- compared to the average order value (AOV). If a customer's total spending exceeds the AOV, 
-- label them as 'Gold'; otherwise, label them as 'Silver'. Write an SQL query to determine each segment's 
-- total number of orders and total revenue.

```sql

SELECT 
	segment,
	SUM(total_orders) as total_orders,
	SUM(total_spent) as total_revenue
FROM

	(SELECT 
		customer_id,
		SUM(total_amount) as total_spent,
		COUNT(order_id) as total_orders,
		CASE 
			WHEN SUM(total_amount) > (SELECT AVG(total_amount) FROM orders) THEN 'Gold'
			ELSE 'silver'
		END as segment
	FROM orders
	group by customer_id
	) as t1
GROUP BY segment;

```

### 13. Rider Monthly Earnings: 
-- Calculate each rider's total monthly earnings, assuming they earn 8% of the order amount.

```sql

SELECT 
	d.rider_id,
	TO_CHAR(o.order_date, 'mm-yy') as month,
	SUM(total_amount) as revenue,
	SUM(total_amount)* 0.08 as riders_earning
FROM orders as o
JOIN deliveries as d
ON o.order_id = d.order_id
GROUP BY d.rider_id, month
ORDER BY d.rider_id, month;

```

### 14. Rider Ratings Analysis: 
-- Find the number of 5-star, 4-star, and 3-star ratings each rider has.
-- Riders receive this rating based on their delivery time.
-- If the orders are delivered in less than 15 minutes of order time, the riders get 5 star rating,
-- if they deliver between 15 to 20 minutes, they get 4 star rating, 
-- if they deliver after 20 minutes, they get 3 star rating.

```sql

WITH time_cte as
(
	SELECT 
		o.order_id,
		o.order_time,
		d.delivery_time,
		ROUND(EXTRACT(EPOCH FROM (d.delivery_time - o.order_time + 
		CASE WHEN d.delivery_time < o.order_time THEN INTERVAL '1 day' 
		ELSE INTERVAL '0 day' END ))/60,2) as delivery_took_time,
		d.rider_id
	FROM orders as o
	JOIN deliveries as d
	ON o.order_id = d.order_id
	WHERE delivery_status = 'Delivered'
),
rating_cte as 
(	
	SELECT
		rider_id,
		delivery_took_time,
		CASE 
			WHEN delivery_took_time < 15 THEN '5 star'
			WHEN delivery_took_time BETWEEN 15 AND 20 THEN '4 star'
			ELSE '3 star'
		END as stars	
	FROM time_cte
)
SELECT 
	rider_id,
	stars,
	COUNT(*) as total_stars
FROM rating_cte
GROUP BY rider_id,stars
ORDER BY rider_id,total_stars DESC;

```

### 15. Q.15 Order Frequency by Day: 
-- Analyze order frequency per day of the week and identify the peak day for each restaurant.

```sql

SELECT 
	restaurant_name, day, total_orders
FROM
(
	SELECT 
		r.restaurant_name,
		TO_CHAR(o.order_date, 'Day') as day,
		COUNT(o.order_id) as total_orders,
		DENSE_RANK() OVER(PARTITION BY r.restaurant_name ORDER BY COUNT(o.order_id)  DESC) as rank
	FROM orders as o
	JOIN
	restaurants as r
	ON o.restaurant_id = r.restaurant_id
	GROUP BY r.restaurant_name, day
	) as t1
WHERE rank = 1;

```

### 16. Customer Lifetime Value (CLV): 
-- Calculate the total revenue generated by each customer over all their orders.

```sql

SELECT 
	o.customer_id,
	c.customer_name,
	SUM(o.total_amount) as CLV
FROM orders as o
JOIN customers as c
ON o.customer_id = c.customer_id
GROUP BY o.customer_id, c.customer_name
ORDER BY o.customer_id;

```

### 17. Monthly Sales Trends: 
-- Identify sales trends by comparing each month's total sales to the previous month.

```sql

SELECT 
	EXTRACT(YEAR FROM order_date) as year,
	EXTRACT(MONTH FROM order_date) as month,
	SUM(total_amount) as current_month_sale,
	LAG(SUM(total_amount),1) OVER(ORDER BY EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date)) as prev_month_sale
FROM orders
GROUP BY year, month;

```

### 18. Rider Efficiency: 
-- Evaluate rider efficiency by determining average delivery times and identifying the lowest and highest averages.

```sql

WITH rider_cte
AS
(
	SELECT 
		d.rider_id,
		AVG(EXTRACT(EPOCH FROM (d.delivery_time - o.order_time + 
		CASE WHEN d.delivery_time < o.order_time THEN INTERVAL '1 day' ELSE
		INTERVAL '0 day' END))/60) as avg_time
	FROM orders as o
	JOIN deliveries as d
	ON o.order_id = d.order_id
	WHERE d.delivery_status = 'Delivered'
	GROUP BY d.rider_id
)

SELECT 
	ROUND(MIN(avg_time),2) as min_average_time,
	ROUND(MAX(avg_time),2) as max_average_time
FROM rider_cte;

```

### 19. Order Item Popularity: 
-- Track the popularity of specific order items over time and identify seasonal demand spikes.

```sql

SELECT 
		order_item,
		CASE 
			WHEN EXTRACT(MONTH FROM order_date) BETWEEN 4 AND 6 THEN 'Spring'
			WHEN EXTRACT(MONTH FROM order_date) > 6 AND 
			EXTRACT(MONTH FROM order_date) < 9 THEN 'Summer'
			ELSE 'Winter'
		END as seasons,
		COUNT(order_id) as total_orders
	FROM orders
GROUP BY order_item, seasons
ORDER BY order_item, total_orders DESC;

```

### 20. Rank each city based on the total revenue for last year 2023

```sql

SELECT 
	r.city,
	SUM(total_amount) as total_revenue,
	DENSE_RANK() OVER (ORDER BY SUM(total_amount) DESC) as city_rank
FROM orders as o
JOIN
restaurants as r
ON o.restaurant_id = r.restaurant_id
WHERE EXTRACT(YEAR FROM o.order_date)=2023
GROUP BY r.city;

```

## Conclusion

This project demonstrates expertise in handling complex SQL queries and solving real-world business challenges within the context of a food delivery service like Zomato. By applying a structured problem-solving approach and advanced data manipulation techniques, the analysis uncovers actionable insights that can drive data-informed decisions.

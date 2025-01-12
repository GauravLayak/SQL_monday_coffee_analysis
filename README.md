# Monday Coffee Analysis SQL Project

## Objective
The goal of this project is to analyze the sales data of Monday Coffee, a company that has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new coffee shop locations based on consumer demand and sales performance.

## Project Structure

### Database Setup

- **Database Creation**: The project starts by creating a database named `monday_coffee_db`.
- **Tables Creation**:
  1. **city table**:
 ```sql
CREATE TABLE city (
	city_id INT PRIMARY KEY,
	city_name VARCHAR(10),
	population BIGINT,
	estimated_rent FLOAT,
	city_rank INT
);
```
 2. **products table**:
 ```sql
CREATE TABLE products (
	product_id INT PRIMARY KEY,
	product_name VARCHAR(50),
	price FLOAT
);
```
 3. **customers table**:
 ```sql
CREATE TABLE customers (
	customer_id INT PRIMARY KEY,
	customer_name VARCHAR(20),
	city_id INT,
	CONSTRAINT fk_city FOREIGN KEY (city_id) REFERENCES city(city_id)
);
```
  4. **sales table**:
 ```sql
CREATE TABLE sales (
sale_id	INT PRIMARY KEY,
sale_date DATE,
product_id INT,
customer_id INT,
total FLOAT,
rating INT,
CONSTRAINT fk_products FOREIGN KEY (product_id) REFERENCES products(product_id),
CONSTRAINT fk_customers FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```
### Data Exploration : 
```sql
SELECT COUNT(*) FROM sales;
SELECT COUNT(*) FROM city;
SELECT COUNT(*) FROM customers;
SELECT COUNT(*) FROM products;
```
### Data Analysis & Findings :
## Key Questions
1. **How many people in each city are estimated to consume coffee, given that 25% of the population does?**  
  ```sql
SELECT city_name,
	ROUND((population * 0.25)/1000000,2) AS coffee_consumers_per_million,
	city_rank
FROM city
ORDER BY 2 DESC;
```

2. **What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?**  
  ```sql
SELECT ci.city_name,
	SUM(total) AS total_revenue
FROM sales AS s
JOIN customers AS c
ON s.customer_id = c.customer_id
JOIN city AS ci
ON ci.city_id = c.city_id
	WHERE EXTRACT(YEAR FROM s.sale_date) = 2023
	AND
	EXTRACT(QUARTER FROM s.sale_date) = 4
	GROUP BY 1
	ORDER BY 2 DESC;
```

3. **How many units of each coffee product have been sold?**
 ```sql
SELECT 
	p.product_name,
	COUNT(s.sale_id) AS total_orders
FROM products AS p
LEFT JOIN 
sales AS s
ON s.product_id = p.product_id
GROUP BY 1 
ORDER BY 2 DESC;
```
   
4. **What is the average sales amount per customer in each city?**
```sql
SELECT ci.city_name,
	SUM(total) AS total_revenue,
	COUNT(DISTINCT s.customer_id) AS total_customer,
		ROUND(
			SUM(total)::numeric/COUNT(DISTINCT s.customer_id)::numeric, 2) AS avg_sale_per_cust
FROM sales AS s
JOIN customers AS c
ON s.customer_id = c.customer_id
JOIN city AS ci
ON ci.city_id = c.city_id
		GROUP BY 1
	ORDER BY 2 DESC;
```  

5. **Provide a list of cities along with their populations and estimated coffee consumers.**
```sql
WITH city_table AS (
SELECT 
	city_name,
	ROUND(population * 0.25/1000000, 2) AS coffee_consumers
FROM city),
customer_table AS (
SELECT ci.city_name,
	COUNT(DISTINCT c.customer_id) AS unique_cust
FROM sales AS s
	JOIN customers AS c
	ON c.customer_id = s.customer_id
	JOIN city AS ci
	ON ci.city_id = c.city_id
GROUP BY 1)
SELECT ct.city_name,
	ct.coffee_consumers,
	cit.unique_cust
FROM city_table AS ct
JOIN
customer_table AS cit
ON cit.city_name = ct.city_name;
```

6. **What are the top 3 selling products in each city based on sales volume?**  
```sql
SELECT * FROM (
	SELECT 
		ci.city_name,
		p.product_name,
		COUNT(s.sale_id) AS total_orders,
		DENSE_RANK() OVER(PARTITION BY ci.city_name ORDER BY COUNT(s.sale_id) DESC) AS rank 
	FROM sales AS s
	JOIN products AS p
	ON s.product_id = p.product_id
	JOIN customers AS c
	ON c.customer_id = s.customer_id
	JOIN city as ci
	ON ci.city_id = c.city_id
	GROUP BY 1, 2
	) AS t1
WHERE rank <= 3;
```   

7. **How many unique customers are there in each city who have purchased coffee products?**  
```sql
SELECT 
	ci.city_name,
	COUNT(DISTINCT c.customer_id) AS unique_customers
FROM city AS ci
LEFT JOIN 
customers AS c
ON c.city_id = ci.city_id
JOIN sales AS s
ON s.customer_id = c.customer_id
WHERE 
s.product_id IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14) 
GROUP BY 1;
```
8. **Find each city and their average sale per customer and avg rent per customer**  
```sql
WITH city_table
AS	(
	SELECT 
		ci.city_name,
		SUM(s.total) AS total_revenue,
		COUNT(DISTINCT s.customer_id) AS total_customers,
		ROUND(
			SUM(s.total)::numeric/ COUNT(DISTINCT s.customer_id)::numeric, 2
		) AS avg_sale_per_cust
	FROM city AS ci
	LEFT JOIN 
	customers AS c
	ON c.city_id = ci.city_id
	JOIN sales AS s
	ON s.customer_id = c.customer_id
	GROUP BY 1
	ORDER BY 2 DESC
	),
city_name AS (
	SELECT city_name,
	estimated_rent
	FROM city
	)
SELECT c.city_name,
	c.estimated_rent,
	ct.total_customers,
	ct.avg_sale_per_cust,
	ROUND(
		c.estimated_rent::numeric/ct.total_customers::numeric, 2) AS avg_rent_per_cust
FROM city AS c
JOIN city_table AS ct
ON c.city_name = ct.city_name
ORDER BY 4 DESC;
```

9. **Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly)**  
```sql
WITH monthly_sales AS (
	SELECT 
		ci.city_name,
		EXTRACT(MONTH FROM sale_date) AS month,
		EXTRACT(YEAR FROM sale_date) AS year,
		SUM(s.total) AS total_sales
	FROM sales AS s
	JOIN customers AS c
	ON c.customer_id = s.customer_id
	JOIN city AS ci
	ON ci.city_id = c.city_id
	GROUP BY 1, 2, 3
	ORDER BY 1, 3, 2 ),
growth_ratio AS (
		SELECT city_name,
			month, 
			year,
			total_sales AS current_month_sale,
			LAG(total_sales, 1) OVER(PARTITION BY city_name ORDER BY year, month) AS last_month_sale
		FROM monthly_sales ) 
SELECT city_name,
		month,
		year,
		last_month_sale,
		current_month_sale,
		ROUND(
			(current_month_sale - last_month_sale)::numeric/last_month_sale::numeric * 100, 2) AS growth_ratio
FROM growth_ratio
WHERE last_month_sale IS NOT NULL;
```

## Conclusion

This project serves as a comprehensive introduction to SQL for data analysts, covering database setup, exploratory data analysis, and business-driven SQL queries. The findings from this project can help drive business decisions by understanding sales patterns, customer behavior, and product performance.


# Mysql_Advance_EDA_Analysis_project2
this shows advance EDA analysis of key metrics of the Customers behaviour and explains about all window_functions , frames , joins , sets , aggreates etc....


PROCESS OF ADVANCE EDA ANALYSIS INCLUDES 

1) CHANGE OVER TIME ANALYSIS 

2) CUMMULATIVE ANALYSIS - helps to understand whether our bussiness is growing or declining.

3) PERFORMANCE ANALYSIS - compare current value to target value , helps mesure success and compare performance.

4) PART TO PART WHOLE ANALYSIS - analyse how individual part is performing compared to overall , allow us to understand which category shows great impact on bussiness.

5) DATA SEGEMENTATION - group the data based on specific range , helps correlation between two measures.

6)BUILD CUSTOMER REPORT - get all key metrics for the customer behaviour

FINALLY WE CALCULATE AVG ORDER VALUE and AVG MONTHLY SPENDING .


THE MySql Code is :


USE datawarehouseanalytics;

SELECT * FROM gold_dim_products;
SELECT * FROM gold_fact_sales;
SELECT * FROM gold_dim_customers;

-- 1) CHANGE OVER TIME ANALYSIS --
SELECT 
YEAR(order_date) AS order_year,
MONTH(order_date) AS order_month,
SUM(sales_amount) AS total_sales,
COUNT(DISTINCT customer_key) as total_customers,
SUM(quantity) AS total_quantity
FROM gold_fact_sales
WHERE order_date IS NOT NULL
GROUP BY YEAR(order_date) , MONTH(order_date)
ORDER BY YEAR(order_date) , MONTH(order_date);

-- OR 

SELECT 
DATE_FORMAT(order_date , "%Y-%m-%d") AS order_date,
SUM(sales_amount) AS total_sales,
COUNT(DISTINCT customer_key) as total_customers,
SUM(quantity) AS total_quantity
FROM gold_fact_sales
WHERE order_date IS NOT NULL
GROUP BY DATE_FORMAT(order_date , "%Y-%m-%d") 
ORDER BY DATE_FORMAT(order_date , "%Y-%m-%d");

-- 2) CUMMULATIVE ANALYSIS 
-- helps to understand whether our bussiness is growing or declining.
-- Task - calculate the total sales per month and runnning total of sales over time.

SELECT order_date , 
total_sales,
SUM(total_sales) OVER(PARTITION BY order_date ORDER BY order_date) AS running_total_sales,
AVG(avg_price) OVER(ORDER BY order_date) AS moving_avg_price
FROM 
(
-- WINDOW FUNCTION
SELECT 
DATE_FORMAT(order_date,"%Y-%m-%d") AS order_date,
SUM(sales_amount) as total_sales,
AVG(price) AS avg_price
FROM gold_fact_sales
WHERE order_date IS NOT NULL
GROUP BY DATE_FORMAT(order_date,"%Y-%m-%d")
ORDER BY DATE_FORMAT(order_date,"%Y-%m-%d")
) t;
-- IN 37,38 Th line u can use also partittion by or not (ur wish) 
-- BUT in that two lines sum(alias_name1) and avg(alias_name2) - means name must be like ur defined user name . if original table name used then it shows error.


-- 3) PERFORMANCE ANALYSIS - compare current value to target value , helps mesure success and compare performance.
-- Task - analyse yearly performance of the products by comparing each products sales to both its AVERAGE SALES PERFORMANCE and PREVIOUS YEARS SALES. 
WITH yearly_product_sales AS 
(
SELECT 
YEAR(f.order_date) AS order_year,
p.product_name,
SUM(f.sales_amount) AS current_sales
FROM gold_fact_sales f
LEFT JOIN gold_dim_products p 
ON f.product_key = p.product_key
WHERE f.order_date IS NOT NULL 
GROUP BY YEAR(f.order_date) , 
p.product_name
)
SELECT product_name,
order_year,
current_sales,
AVG(current_sales) OVER(PARTITION BY product_name ORDER BY current_sales DESC) AS avg_sales,
current_sales - AVG(current_sales) OVER(PARTITION BY product_name) AS diff_avg,
CASE
    WHEN current_sales - AVG(current_sales) OVER(PARTITION BY product_name) > 0 THEN "above avg"
    WHEN current_sales - AVG(current_sales) OVER(PARTITION BY product_name) < 0 THEN "below avg"
    ELSE "avg"
END avg_change,
-- YEAR OVER YEAR ANALYSIS 
LAG(current_sales) OVER(PARTITION BY product_name ORDER BY order_year) AS py_sales,
current_sales - LAG(current_sales) OVER(PARTITION BY product_name ORDER BY order_year) AS diff_py,
CASE
    WHEN current_sales - LAG(current_sales) OVER(PARTITION BY product_name ORDER BY order_year) > 0 THEN "increase"
    WHEN current_sales - LAG(current_sales) OVER(PARTITION BY product_name ORDER BY order_year) < 0 THEN "decrease"
    ELSE "no change"
END py_change
FROM yearly_product_sales
ORDER BY product_name , order_year;


-- 4) PART TO PART WHOLE ANALYSIS - analyse how individual part is performing compared to overall , allow us to understand which category shows great impact on bussiness.
-- Task - which categories contribute to most to overall sales 
WITH category_sales AS 
(
SELECT p.category AS category,
SUM(f.sales_amount) AS total_sales
FROM gold_fact_sales f
LEFT JOIN gold_dim_products p
ON p.product_key = f.product_key
WHERE category IS NOT NULL
GROUP BY category
)
SELECT category, 
total_sales,
SUM(total_sales) OVER() overall_sales,
CONCAT(ROUND((CAST(total_sales AS FLOAT) /SUM(total_sales) OVER())*100 , 2) , "%") AS percentage_of_total 
FROM category_sales
ORDER BY total_sales DESC;

-- 5) DATA SEGEMENTATION - group the data based on specific range , helps correlation between two measures.
-- Task - segement products into cost ranges and count how many products falls into each segement.
WITH product_segements AS
(
SELECT product_name,
product_key,
cost,
CASE 
    WHEN cost < 100 THEN "below 100"
	WHEN cost BETWEEN 100 AND 500 THEN "100-500"
	WHEN cost BETWEEN 500 AND 1000 THEN "500-1000"
    ELSE "above 1000"
END "cost_range"
FROM gold_dim_products
)

SELECT cost_range,
COUNT(product_key) AS total_products 
FROM product_segements
GROUP BY cost_range
ORDER BY total_products DESC;

-- group them now into three segements based on their behaviour

WITH customer_spending AS
(
SELECT c.customer_key,
SUM(f.sales_amount) AS total_spending,
MIN(f.order_date) AS first_order,
MAX(f.order_date) AS last_order,
DATEDIFF(MONTH , first_order , last_order) AS lifespan
FROM gold_fact_sales f 
LEFT JOIN gold_dim_customers c
ON f.customer_key = c.customer_key
GROUP BY c.customer_key
)

SELECT customer_segement,
COUNT(customer_key) AS total_customers
FROM (
    SELECT 
    customer_key,
    CASE 
        WHEN lifespan >=12 AND total_spending >=5000 THEN "VIP"
        WHEN lifespan >=12 AND total_spending <5000 THEN "regular"
        ELSE "new"
    END customer_segement
    FROM customer_spending
    ) t
GROUP BY customer_segement
ORDER BY total_customers;


-- 6)BUILD CUSTOMER REPORT - get all key metrics for the customer behaviour

-- CREATE VIEW gold_report_customers AS 

WITH base_query AS (
    SELECT 
        f.order_number, 
        f.product_key, 
        f.sales_amount, 
        f.quantity,               -- 1. ADDED: Needed for SUM(quantity) in the outer query
        f.order_date, 
        c.customer_key,           -- 2. FIXED: Removed the duplicate f.customer_key
        c.customer_number, 
        CONCAT(c.first_name, ' ', c.last_name) AS customer_name, 
        TIMESTAMPDIFF(YEAR, c.birthdate, CURDATE()) AS age  -- u can also use now() but it shows hh-mm-ss
    FROM gold_fact_sales f 
    LEFT JOIN gold_dim_customers c 
        ON f.customer_key = c.customer_key 
    WHERE f.order_date IS NOT NULL
) , customer_aggregation AS 
(
SELECT 
    customer_key, 
    customer_number, 
    customer_name,                -- 3. FIXED: Changed 'full_name' to 'customer_name' to match the CTE
    age, 
    COUNT(DISTINCT order_number) AS total_orders, 
    SUM(sales_amount) AS total_sales, 
    SUM(quantity) AS total_quantity, 
    MAX(order_date) AS last_order_date, 
    TIMESTAMPDIFF(YEAR, MIN(order_date), MAX(order_date)) AS lifespan, 
    COUNT(product_key) AS total_products 
FROM base_query 
GROUP BY 
    customer_key, 
    customer_number,              -- 4. FIXED: Added customer_number to prevent GROUP BY errors
    customer_name, 
    age
)

SELECT 
	customer_key, 
    customer_number,              -- 4. FIXED: Added customer_number to prevent GROUP BY errors
    customer_name, 
    age,
    CASE 
        WHEN age<10 THEN "child"
        WHEN age BETWEEN 10 AND 20 THEN "teenage"
        WHEN age BETWEEN 20 AND 50 THEN "senior"
	    ELSE "old"
    END age_category,
total_orders,
total_sales,
total_quantity,
last_order_date,
total_products,
lifespan,
-- AVG ORDER VALUE
CASE 
    WHEN total_sales = 0 THEN 0
    ELSE total_sales/total_orders
END avg_order_value,

-- AVG MONTHLY SPENDING 

CASE 
    WHEN lifespan = 0 THEN total_sales
    ELSE total_sales/lifespan
END avg_monthly_spending

FROM customer_aggregation;

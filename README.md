## Problem Statement
Get insight on sales from an e-commerce website that has information of 100k orders made from 2016 to 2018. The dataset is stored in multiple tables related to orders, order payment methods, product price and other details, order reviews, order items, customer details, sellers details, geolocation information.

The information can shed light on various aspects of the business, such as order processing, pricing strategies, payment and shipping efficiency, customer demographics, product characteristics, and customer satisfaction levels.

## 1. Exploring the data.
### What is the time period for which the data is given?
```sql
SELECT 
  min(DATE(order_purchase_timestamp)) as first_date, 
  max(DATE(order_purchase_timestamp)) as last_date, 
  DATE_DIFF(max(DATE(order_purchase_timestamp)),min(DATE(order_purchase_timestamp)), DAY) as days_difference
FROM orders;
```

### List out all the cities and states in the dataset.
```sql
SELECT DISTINCT geolocation_city as cities 
FROM geolocation 
ORDER BY cities 
LIMIT 10; 

SELECT DISTINCT geolocation_state as states 
FROM geolocation 
ORDER BY states 
LIMIT 10;
```
### Order Status Counts
```sql
SELECT order_status,
  COUNT(*) count_status
FROM orders
GROUP BY order_status;
```

## 2. In-depth exploration:
### 1. Is there a growing trend on e-commerce in the region?
We can look at the number of purchases made across different months or years. Number of orders made increase rapidly from 2016 to 2017, then increased marginally from 2017 to 2018. There are more purchases in the first half or middle of the year like May, July, August than in the period after the month of August.
```sql
WITH base AS (
  SELECT EXTRACT(month FROM order_purchase_timestamp) order_months,
  COUNT(1) no_orders,
  SUM(price) total_price
  FROM orders o JOIN order_items oi ON oi.order_id = o.order_id
  GROUP BY EXTRACT(month FROM order_purchase_timestamp) 
  )
SELECT * FROM base
ORDER BY total_price DESC;

WITH base AS (
  SELECT
    EXTRACT(year FROM order_purchase_timestamp) order_year,
    EXTRACT(month FROM order_purchase_timestamp) order_month,
    COUNT(1) order_counts,
    SUM(price) total_price
  FROM orders o JOIN order_items oi ON oi.order_id = o.order_id
  GROUP BY EXTRACT(year FROM order_purchase_timestamp), EXTRACT(month FROM order_purchase_timestamp) 
  )
SELECT * FROM base 
ORDER BY total_price DESC;
```

### 2. During what time of the day, do the customers mostly place their orders? (Dawn, Morning, Afternoon or Night)?
```sql
WITH sales_time AS 
  (SELECT hours,no_orders, 
  CASE WHEN hours>= 0 AND hours < 6 THEN 'Dawn' 
  WHEN hours>=6 AND hours<12 THEN 'Morning' 
  WHEN hours>=12 AND hours<18 THEN 'Afternoon' 
  ELSE 'Evening' END AS Time 
  FROM ( 
    SELECT EXTRACT(HOUR FROM order_purchase_timestamp) as hours, 
    COUNT(order_id) as no_orders 
    FROM orders 
    GROUP BY hours 
    ORDER BY hours)
  ) 
SELECT Time, SUM(no_orders) AS no_order 
FROM sales_time 
GROUP BY Time;
```

## 3.	Evolution of E-commerce orders in the region:
### 1.	Get month on month orders by region, states (No. of orders in each month for each state is shown)
```sql
SELECT EXTRACT(MONTH FROM order_purchase_timestamp) as months, 
  customer_state as states, 
  COUNT(order_id) as no_orders 
FROM orders AS o JOIN customers AS c ON o.customer_ID = c.customer_id 
GROUP BY months,states 
ORDER BY months,states;
```
WITH
months_region AS (
WITH
geolocation_region AS (
SELECT
*,
CASE
WHEN geolocation_state IN ('RO', 'AC', 'AM', 'RR', 'PA', 'AP', 'TO') THEN 'Norte'
WHEN geolocation_state IN ('MA','PI', 'CE', 'RN', 'PB', 'PE', 'AL', 'SE', 'BA') THEN 'Nordeste'
WHEN geolocation_state IN ('MG', 'ES', 'RJ', 'SP') THEN 'Sudeste'
WHEN geolocation_state IN ('PR', 'SC', 'RS') THEN 'Sul'
ELSE 'Centro-Oeste' END region
FROM geolocation )
SELECT EXTRACT(year FROM order_purchase_timestamp) year_ord,
EXTRACT(month FROM order_purchase_timestamp) month_ord,
region,
customer_state state,
COUNT(1) no_of_orders_per_region_state
FROM
scaler.orders o
JOIN
scaler.customers c
ON
c.customer_id = o.customer_id
JOIN
geolocation_region gr
ON
c.customer_zip_code_prefix = gr.geolocation_zip_code_prefix
GROUP BY

EXTRACT(year
FROM
order_purchase_timestamp),
EXTRACT(month
FROM
order_purchase_timestamp),
region,
customer_state )
SELECT
*
FROM
months_region
ORDER BY
year_ord,
month_ord

### 2. How are the customers distributed across all the states?
The most number (40,302) of customers are located in SP, then RJ, then MG, and so on with the least number in RR.
```sql
SELECT customer_state as states, 
  COUNT(DISTINCT customer_unique_id) as no_customers 
FROM customers
GROUP BY states 
ORDER BY no_customers DESC;
```
## 4.	Impact on Economy: Analyze the money movement by e-commerce by looking at order prices, freight and others.
### 1. What is the percent increase in cost of order from 2017 to 2018 (include months between Jan to Aug only)?
```sql
WITH yearly_costs AS 
  (SELECT EXTRACT(MONTH FROM order_purchase_timestamp) as months, 
  EXTRACT(YEAR FROM order_purchase_timestamp) as years, 
  SUM(payment_value) as cost 
  FROM orders AS o JOIN payments AS p ON o.order_ID = p.order_id
  GROUP BY years, months 
  HAVING months IN (1,2,3,4,5,6,7,8) AND years IN (2017,2018) 
  ORDER BY years,months) 
SELECT SUM(cost) as costs_year2018, 
LAG(SUM(cost),1) OVER(ORDER BY years) as costs_year2017, 
ROUND(((SUM(cost)/(LAG(SUM(cost),1) OVER(ORDER BY years)))-1)*100,2) AS percent_change 
FROM yearly_costs 
GROUP BY years 
ORDER BY years DESC 
LIMIT 1;
```

### 2. Mean and sum of price and freight value by customer states.
```sql
SELECT customer_state,
  ROUND(SUM(price),2) as sum_price, 
  ROUND(AVG(price),2) as avg_price, 
  ROUND(SUM(freight_value),2) as sum_freight, 
  ROUND(AVG(freight_value),2) as avg_freight 
FROM orders AS o JOIN customers AS c ON o.customer_id = c.customer_id JOIN order_items AS oi ON o.order_ID = oi.order_id 
GROUP BY customer_state 
ORDER BY customer_state 
LIMIT 10;
```

## 5. Analysis on sales, freight and delivery time
### 1. Calculating the days difference between purchasing, delivery and estimated delivery dates
```sql
SELECT DATE(order_purchase_timestamp) AS purchase, 
  DATE(order_delivered_carrier_date) AS delivery, 
  DATE(order_estimated_delivery_date) AS estimated, 
  DATE_DIFF(order_estimated_delivery_date,order_delivered_carrier_date,DAY) AS estim_deliv_diff, 
  DATE_DIFF(order_delivered_carrier_date,order_purchase_timestamp, DAY) AS deliv_purchase_diff, 
FROM orders
LIMIT 10;

SELECT DATE(order_purchase_timestamp) AS purchase, 
  DATE(order_delivered_customer_date) AS delivery, 
  DATE(order_estimated_delivery_date) AS estimated, 
  DATE_DIFF(order_delivered_customer_date,order_purchase_timestamp, DAY) AS time_to_delivery, 
  DATE_DIFF(order_estimated_delivery_date,order_delivered_customer_date,DAY) AS diff_estimated_delivery, 
FROM orders
ORDER BY delivery DESC 
LIMIT 10;
```

### 2.	Create columns time_to_delivery and diff_estimated_delivery
SELECT 
  DATE(order_purchase_timestamp) AS purchase,
  DATE(order_delivered_customer_date) AS delivery,
  DATE(order_estimated_delivery_date) AS estimated,
  DATE_DIFF(order_delivered_customer_date,order_purchase_timestamp, DAY) AS time_to_delivery,
  DATE_DIFF(order_estimated_delivery_date,order_delivered_customer_date,DAY) AS diff_estimated_delivery,
  FROM `scaler-dsml-361413.ecommerce.orders`
  ORDER BY delivery DESC
  LIMIT 10;

### 3. Group data by state, take mean of freight_value, time_to_delivery, diff_estimated_delivery
```sql
CREATE VIEW vw_delivery_dates AS 
SELECT customer_id, freight_value, 
  DATE(order_purchase_timestamp) AS purchase, 
  DATE(order_delivered_customer_date) AS delivery, 
  DATE(order_estimated_delivery_date) AS estimated, 
  DATE_DIFF(order_delivered_customer_date,order_purchase_timestamp, DAY) AS time_to_delivery, 
  DATE_DIFF(order_estimated_delivery_date,order_delivered_customer_date,DAY) AS diff_estimated_delivery, 
FROM orders AS o JOIN order_items AS oi ON o.order_id = oi.order_id; 

SELECT customer_state, 
  ROUND(AVG(freight_value),2) AS mean_freight, 
  CEIL(AVG(time_to_delivery)) AS mean_time_to_delivery, 
  CEIL(AVG(diff_estimated_delivery)) AS mean_diff_estim_delivery 
FROM vw_delivery_dates AS vw JOIN customers AS c ON vw.customer_id=c.customer_id 
GROUP BY customer_state 
ORDER BY customer_state;
```

### 4. Top 5 states with highest/lowest average freight value
State RR has the most expensive freight value, and SP has cheapest freight value.
```sql
SELECT customer_state, 
  ROUND(AVG(freight_value),2) AS mean_freight, 
  CEIL(AVG(time_to_delivery)) AS mean_time_to_delivery, 
  CEIL(AVG(diff_estimated_delivery)) AS mean_diff_estim_delivery 
FROM vw_delivery_dates AS vw JOIN customers AS c ON vw.customer_id=c.customer_id 
GROUP BY customer_state 
ORDER BY mean_freight 
LIMIT 5;
```

### Top 5 states with highest/lowest average time to delivery
State SP has fastest delivery from purchase date, while states AP and RR have slowest.
```sql
SELECT customer_state, 
  ROUND(AVG(freight_value),2) AS mean_freight, 
  CEIL(AVG(time_to_delivery)) AS mean_time_to_delivery,
  CEIL(AVG(diff_estimated_delivery)) AS mean_diff_estim_delivery 
FROM vw_delivery_dates AS vw JOIN customers AS c ON vw.customer_id=c.customer_id 
GROUP BY customer_state 
ORDER BY mean_time_to_delivery DESC 
LIMIT 5;
```

### Top 5 states where delivery is really fast/ not so fast compared to estimated date 
State AL has fastest delivery compared to estimated date, and state AC has the slowest delivery or highest delay from estimated date.
```sql
SELECT customer_state, 
  ROUND(AVG(freight_value),2) AS mean_freight, 
  CEIL(AVG(time_to_delivery)) AS mean_time_to_delivery, 
  CEIL(AVG(diff_estimated_delivery)) AS mean_diff_estim_delivery 
FROM vw_delivery_dates AS vw JOIN customers AS c ON vw.customer_id=c.customer_id 
GROUP BY customer_state 
ORDER BY mean_diff_estim_delivery DESC 
LIMIT 5;
```

## 6. Payment type analysis:
### 1. Month over Month count of orders for different payment types 
Most popular mode of payment is credit card, then UPI, then vouchers and finally debit cards are the least popular method. The most number of purchases are made in the middle of a year.
```sql
SELECT payment_type, 
  EXTRACT(MONTH FROM order_purchase_timestamp) as months, 
  COUNT(o.order_id) as no_orders 
FROM orders AS o JOIN payments AS p ON o.order_id=p.order_id 
GROUP BY payment_type,months 
ORDER BY payment_type,months;
```

### 2. Distribution of payment installments and count of orders 
```sql
SELECT payment_installments, 
  COUNT(o.order_id) as no_orders 
FROM orders AS o JOIN payments AS p ON o.order_id=p.order_id 
GROUP BY payment_installments 
ORDER BY payment_installments;
```
## Insights:
•	Number of orders made increase rapidly from 2016 to 2017, then increased marginally from 2017 to 2018. This could be because of lesser number of months in year 2016 (since September to December) compared to all the months in 2017.  There was 136.98% increase in sales from 2017 to 2018.
•	There are more purchases in the first half or middle of the year like May, July, August than in the period after the month of August. Business should focus on improving sales in the months after August.
•	Brazilians tend to buy more during afternoon or evening time maybe because of off work hours.
•	The most number (40,302) of customers are located in SP, then RJ, then MG, and so on with the least number in RR.
•	State RR has the most expensive freight value, and SP has cheapest freight value.
•	State SP has fastest delivery from purchase date, while states AP and RR have slowest.
•	State AL has fastest delivery compared to estimated date, and state AC has the slowest delivery or highest delay from estimated date.
•	Most popular mode of payment is credit card, then UPI, then vouchers and finally debit cards are the least popular method. Most number of purchases are made in the middle of a year.
•	Most customers pay in a single or fewer than 5 number of installments.

## Recommendations:
•	Business should focus on improving sales in the months after August. It could offer discounts during festivals or holiday season or year-end sales between September to December. 
•	Since the customers get more time to buy in the afternoon and evening after work hours, the store could provide some discounts to increase sales during those times in a day.
•	State SP has the most number of customers and also the fastest delivery services. Delivery dates are still not close to estimated dates with lowest delay being 8 days in the AL state. Delivery vehicles and drivers need to increased in all states. Supply chain delays also need to be managed better to decrease time to delivery and time delay from estimated to delivery date.
•	Since credit cards are the most preferred mode of payment, customers could be offered points on credit cards for every purchase that can entice them for more purchases.

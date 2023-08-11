## Problem Statement
This is a public dataset of 100k orders made at Olist Store, a Brazilian ecommerce company, from 2016 to 2018 at multiple marketplaces across Brazil. Its features store order information from multiple dimensions: order status, price, payment and freight performance, customer location, product attributes and reviews written by customers. There is a table about geolocation dataset that relates Brazilian zip codes to latitude/longitude coordinates.

## Dataset source
Olist, &amp; André Sionek. (2018). <i>Brazilian E-Commerce Public Dataset by Olist</i> [Data set]. Kaggle. https://doi.org/10.34740/KAGGLE/DSV/195341

## Tools
SQL(BigQuery), Excel, Tableau

## Data Schema
The data is stored in multiple tables shown below for reference:
<p align="center">
<img src="https://imgur.com/HRhd2Y0.png" width=100% height=100%>

## 1. Exploring the data:
The CSV files were first loaded in Microsoft Excel and transformed using functions such as VLOOKUP and conditional functions. Some more useful columns were created such as translation of Spanish product category names to English names for our understanding. 

Later, the files were loaded in Google BigQuery and the various queries created are shown below:

### The time period for orders in dataset
```sql
SELECT 
  min(DATE(order_purchase_timestamp)) as first_date, 
  max(DATE(order_purchase_timestamp)) as last_date, 
  DATE_DIFF(max(DATE(order_purchase_timestamp)),min(DATE(order_purchase_timestamp)), DAY) as days_difference
FROM orders;
```
The dates for orders available in the dataset range from 2016-09-04 to 2018-10-17.
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/1.PNG" width=40% height=40%>

### List of the cities and states in the dataset
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
A small snippet of the list of cities and states of customer address locations is shown below:
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/2.PNG" width=20% height=20%> <img src="https://github.com/mkadwani/SQLproject/blob/screenshots/3.PNG" width=18% height=18%>

### Count of order status
```sql
SELECT order_status,
  COUNT(*) count_status
FROM orders
GROUP BY order_status
ORDER BY count_status DESC;
```

<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/4b.PNG" width=40% height=40%>

## 2. In-depth exploration:
### 1. Is there a growing trend on e-commerce in the region?
We can look at the number of purchases made across different months or years. Number of orders made increase rapidly from 2016 to 2017, then increased marginally from 2017 to 2018. There are more purchases in the first half or middle of the year like May, July, August than in the period after the month of August.
```sql
SELECT
  EXTRACT(year FROM order_purchase_timestamp) order_year,
  EXTRACT(month FROM order_purchase_timestamp) order_month,
  COUNT(1) order_counts,
  SUM(price) total_price
FROM orders o JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY 1,2
ORDER BY 1,2
```
<p align="center"><img src="https://github.com/mkadwani/SQLproject/blob/screenshots/excel5.PNG" width=50% height=50%>
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/5.PNG" width=40% height=40%>

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
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/6.PNG" width=40% height=40%>

## 3.	Evolution of E-commerce orders in the region:
### 1. Month on month orders by region, states (No. of orders in each month for each state is shown)
```sql
SELECT EXTRACT(MONTH FROM order_purchase_timestamp) as months, 
  customer_state as states, 
  COUNT(order_id) as no_orders 
FROM orders AS o JOIN customers AS c ON o.customer_ID = c.customer_id 
GROUP BY months,states 
ORDER BY months,states;
```
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/7.PNG" width=40% height=40%>

### 2. Distribution of customers across all the states
The most number (40,302) of customers are located in SP, then RJ, then MG, and so on with the least number in RR.
```sql
SELECT customer_state as states, 
  COUNT(DISTINCT customer_unique_id) as no_customers 
FROM customers
GROUP BY states 
ORDER BY no_customers DESC;
```
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/8.PNG" width=40% height=40%>

## 4.	Impact on Economy:
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
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/9.PNG" width=40% height=40%>

### 2. Mean and sum of price and freight value by customer states
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
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/10.PNG" width=40% height=40%>
 
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
```
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/11.png" width=40% height=40%>

### 2.	Create columns time_to_delivery and diff_estimated_delivery
```sql
SELECT 
  DATE(order_purchase_timestamp) AS purchase,
  DATE(order_delivered_customer_date) AS delivery,
  DATE(order_estimated_delivery_date) AS estimated,
  DATE_DIFF(order_delivered_customer_date,order_purchase_timestamp, DAY) AS time_to_delivery,
  DATE_DIFF(order_estimated_delivery_date,order_delivered_customer_date,DAY) AS diff_estimated_delivery,
FROM `scaler-dsml-361413.ecommerce.orders`
ORDER BY delivery DESC
LIMIT 10;
```
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/12.png" width=40% height=40%>

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
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/13.png" width=40% height=40%>

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
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/14.png" width=40% height=40%>

<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/14b.png" width=40% height=40%>

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
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/15.png" width=40% height=40%>

<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/15b.png" width=40% height=40%>

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
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/16.png" width=40% height=40%>
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/16b.png" width=40% height=40%>

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
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/17a.png" width=20% height=20%><img src="https://github.com/mkadwani/SQLproject/blob/screenshots/17b.png" width=20% height=20%><img src="https://github.com/mkadwani/SQLproject/blob/screenshots/17c.png" width=20% height=20%><img src="https://github.com/mkadwani/SQLproject/blob/screenshots/17d.png" width=20% height=20%>

### 2. Distribution of payment installments and count of orders 
```sql
SELECT payment_installments, 
  COUNT(o.order_id) as no_orders 
FROM orders AS o JOIN payments AS p ON o.order_id=p.order_id 
GROUP BY payment_installments 
ORDER BY payment_installments;
```
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/18.png" width=40% height=40%>

## Tableau dashboard
<p align="center">
<img src="https://github.com/mkadwani/SQLproject/blob/screenshots/tableau.PNG" width=100% height=100%>
  
## Insights:
-	Number of orders made increase rapidly from 2016 to 2017, then increased marginally from 2017 to 2018. This could be because of lesser number of months in year 2016 (since September to December) compared to all the months in 2017.  There was 136.98% increase in sales from 2017 to 2018.
-	There are more purchases in the first half or middle of the year like May, July, August than in the period after the month of August. Business should focus on improving sales in the months after August.
- Brazilians tend to buy more during afternoon or evening time maybe because of off work hours.
-	The most number (40,302) of customers are located in SP, then RJ, then MG, and so on with the least number in RR.
-	State RR has the most expensive freight value, and SP has cheapest freight value.
-	State SP has fastest delivery from purchase date, while states AP and RR have slowest.
-	State AL has fastest delivery compared to estimated date, and state AC has the slowest delivery or highest delay from estimated date.
-	Most popular mode of payment is credit card, then UPI, then vouchers and finally debit cards are the least popular method. Most number of purchases are made in the middle of a year.
-	Most customers pay in a single or fewer than 5 number of installments.

## Recommendations:
-	Business should focus on improving sales in the months after August. It could offer discounts during festivals or holiday season or year-end sales between September to December. 
- Since the customers get more time to buy in the afternoon and evening after work hours, the store could provide some discounts and target more customers for marketing campaigns to increase sales during those times in a day.
-	State SP has the most number of customers and also the fastest delivery services. Delivery dates are still not close to estimated dates with lowest delay being 8 days in the AL state. Delivery vehicles and drivers need to increased in all states. Supply chain delays also need to be managed better to decrease time to delivery and time delay from estimated to delivery date.
-	Since credit cards are the most preferred mode of payment, customers could be offered points on credit cards for every purchase that can entice them for more purchases.

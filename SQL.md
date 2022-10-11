### Time period for which data is given
```
SELECT max(DATE(order_purchase_timestamp)) as first_date, 
min(DATE(order_purchase_timestamp)) as last_date, 
DATE_DIFF(max(DATE(order_purchase_timestamp)),min(DATE(order_purchase_timestamp)), DAY) as days_difference 
FROM orders;
```

### List of cities and states in dataset
```
SELECT DISTINCT geolocation_city as cities 
FROM geolocation 
ORDER BY cities 
LIMIT 10; 

SELECT DISTINCT geolocation_state as states 
FROM geolocation 
ORDER BY states 
LIMIT 10;
```

### Is there a growing trend on e-commerce in Brazil?
```
SELECT EXTRACT(YEAR FROM order_purchase_timestamp) as years, 
COUNT(order_id) as no_orders 
FROM orders
GROUP BY years 
ORDER BY years 
LIMIT 10;

SELECT EXTRACT(MONTH FROM order_purchase_timestamp) as months, 
COUNT(order_id) as no_orders, 
FROM orders
GROUP BY months 
ORDER BY months;
```

### Time at which most customers prefer to buy
```
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
SELECT Time,
SUM(no_orders) AS no_order 
FROM sales_time 
GROUP BY Time;
```

### Get orders by months and states
```
SELECT EXTRACT(MONTH FROM order_purchase_timestamp) as months, 
customer_state as states, 
COUNT(order_id) as no_orders 
FROM orders AS o JOIN customers AS c ON o.customer_ID = c.customer_id 
GROUP BY months,states 
ORDER BY months,states;
```

### Distribution of customers in Brazil
```
SELECT customer_state as states, 
COUNT(DISTINCT customer_unique_id) as no_customers 
FROM customers
GROUP BY states 
ORDER BY no_customers DESC;
```

### Percent increase in cost of order from 2017 to 2018
```
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

### Mean and sum of price and freight value by customer states
```
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

## Analysis on sales, freight and delivery time
### Calculating the days difference between purchasing, delivery and estimated delivery dates
```
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

### Group data by state, take mean of freight_value, time_to_delivery, diff_estimated_delivery
```
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

### Top 5 states with highest/lowest average freight value
```
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
```
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
```
SELECT customer_state, 
ROUND(AVG(freight_value),2) AS mean_freight, 
CEIL(AVG(time_to_delivery)) AS mean_time_to_delivery, 
CEIL(AVG(diff_estimated_delivery)) AS mean_diff_estim_delivery 
FROM vw_delivery_dates AS vw JOIN customers AS c ON vw.customer_id=c.customer_id 
GROUP BY customer_state 
ORDER BY mean_diff_estim_delivery DESC 
LIMIT 5;
```

## Payment type analysis:
#### Month over Month count of orders for different payment types 
```
SELECT payment_type, 
EXTRACT(MONTH FROM order_purchase_timestamp) as months, 
COUNT(o.order_id) as no_orders 
FROM orders AS o JOIN payments AS p ON o.order_id=p.order_id 
GROUP BY payment_type,months 
ORDER BY payment_type,months;
```

### Distribution of payment installments and count of orders 
```
SELECT payment_installments, 
COUNT(o.order_id) as no_orders 
FROM orders AS o JOIN payments AS p ON o.order_id=p.order_id 
GROUP BY payment_installments 
ORDER BY payment_installments;
```

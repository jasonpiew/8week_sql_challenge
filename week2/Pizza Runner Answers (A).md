## Before we begin, let's take a look at our dataset
### It's always a best practice to analyze our dataset before we proceed on
```sql
SELECT *
FROM customer_orders AS co;
SELECT *
FROM runner_orders AS ro 
```

![1](https://user-images.githubusercontent.com/83629572/203703620-dd9ae25d-10eb-4930-ba79-6badf2a1a8a3.png)
![1](https://user-images.githubusercontent.com/83629572/203703702-b5f09a4e-9b74-4521-82ce-0237c9e583a3.png)

## Seems like we have a bit of a dirty data
### Let's clean the customer_orders table first
- clean null string and empty into 0 value in exclusions
- clean null string, empty string and also [NULL] into 0 value in extras
- wrap it inside a view

```sql
CREATE VIEW customer_orders_cleaned AS
SELECT
	order_id,
	customer_id,
	pizza_id,
	CASE
		WHEN exclusions = 'null' THEN '0'
		WHEN exclusions IS NULL THEN '0'
		WHEN exclusions = '' THEN '0'
		ELSE exclusions
	END AS exclusions,
	CASE
		WHEN extras IS NULL THEN '0'
		WHEN extras = 'null' THEN '0'
		WHEN extras = '' THEN '0'
		ELSE extras
	END AS extras,
	order_time
FROM
	customer_orders AS co
```
#### Voila, looks better!
![1](https://user-images.githubusercontent.com/83629572/203704564-9f1e744a-f8e0-40ad-970d-1705b7910d27.png)

### let us clean the runner_orders table
- transform null pickup time and change format of pickup time column to datetime
- clean up distance column, removing KM, clean null and change type into float
- clean up duration column, removing minutes,mins, null
- clean up cancellation column, replace null values to 'no cancellation'
- create a view for the runners_orders cleaned table

```sql
CREATE VIEW runner_orders_cleaned AS
WITH cleaned_runner_order as(
SELECT
	order_id,
	runner_id,
	CASE
		WHEN pickup_time = 'null' THEN NULL
		ELSE pickup_time
	END AS pickup_time,
	CASE
		WHEN distance LIKE 'null' THEN '0'
		WHEN distance ILIKE '%km' THEN trim('km' FROM distance)
		ELSE distance
	END,
	CASE
		WHEN duration = 'null' THEN '0'
		WHEN duration LIKE '%mins' THEN trim('mins' FROM duration)
		WHEN duration LIKE '%minutes' THEN trim('minutes' FROM duration)
		WHEN duration LIKE '%minute' THEN trim('minute' FROM duration)
		ELSE duration
	END,
	CASE
		WHEN cancellation = '' THEN 'No Cancellation'
		WHEN cancellation IS NULL THEN 'No Cancellation'
		WHEN cancellation = 'null' THEN 'No Cancellation'
		ELSE cancellation
	END
FROM
	runner_orders AS ro
)
,cleaned_data_type as(
SELECT ORDER_id,
runner_id,
pickup_time::timestamp,
distance::float,
duration::int,
cancellation
FROM cleaned_runner_order 
)
SELECT *
FROM cleaned_data_type 
;

```

![1](https://user-images.githubusercontent.com/83629572/203708247-aea78714-a75f-4e35-bc2f-ced86f689f61.png)

#### All set, let's get into the data!

# Questions
## Pizza Metrics

1. How many pizzas were ordered?
```sql
SELECT
	count(order_id) AS "Pizza Ordered"
FROM
	customer_orders_cleaned;
```
![1](https://user-images.githubusercontent.com/83629572/203708659-38484da0-6ca9-4984-aa73-b94760eef7dc.png)

2. How many unique customer orders were made?
```sql
SELECT
	count(DISTINCT order_id) AS "Unique Order"
FROM
	customer_orders_cleaned;
 ```
 ![1](https://user-images.githubusercontent.com/83629572/203708848-cc948caa-7871-4e9d-9bdf-6412e190e72a.png)


3. How many successful orders were delivered by each runner?
```sql
SELECT
	count(order_id) AS "Successful Order"
FROM
	runner_orders_cleaned
WHERE
	pickup_time IS NOT NULL;
```
![1](https://user-images.githubusercontent.com/83629572/203709016-30fecbd6-f52d-4816-bfa6-85262a5ffc2a.png)


4. How many of each type of pizza was delivered?
```sql
SELECT
	count(pizza_id),
	pizza_name
FROM
	customer_orders_cleaned AS coc
JOIN pizza_names AS pn
		USING(pizza_id)
JOIN runner_orders_cleaned AS roc
		USING(order_id)
WHERE
	pickup_time IS NOT NULL
GROUP BY
	pizza_name;
```
![1](https://user-images.githubusercontent.com/83629572/203709131-8f84986b-b9c3-476a-ad99-697601cf73b0.png)


5. How many Vegetarian and Meatlovers were ordered by each customer?
```sql
SELECT
	customer_id,
	count(pizza_id),
	pizza_name
FROM
	customer_orders_cleaned AS coc
JOIN pizza_names AS pn
		USING(pizza_id)
JOIN runner_orders_cleaned AS roc
		USING(order_id)
WHERE
	pickup_time IS NOT NULL
GROUP BY
	pizza_name,
	customer_id
ORDER BY
	customer_id ASC;
```
![1](https://user-images.githubusercontent.com/83629572/203710147-b0d8c490-bfb1-4401-af2e-b69e88f64e63.png)


6. What was the maximum number of pizzas delivered in a single order?
```sql
WITH total_order AS(
	SELECT
		order_id,
		count(order_id) AS "Total Order"
	FROM
		customer_orders_cleaned AS coc
	JOIN runner_orders_cleaned AS roc
			USING (order_id)
	WHERE
		pickup_time IS NOT NULL
	GROUP BY
		order_id
)
 SELECT
	max("Total Order") AS "Max ordered"
FROM
	total_order;
```
![1](https://user-images.githubusercontent.com/83629572/203710284-25a9218f-04db-44ca-aab8-1bb8db58823e.png)


7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
```sql
SELECT
	customer_id,
	sum(CASE WHEN exclusions = '0' THEN 1
ELSE 0
END) AS no_change,
	sum(CASE WHEN exclusions != '0' THEN 1
ELSE 0
END) AS at_least_1_change
FROM
	customer_orders_cleaned AS co
JOIN runner_orders_cleaned AS roc
		USING(order_id)
WHERE
	pickup_time IS NOT NULL
GROUP BY
	customer_id
```
![1](https://user-images.githubusercontent.com/83629572/203710667-846eb308-1b2e-438c-bde8-fd54eaffe957.png)


8. How many pizzas were delivered that had both exclusions and extras?
```sql
SELECT
	customer_id,
	sum(CASE WHEN exclusions != '0' AND extras != '0' THEN 1
ELSE 0
END) AS exclusion_and_extras
FROM
	customer_orders_cleaned AS coc
JOIN runner_orders_cleaned AS roc
		USING(order_id)
WHERE
	pickup_time IS NOT NULL
GROUP BY
	customer_id
ORDER BY
	exclusion_and_extras DESC;
```
![1](https://user-images.githubusercontent.com/83629572/203710821-b91ef1f8-2246-4317-b809-d83bba5186c1.png)


9. What was the total volume of pizzas ordered for each hour of the day?
```sql
SELECT
	EXTRACT(
		HOUR
	FROM
		order_time
	) AS HOUR,
	count(*)
FROM
	customer_orders_cleaned AS coc
JOIN runner_orders_cleaned AS roc
		USING(order_id)
WHERE
	pickup_time IS NOT NULL
GROUP BY
	HOUR
ORDER BY
	HOUR ASC;
```
![1](https://user-images.githubusercontent.com/83629572/203710993-f9a58d06-6e2c-4c78-a66e-fb25550711c6.png)

10. What was the volume of orders for each day of the week?
```sql
SELECT
	to_char(order_time, 'Day') AS "Day Name",
	count(*) AS "Total Ordered"
FROM
	customer_orders_cleaned AS coc
JOIN runner_orders_cleaned AS roc
		USING(order_id)
WHERE
	pickup_time IS NOT NULL
GROUP BY
	"Day Name"
```
![1](https://user-images.githubusercontent.com/83629572/203711250-48d3512b-8b19-489a-9af1-0d43289ea388.png)

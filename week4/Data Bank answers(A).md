# Data Bank

Let's give a look at our data

![image](https://user-images.githubusercontent.com/83629572/212333361-85c49388-1129-45e0-a9c5-f9f92a58e7e8.png)

![image](https://user-images.githubusercontent.com/83629572/212333411-9bad2520-03d6-42a9-9784-4c3aa757294e.png)

![image](https://user-images.githubusercontent.com/83629572/212333466-f2a66c8e-26b2-44a5-ab01-b864fa930536.png)

I couldn't show the whole rows, but after I checked, there's no error to be found. We can now proceed.

# Questions
## A. Customer Nodes Exploration
1. How many unique nodes are there on the Data Bank system?
```sql
SELECT
	count(DISTINCT(node_id)) AS totalunique_node
FROM
	customer_nodes AS cn;
```
![image](https://user-images.githubusercontent.com/83629572/212334035-1275380e-d93b-4802-a891-f7b5736f0d55.png)

2. What is the number of nodes per region?
```sql
SELECT
	count(node_id) AS node_count,
	region_name
FROM
	customer_nodes AS cn
LEFT JOIN regions AS r
		USING (region_id)
GROUP BY
	2;
```
![image](https://user-images.githubusercontent.com/83629572/212334145-676d9b25-fb4a-4311-b73b-6ca026add012.png)

3. How many customers are allocated to each region?
```sql
SELECT
	count(DISTINCT customer_id) AS customer_count,
	region_name
FROM
	customer_nodes AS cn
LEFT JOIN regions AS r
		USING (region_id)
GROUP BY
	2;
```
![image](https://user-images.githubusercontent.com/83629572/212334229-017dae2f-f8d3-478b-92b1-e92668d88656.png)

4. How many days on average are customers reallocated to a different node? (sorry I couldn't show all the customer lists)
```sql
SELECT
	 DISTINCT customer_id,
	round(avg(end_date - start_date)OVER(), 2) AS average_day_all_cust,
	round(avg(end_date - start_date)OVER(PARTITION BY customer_id), 2) AS average_day_each_cust
FROM
	customer_nodes AS cn
WHERE
	end_date != '9999-12-31'
ORDER BY
	customer_id ASC;
```
![image](https://user-images.githubusercontent.com/83629572/212334903-bf9b25ba-4489-4f31-8faf-522165c70009.png)

5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?

NOTE: I can't guarantee if this is the correct method of approaching the questions, sorry for the last questions xx!
```sql
WITH value_per_region AS(
	SELECT
		region_id,
		end_date - start_date AS value
		FROM customer_nodes AS cn
	WHERE
		end_date != '9999-12-31'
		--AND region_id = 1
)
,finding_percentile as(
SELECT
	DISTINCT value,
	region_id ,
	percent_rank() OVER(ORDER BY value)* 100 AS percentilerank
FROM
	value_per_region
ORDER BY
	region_id ASC,
	3 DESC
)
,percentile_value as(
SELECT value AS relocation_days,
region_id,
floor(round(percentilerank::numeric,2)) AS percentilerank 
FROM finding_percentile
)
SELECT relocation_days,
region_id,
percentilerank
FROM percentile_value
WHERE percentilerank IN (96,80,49);
```
![image](https://user-images.githubusercontent.com/83629572/212343468-8411088b-11f5-4fe6-8e44-4967f9146754.png)

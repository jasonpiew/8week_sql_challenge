# Foodie-Fi
Let's try to query the data we're working with

![1](https://user-images.githubusercontent.com/83629572/204077277-83bb2e72-281f-454d-876a-94f75f243311.png)
![1](https://user-images.githubusercontent.com/83629572/204077293-d0589ca1-742c-4ea5-a8ec-3d858ae35b89.png)

Seems like there's no problem with the dataset, we can proceed to the questions.

# Questions
1. How many customers has Foodie-Fi ever had?
```sql
SELECT
	count(DISTINCT s.customer_id) AS totalcustomerunique
FROM
	subscriptions AS s
```
![1](https://user-images.githubusercontent.com/83629572/204077391-edf517a6-0804-48cc-af6a-692585d41f15.png)

2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value
```sql
WITH extracted AS(
	SELECT
		EXTRACT( YEAR FROM start_date) AS yy,
		to_char(start_date, 'MM') AS mm,
		to_char(start_date, 'Month') AS mm_name,
		customer_id AS trialplan
	FROM
		subscriptions AS s
	WHERE
		plan_id = 0
)
SELECT
	yy,
	mm,
	mm_name,
	count(trialplan)
FROM
	extracted
GROUP BY
	yy,
	mm,
	mm_name
ORDER BY
	yy ASC,
	mm ASC
```
![1](https://user-images.githubusercontent.com/83629572/204077571-fb9fb44c-ec52-41eb-b811-a028de066487.png)


3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name
```sql
WITH extracted AS(
	SELECT
		EXTRACT( YEAR FROM start_date ) AS yy,
		s.plan_id,
		p.plan_name,
		customer_id AS total
	FROM
		subscriptions AS s
	JOIN "plans" AS p 
ON
		s.plan_id = p.plan_id
)
,
extracted_and_summed AS(
	SELECT
		yy,
		plan_id,
		plan_name,
		count(total) AS totalplan
	FROM
		extracted
	GROUP BY
		1,
		2,
		3
)
,
extracted_and_summed_partitioned AS (
	SELECT
		*,
		sum(totalplan)OVER(
			PARTITION BY yy
		) AS totalperyear
	FROM
		extracted_and_summed
)
SELECT
	*,
	round(totalplan::NUMERIC / totalperyear * 100, 2) AS percentage_differ
FROM
	extracted_and_summed_partitioned
WHERE
	yy = '2021';
```
![1](https://user-images.githubusercontent.com/83629572/204077675-5bdd5d42-1a3b-442b-b4b5-577a1e16a679.png)


4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?
```sql
WITH customer_that_churned AS(
	SELECT
		customer_id,
		s.plan_id,
		p.plan_name
	FROM
		subscriptions AS s
	JOIN "plans" AS p 
ON
		s.plan_id = p.plan_id
	WHERE
		s.plan_id = 4
)
,
customer_that_churned_summed AS(
	SELECT
		plan_id,
		plan_name,
		count(*) AS customerthatchurned
	FROM
		customer_that_churned
	GROUP BY
		1,
		2
)
SELECT
	*,
	(
		SELECT
			count(DISTINCT customer_id)
			FROM subscriptions
	) AS totalcustomer,
	100 * customerthatchurned::NUMERIC /(
		SELECT
			count(DISTINCT customer_id)
			FROM subscriptions
	) AS percentage
FROM
	customer_that_churned_summed;
```
![1](https://user-images.githubusercontent.com/83629572/204077796-67828d07-32d5-44d6-afa1-216316ce6a11.png)

5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?
```sql
--using lead to find customer that churned after free trial
WITH customer_churned_afterfreetrial AS (
	SELECT
		customer_id,
		p.plan_name,
		LEAD(p.plan_name, 1)OVER(
			PARTITION BY customer_id
		ORDER BY
			start_date
		) AS nextplan
	FROM
		subscriptions AS s
	JOIN "plans" AS p 
ON
		s.plan_id = p.plan_id
)
--filter customer that churned after trial
SELECT
	count(DISTINCT customer_id) AS trial_then_churn,
	count(DISTINCT customer_id)::NUMERIC /(
		SELECT
			count(DISTINCT customer_id)
			FROM subscriptions
	)* 100 AS percentagechurn
FROM
	customer_churned_afterfreetrial
WHERE
	plan_name = 'trial'
	AND nextplan = 'churn'
```
![1](https://user-images.githubusercontent.com/83629572/204077932-e57c7da8-2970-4ab9-8b69-a825c62ac10a.png)

6. What is the number and percentage of customer plans after their initial free trial?
```sql
WITH customernextplan AS (
	SELECT
		customer_id,
		p.plan_name,
		LEAD(p.plan_name, 1)OVER(
			PARTITION BY customer_id
		ORDER BY
			start_date
		) AS nextplan
	FROM
		subscriptions AS s
	JOIN "plans" AS p 
ON
		s.plan_id = p.plan_id
)
SELECT
	nextplan AS nextplanaftertrial,
	count(DISTINCT customer_id) AS totalplancount,
	100 * count(DISTINCT customer_id)::NUMERIC /(
		SELECT
			count(DISTINCT customer_id)
			FROM subscriptions
	) AS percentagechurn
FROM
	customernextplan
WHERE
	plan_name = 'trial'
GROUP BY
	1
```
![1](https://user-images.githubusercontent.com/83629572/204078095-ba3697d9-a232-4e95-a012-4d36b829da32.png)

7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?
```sql
WITH customerplanandyearfilter AS (
	SELECT
		customer_id,
		s.plan_id,
		p.plan_name,
		s.start_date,
		LEAD(s.plan_id, 1)OVER(
			PARTITION BY customer_id
		ORDER BY
			start_date
		) AS startnextplan
	FROM
		subscriptions AS s
	JOIN "plans" AS p 
ON
		s.plan_id = p.plan_id
	WHERE
		EXTRACT(
			YEAR
		FROM
			start_date
		)= 2020
)
,
customerplanandyearfilter_summed AS(
	SELECT
		count(customer_id) AS totalofplan,
		plan_id
	FROM
		customerplanandyearfilter
	WHERE
		startnextplan IS NULL
		--meaning this is the latest plan of their subscription
	GROUP BY
		2
);

SELECT
	plan_id,
	totalofplan,
	100 * totalofplan::NUMERIC /(
		SELECT
			count(DISTINCT customer_id)
			FROM subscriptions
	) AS percentage
FROM
	customerplanandyearfilter_summed
GROUP BY
	1,
	2;
```
![1](https://user-images.githubusercontent.com/83629572/204078229-b2dfef74-10b0-4e24-b51e-f11c73f2a7bd.png)

8. How many customers have upgraded to an annual plan in 2020?
```sql
SELECT
	count(DISTINCT customer_id) AS annualplancustomer
FROM
	subscriptions AS s
WHERE
	plan_id = 3
	AND EXTRACT(
		YEAR
	FROM
		start_date
	) = 2020
```
![1](https://user-images.githubusercontent.com/83629572/204078268-77669f5f-4bea-4f47-9912-3be5f5ff08cd.png)

9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?
```sql
WITH start_date_trial AS(
	SELECT
		customer_id,
		start_date
	FROM
		subscriptions
	WHERE
		plan_id = 0
)
,
annual_date AS(
	SELECT
		customer_id,
		start_date AS annual_date_plan
	FROM
		subscriptions
	WHERE
		plan_id = 3
)
SELECT
	round(avg(annual_date_plan-start_date),2) AS average_taken
FROM
	start_date_trial sdt
JOIN annual_date ad 
ON
	sdt.customer_id = ad.customer_id
```
![1](https://user-images.githubusercontent.com/83629572/204078397-cbe6501c-c7f6-4dbd-acec-fd37dd213019.png)

10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)
```sql
WITH start_date_trial AS(
	SELECT
		customer_id,
		start_date AS date_trial
	FROM
		subscriptions
	WHERE
		plan_id = 0
)
,
annual_date AS(
	SELECT
		customer_id,
		start_date AS annual_date_plan
	FROM
		subscriptions
	WHERE
		plan_id = 3
)
,
--sort value into 10 bins
daygap_bucket AS (
	SELECT
		WIDTH_BUCKET(ad.annual_date_plan - sdt.date_trial, 0, 300, 10) AS daygap
	FROM
		start_date_trial sdt
	JOIN annual_date ad 
ON
		sdt.customer_id = ad.customer_id
)
SELECT
	concat((daygap -1) * 30 , '-',(daygap) * 30, ' days') AS timebreakdown,
	count(*) customercount
FROM
	daygap_bucket
GROUP BY
	1,
	daygap
ORDER BY
	daygap ASC
```
![1](https://user-images.githubusercontent.com/83629572/204078588-adedbf68-14b8-4e33-90ce-3b943d1a7d89.png)

11.How many customers downgraded from a pro monthly to a basic monthly plan in 2020?
```sql
WITH downgraded AS (
	SELECT
		*,
		LEAD(plan_id, 1)OVER(
			PARTITION BY customer_id
		ORDER BY
			start_date
		) AS nextplan
	FROM
		subscriptions
	WHERE
		EXTRACT(
			YEAR
		FROM
			start_date
		)= 2020
)
SELECT
	count(*) AS downgrade_count
FROM
	downgraded
WHERE
	nextplan = 1
	AND plan_id = 2
```

![1](https://user-images.githubusercontent.com/83629572/204078838-e27ce36c-1e35-41bc-b866-460ac434836a.png)

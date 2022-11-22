# Answers 

1. What is the total amount each customer spent at the restaurant?
```sql
SELECT
	s2.customer_id,
	sum(mm.price) AS totalspent
FROM
	sales AS s2
INNER JOIN menu AS mm
ON
	mm.product_id = s2.product_id
GROUP BY
	s2.customer_id
ORDER BY
	customer_id
```
![1](https://user-images.githubusercontent.com/83629572/203220416-d3af9c10-5302-4705-b081-9796948e80c9.png)

2. How many days has each customer visited the restaurant?
```sql
SELECT
	s.customer_id,
	count(s.customer_id)
FROM
	sales AS s
GROUP BY
	s.customer_id
 ```
 ![1](https://user-images.githubusercontent.com/83629572/203220718-7b51ccfe-8d89-4f0c-a540-c6baf91f9801.png)

3. What was the first item from the menu purchased by each customer?
```sql
SELECT
	DISTINCT(customer_id),
	product_name
FROM
	sales s
JOIN menu m 
ON
	m.product_id = s.product_id
WHERE
	s.order_date = ANY 
      (
		SELECT
			min(order_date)
		FROM
			sales
		GROUP BY
			customer_id
	)
ORDER BY
	customer_id
```
![1](https://user-images.githubusercontent.com/83629572/203220946-659ac625-349c-4283-99df-61bc03578873.png)


4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT
	m.product_name,
	count(m.product_name)
FROM
	menu AS m
INNER JOIN sales AS s 
ON
	m.product_id = s.product_id
GROUP BY
	m.product_name
LIMIT 1
```
![1](https://user-images.githubusercontent.com/83629572/203221099-2accfe79-e413-4d75-bddf-8556f6187d36.png)

5. Which item was the most popular for each customer?
```sql
WITH t1 AS(
	SELECT
		s.customer_id,
		m.product_name,
		count(m.product_id),
		DENSE_RANK() OVER(
			PARTITION BY s.customer_id
		ORDER BY
			count(m.product_id) DESC
		) AS rn
	FROM
		sales AS s
	INNER JOIN menu AS m 
ON
		s.product_id = m.product_id
	GROUP BY
		1,
		2
)
SELECT
	customer_id ,
	product_name ,
	count
FROM
	t1
WHERE
	rn = 1;
```
![1](https://user-images.githubusercontent.com/83629572/203221344-9e41d4ec-35d5-4ba5-bbab-e897e1ee50da.png)

6. Which item was purchased first by the customer after they became a member?
```sql
WITH cust_purchase AS(
	SELECT
		m.customer_id,
		product_name,
		join_date,
		order_date,
		DENSE_RANK() OVER(
			PARTITION BY customer_id
		ORDER BY
			order_date ASC
		) AS first_purchase
	FROM
		members AS m
	JOIN sales AS s
			USING(customer_id)
	JOIN menu AS m2 
ON
		s.product_id = m2.product_id
	WHERE
		s.order_date >= m.join_date
)
SELECT
	customer_id,
	product_name,
	join_date,
	order_date
FROM
	cust_purchase
WHERE
	first_purchase = 1
 ```
 ![1](https://user-images.githubusercontent.com/83629572/203222440-d5d9985f-e98f-40f2-8f96-787344faefca.png)


7. Which item was purchased just before the customer became a member?
```sql
WITH cust_purchase AS(
	SELECT
			s.customer_id,
			m.product_name,
			s.order_date,
			DENSE_RANK() OVER(
				PARTITION BY s.customer_id
		ORDER BY
				s.order_date
		) AS RANK,
			m2.join_date
	FROM
			sales AS s
	JOIN menu AS m 
ON
			s.product_id = m.product_id
	JOIN members AS m2 
ON
			m2.customer_id = s.customer_id
	WHERE
			s.order_date <= m2.join_date
)
SELECT
	customer_id,
	product_name,
	order_date,
	join_date
FROM
	cust_purchase
WHERE
	RANK = 1;
```
![1](https://user-images.githubusercontent.com/83629572/203222753-8b667956-251a-412f-849c-6cd78deddb88.png)

8. What is the total items and amount spent for each member before they became a member?
```sql
SELECT
	m2.customer_id,
	count(m.product_name),
	sum(m.price) AS amountspent
FROM
	sales AS s
JOIN menu AS m 
ON
	s.product_id = m.product_id
JOIN members AS m2 
ON
	m2.customer_id = s.customer_id
WHERE
	s.order_date < m2.join_date
GROUP BY
	m2.customer_id
ORDER BY
	customer_id
```
![1](https://user-images.githubusercontent.com/83629572/203223601-914647c6-1b3f-4b22-b030-679da66fdd27.png)

9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```sql
SELECT
	customer_id,
	sum(point)
FROM
	(
		SELECT
			s.customer_id,
			CASE
				WHEN m.product_name = 'sushi' THEN price * 20
				WHEN m.product_name != 'sushi' THEN price * 10
			END AS point
		FROM
			sales AS s
		JOIN menu AS m 
ON
			s.product_id = m.product_id
	)t1
GROUP BY
	1
```
![1](https://user-images.githubusercontent.com/83629572/203223760-c4e58bc1-399c-4bdc-9c22-5642f919c728.png)

10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```sql
WITH time_limit AS (
	SELECT
		customer_id,
		join_date,
		join_date + INTERVAL '7 day' AS first_week
	FROM
		members AS m
)
,
cust_purchase AS(
	SELECT
		s.customer_id,
		order_date,
		join_date,
		first_week::date,
		CASE
			WHEN order_date BETWEEN join_date AND first_week THEN price * 20
			WHEN product_name = 'sushi' THEN price * 20
			ELSE price * 10
		END AS NEW_price,
		product_name,
		price
	FROM
		sales AS s
	JOIN menu AS m
			USING(product_id)
	JOIN time_limit AS t
			USING (customer_id)
	WHERE
		order_date <= '2022-02-01'
)
SELECT
	customer_id,
	sum(new_price) AS total_point
FROM
	cust_purchase
GROUP BY
	1
```
![1](https://user-images.githubusercontent.com/83629572/203227182-38d36c74-b8ec-442c-a969-d918395594ad.png)

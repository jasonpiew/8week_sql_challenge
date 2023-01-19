# B. Customer Transactions
1. What is the unique count and total amount for each transaction type?
```sql
SELECT
	txn_type,
	count(DISTINCT txn_amount) AS distinct_count,
	sum(txn_amount) AS amount_sum
FROM
	customer_transactions AS ct
GROUP BY
	1;
```
![image](https://user-images.githubusercontent.com/83629572/213355468-eb89243a-d84c-49b3-bb93-3d116f530615.png)

2. What is the average total historical deposit counts and amounts for all customers?
```sql
SELECT
	avg(txn_amount) AS average_amount,
	count(customer_id)/(
		SELECT
			count(DISTINCT customer_id)
		FROM
			customer_transactions AS ct
	) AS average_deposit
FROM
	customer_transactions AS ct2
WHERE
	txn_type = 'deposit';
```
![image](https://user-images.githubusercontent.com/83629572/213355554-d147b879-d32a-4c7d-8869-ace418d4934c.png)

3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
```sql
WITH customer_purchase AS(
	SELECT
		customer_id,
		sum(CASE WHEN txn_type = 'deposit' THEN 1 ELSE 0 END) AS total_deposit,
		sum(CASE WHEN txn_type = 'purchase' OR txn_type = 'withdrawal' THEN 1 ELSE 0 END) AS total_withdrawal_or_purchase,
		EXTRACT(
			MONTH
		FROM
			txn_date
		) AS mm
	FROM
		customer_transactions AS ct
	GROUP BY
		1,
		4
)
SELECT
	count(*) total_customer,
	mm
FROM
	customer_purchase
WHERE
	total_deposit > 1
	AND total_withdrawal_or_purchase >= 1
GROUP BY
	2
ORDER BY
mm ASC;
```
![image](https://user-images.githubusercontent.com/83629572/213355635-060efeb0-6830-4e32-84f1-e17a9a1c32cf.png)

4. What is the closing balance for each customer at the end of the month?
```sql
WITH cust_trans AS(
	SELECT
		customer_id,
		EXTRACT(
			MONTH
		FROM
			txn_date
		) AS mm,
		txn_type,
		sum(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) AS transaction_amount
	FROM
		customer_transactions AS ct2
	GROUP BY
		customer_id ,
		mm ,
		txn_amount ,
		txn_type
)
,
customer_final_balance AS (
	SELECT
		customer_id,
		mm,
		txn_type,
		transaction_amount,
		sum(transaction_amount)OVER(
			PARTITION BY customer_id
		ORDER BY
			mm ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
		) AS final_amount
	FROM
		cust_trans
)
,
finding_last_value AS(
	SELECT
		customer_id,
		transaction_amount,
		final_amount,
		mm,
		LAST_VALUE(final_amount)OVER(
			PARTITION BY customer_id,
			mm
		)
	FROM
		customer_final_balance
)
SELECT
	DISTINCT customer_id,
	LAST_VALUE,
	mm
FROM
	finding_last_value
ORDER BY
	customer_id ASC
,
	mm ASC;
```
![image](https://user-images.githubusercontent.com/83629572/213356963-7cfde467-d496-4f4c-9d85-a76ed4b52a84.png)

5. What is the percentage of customers who increase their closing balance by more than 5%?
```sql
Working on it
```

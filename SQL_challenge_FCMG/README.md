#  Challenge: Provide Insights to Management in Consumer Goods Domain

## Problem statement
Atliq Hardwares (imaginary company) is one of the leading computer hardware producers in India and well expanded in other countries too.

However, the management noticed that they do not get enough insights to make quick and smart data-informed decisions. They want to expand their data analytics team by adding several junior data analysts. Tony Sharma, their data analytics director wanted to hire someone who is good at both tech and soft skills. Hence, he decided to conduct a SQL challenge which will help him understand both the skills.


challenge [Link](https://codebasics.io/challenge/codebasics-resume-project-challenge)

---

## Requests and solution

**1. Provide the list of markets in which customer "Atliq Exclusive" operates its
business in the APAC region.**

```sql
SELECT market FROM dim_customer
WHERE customer = 'Atliq Exclusive' AND region = 'APAC';

```
Results:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/d67e1b1e2c4814df98fd74b06cdec41bba7722b2/SQL/Challenge/Q1.png" width=10% height=10%>


**2. What is the percentage of unique product increase in 2021 vs. 2020? The final output contains these fields,\
        unique_products_2020 | unique_products_2021 | percentage_chg**

```sql
WITH c2020 AS(
	SELECT product_code, count(distinct(product_code)) as  unique_products_2020
	FROM fact_sales_monthly
	WHERE fiscal_year = 2020),
c2021 AS(
	SELECT product_code, count(distinct(product_code)) as  unique_products_2021
	FROM fact_sales_monthly
    WHERE fiscal_year = 2021)
SELECT 
	unique_products_2020,
    unique_products_2021,
    (unique_products_2021-unique_products_2020)/unique_products_2020*100 AS percentage_chg
FROM c2020
JOIN c2021
USING (product_code);

```
Results:


<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/d67e1b1e2c4814df98fd74b06cdec41bba7722b2/SQL/Challenge/Q2.png" width=45% height=45%>

**3. Provide a report with all the unique product counts for each segment and sort them in descending order of product counts. The final output contains
2 fields,\
segment | product_count**

```sql
SELECT segment, count(distinct(product_code)) AS product_count
FROM dim_product
GROUP BY segment
ORDER BY product_count DESC;
```
Results:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/d67e1b1e2c4814df98fd74b06cdec41bba7722b2/SQL/Challenge/Q3.png" width=20% height=20%>


**4. Follow-up: Which segment had the most increase in unique products in 2021 vs 2020? The final output contains these fields,
segment | product_count_2020 | product_count_2021 | difference**

```sql
WITH sc2020 AS(
	SELECT p.segment AS segment, count(distinct(s.product_code)) AS product_count_2020
	FROM fact_sales_monthly s
    JOIN dim_product p
    ON s.product_code = p.product_code
	WHERE fiscal_year = 2020
	GROUP BY p.segment),
sc2021 AS(
	SELECT p.segment AS segment, count(distinct(s.product_code)) AS product_count_2021
	FROM fact_sales_monthly s
    JOIN dim_product p
    ON s.product_code = p.product_code
	WHERE fiscal_year = 2021
	GROUP BY p.segment)
SELECT 
	sc2020.segment, product_count_2020, product_count_2021,
	product_count_2021-product_count_2020 AS difference
FROM sc2020 
JOIN sc2021
USING (segment)
ORDER BY difference DESC;
```

Results:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/d67e1b1e2c4814df98fd74b06cdec41bba7722b2/SQL/Challenge/Q4.png" width=50% height=50%>


**5. Get the products that have the highest and lowest manufacturing costs. The final output should contain these fields,
product_code | product | manufacturing_cost**

```sql
SELECT 
	p.product_code, p.product, m. manufacturing_cost
FROM fact_manufacturing_cost m
JOIN dim_product p
ON m.product_code = p. product_code
WHERE manufacturing_cost in
((SELECT max(manufacturing_cost)FROM fact_manufacturing_cost), (SELECT min(manufacturing_cost) FROM fact_manufacturing_cost))
ORDER BY manufacturing_cost DESC;

```
Results:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/d67e1b1e2c4814df98fd74b06cdec41bba7722b2/SQL/Challenge/Q5.png" width=40% height=40%>




**6. Generate a report which contains the top 5 customers who received an average high pre_invoice_discount_pct for the fiscal year 2021 and in the
Indian market. The final output contains these fields,
customer_code | customer | average_discount_percentage**

```sql
SELECT c.customer_code,c.customer, avg(pre_invoice_discount_pct) AS average_discount_percentage
FROM  fact_pre_invoice_deductions p
JOIN dim_customer c
ON p.customer_code = c. customer_code
WHERE fiscal_year = 2021 AND c. market = 'india'
GROUP BY customer_code
ORDER BY average_discount_percentage DESC
LIMIT 5;
```
Results:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/d67e1b1e2c4814df98fd74b06cdec41bba7722b2/SQL/Challenge/Q6.png" width=35% height=35%>



**7. Get the complete report of the Gross sales amount for the customer “Atliq Exclusive” for each month. This analysis helps to get an idea of low and
high-performing months and take strategic decisions. The final report contains these columns:
Month | Year | Gross sales Amount**

```sql
SELECT 
    MONTH(s.date) AS month,
    YEAR (s.date) AS year,
    ROUND(sum(s.sold_quantity* g.gross_price),2)  AS gross_sales_amount
FROM fact_sales_monthly s
JOIN fact_gross_price g
ON s.product_code = g.product_code
JOIN dim_customer c
ON s.customer_code= c. customer_code
WHERE c.customer = 'Atliq Exclusive'
GROUP BY MONTH(s.date) , YEAR (s.date);
```
Results:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/d67e1b1e2c4814df98fd74b06cdec41bba7722b2/SQL/Challenge/q7.png" width=20% height=20%>


**8. In which quarter of 2020, got the maximum total_sold_quantity? The final output contains these fields sorted by the total_sold_quantity,
Quarter
total_sold_quantity**

(1) Create a function for get_fiscal_quarter
```SQL
CREATE DEFINER=`root`@`localhost` FUNCTION `get_fiscal_quarter`(calendar_date DATE) RETURNS char(2) CHARSET utf8mb4
    DETERMINISTIC
BEGIN
DECLARE quarter CHAR(2);
SET quarter = 
	IF (MONTH(calendar_date) IN (9,10,11), 'Q1',
		IF(MONTH(calendar_date) IN (12,1,2), 'Q2',
			IF(MONTH(calendar_date) IN (3,4,5), 'Q3','Q4')));
RETURN quarter;
END;
```

(2)
```sql
SELECT 
	get_fiscal_quarter(date) AS Quarter, 
	sum(sold_quantity) AS total_sold_quantity
FROM fact_sales_monthly
WHERE fiscal_year = 2020
GROUP BY Quarter
ORDER BY total_sold_quantity DESC;
```
Results:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/d67e1b1e2c4814df98fd74b06cdec41bba7722b2/SQL/Challenge/Q8.png" width=20% height=20%>




**9. Which channel helped to bring more gross sales in the fiscal year 2021 and the percentage of contribution? The final output contains these fields,
channel | gross_sales_mln | percentage**


```sql
WITH ctecgs AS(
SELECT 
    c.channel,
    ROUND(sum(s.sold_quantity* g.gross_price),2)/1000000  AS gross_sales_mln
FROM fact_sales_monthly s
JOIN fact_gross_price g
ON s.product_code = g.product_code
JOIN dim_customer c
ON s.customer_code= c. customer_code
WHERE s.fiscal_year = 2021
GROUP BY c.channel)
SELECT *,
	ROUND(gross_sales_mln*100/sum(gross_sales_mln) over() ,2) AS percentage
FROM ctecgs
GROUP BY channel
ORDER BY percentage DESC;
```

Results:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/d67e1b1e2c4814df98fd74b06cdec41bba7722b2/SQL/Challenge/Q9%20.png" width=30% height=30%>


**10. Get the Top 3 products in each division that have a high total_sold_quantity in the fiscal_year 2021? The final output contains these fields,
division | product_code | product | total_sold_quantity | rank_order**

```sql
WITH cteds AS(
SELECT 
    p.division,p.product_code, p.product,
    sum(sold_quantity)  AS total_sold_quantity
FROM fact_sales_monthly s
JOIN dim_product p
ON s.product_code = p.product_code
JOIN dim_customer c
ON s.customer_code= c. customer_code
WHERE s.fiscal_year = 2021
GROUP BY p.division, p.product, p.product_code),
cteds2 AS(
	SELECT *,
		dense_rank() over (partition by division order by total_sold_quantity desc) AS rank_order
	FROM cteds)
SELECT * FROM cteds2
WHERE rank_order<=3
ORDER BY total_sold_quantity DESC
```

Results:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/d67e1b1e2c4814df98fd74b06cdec41bba7722b2/SQL/Challenge/Q10.png" width=50% height=50%>

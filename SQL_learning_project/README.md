## Problem statement
AtliQ Hardware manufactures and distributes computer peripherals, including networking devices, laptops, and mouse..., to a global market (APAC,EU,NA,LATAM). The company supplies products to retailers, e-commerce platforms, and distributors. Additionally, AtliQ has its own retail channels, such as AtliQ Exclusive and AtliQ E-store. The company aims to gain deeper sales insights and make accurate data-driven decisions through data analysis techniques.



## 🗂Outline

- [Dataset introduction](#dataset-introduction)
	- [Original Dataset](#original-dataset)
	- [Data handling layout](#data-handling-layout)
	- [Entity-Releationship Diagrams (ERD)](#entity-releationship-diagrams-erd)
- [P&L Statement Data and Metrics preparation introduction](#pl-statement-data-and-metrics-preparation-introduction)
- [Supply chain Data and Metrics preparation introduction](#supply-chain-data-and-metrics-preparation-introduction)

## Dataset introduction


#### Original Dataset

 dim_customers | dim_product\
 fact_freight_cost | fact_gross_cost | fact_gross_cost\
 fact_pre_invoice_deduction | fact_post_invoice_deduction\
 **fact_forecast_monthly** (Over 1.8 million record)\
 **fact_sales_monthly** (Over 1.4 million record)

<kbd>  <img src="https://github.com/HsiaoChuHao/picture/blob/c63f642dce1751f816ead8411f0fe8d4369e5b53/SQL/original%20table.png" width=60% height=60%>

#### Data handling layout
<kbd>  <img src="https://github.com/HsiaoChuHao/picture/blob/c63f642dce1751f816ead8411f0fe8d4369e5b53/SQL/table%20relationship.png" width=60% height=60%>

#### Entity-Releationship Diagrams (ERD)
<kbd>  <img src="https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/ERD.png" width=75% height=75%>


---

## P&L Statement Data and Metrics preparation introduction

**1. creste SQL user define function for get fiscal year**
   
```mysql 
CREATE DEFINER=`root`@`localhost` FUNCTION `get_fiscal_year`(calendar_date DATE) RETURNS int
    DETERMINISTIC
BEGIN
	DECLARE fiscal_year INT;
    SET fiscal_year = YEAR(DATE_ADD(calendar_date, INTERVAL 4 MONTH));
    RETURN fiscal_year;
END;
```

**2. To calculate gross price and total gross price 
by using fact_sales_monthly combined fact_gross_price and dim_product**

```mysql
SELECT 
s.date, s.product_code, 
p.product, p.variant, s.sold_quantity,g.fiscal_year, g.gross_price,
ROUND(g.gross_price*s.sold_quantity,2) AS gross_price_total
FROM fact_sales_monthly s
JOIN dim_product p
ON s.product_code = p.product_code
JOIN fact_gross_price g
ON 
	s.product_code = g.product_code AND 
	g.fiscal_year = get_fiscal_year(s. date)
WHERE 
	customer_code = 90002002 AND 
	get_fiscal_year(date) = 2021   
ORDER BY date ASC;
```
Result:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/learning_project/P%26L/2.png" width=70% height=70%>




**3-1. Sum up for total gross sales for certain customer(customer_code = 90002002)**
```mysql
SELECT 
s.date, s.product_code, s.sold_quantity,get_fiscal_year(s.date) AS FY, g.gross_price, 
SUM(ROUND(g.gross_price*s.sold_quantity,2)) AS gross_price_total
FROM fact_sales_monthly s
JOIN fact_gross_price g
ON 
	s.product_code = g.product_code AND
	g.fiscal_year = get_fiscal_year(s.date)
WHERE customer_code = 90002002
GROUP BY s.date
ORDER BY s.date ASC;
```
Result:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/learning_project/P%26L/3-1.png" width=70% height=70%>



**3-2. Create the stored procedures to get monthly_gross_sales**

```mysql
CREATE DEFINER=`root`@`localhost` PROCEDURE `customer_monthly_gross_sales`(
in_customer_code TEXT)
BEGIN
	SELECT 
	s.date, 
	SUM(ROUND(g.gross_price*s.sold_quantity,2)) AS gross_price_total
	FROM fact_sales_monthly s
	JOIN fact_gross_price g
	ON 
		s.product_code = g.product_code AND
		g.fiscal_year = get_fiscal_year(s.date)
	WHERE find_in_set(s.customer_code, in_customer_code)>0
	GROUP BY s.date
	ORDER BY s.date ASC;
END;
```
Result:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/learning_project/P%26L/3-2.png" width=60% height=60%>



**4. Calculate the Net_invoice_sales by combine\
fact_sales_monthly, dim_product, fact_gross_price, fact_pre_invoice_deductions**
**(Create in Views for convenient use)**

```mysql
WITH CTE1 AS(
SELECT 
	s.date,s.product_code,
    p.product, p.variant, s.sold_quantity,
	g.gross_price as gross_price_per_item,
	ROUND(g.gross_price*s.sold_quantity,2) AS gross_price_total,
	pre.pre_invoice_discount_pct
FROM fact_sales_monthly s
JOIN dim_product p
	ON s.product_code = p.product_code
JOIN fact_gross_price g
	ON 
		s.product_code = g.product_code AND
		g.fiscal_year = s. fiscal_year
JOIN fact_pre_invoice_deductions pre
	ON 
		pre.customer_code = s.customer_code AND
		pre.fiscal_year = s. fiscal_year
WHERE s. fiscal_year = 2021
LIMIT 1000000)
SELECT 
*, gross_price_total*(1-pre_invoice_discount_pct) AS Net_invoice_sales
FROM CTE1;
```
Result:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/learning_project/P%26L/4.png" width=70% height=70%>



**5. Calulate the total_post_invoice_deduction_pct (discounts_pct + other_deductions_pct)**
**(Create in Views for convenient use)**

```mysql
CREATE VIEW `sales_postinv_discount` AS
SELECT 
		s.date, s.fiscal_year,
		s.customer_code, s.market,
		s.product_code, s.product, s.variant,
		s.sold_quantity, s.gross_price_total,
		s.pre_invoice_discount_pct,
		(s.gross_price_total-s.pre_invoice_discount_pct*s.gross_price_total) as net_invoice_sales,
		(po.discounts_pct+po.other_deductions_pct) as post_invoice_discount_pct
FROM sales_preinv_discount s
JOIN fact_post_invoice_deductions po
	ON po.customer_code = s.customer_code AND
	po.product_code = s.product_code AND
	po.date = s.date;
```
Result:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/learning_project/P%26L/5.png" width=70% height=70%>

**6. Calulate the net sales**
**(Create in Views for convenient use)**

```mysql
SELECT 
		*, 
		net_invoice_sales*(1-total_post_invoice_deduction_pct) as net_sales
FROM sales_postinv_discount;
```
Result:

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/learning_project/P%26L/6.png" width=70% height=70%>

**7. Top5 market in 2021 by net sales**
**(Create in stored procedures for convenient use)**

```mysql
SELECT 
market,
ROUND(SUM(s.net_sales)/1000000,2) as net_sales_mln
FROM sales_net_sales s
WHERE s.fiscal_year = 2021
GROUP BY market
ORDER BY net_sales_mln DESC
LIMIT 5;
```
Result:

[](https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/learning_project/P%26L/7.png)

```mysql
CREATE DEFINER=`root`@`localhost` PROCEDURE `get_top_n_markets_by_net_sales`(
	in_fiscal_year int,
    in_top_n INT
)
BEGIN
	SELECT 
	market,
	ROUND(SUM(s.net_sales)/1000000,2) as net_sales_mln
	FROM sales_net_sales s
	WHERE s.fiscal_year = in_fiscal_year
	GROUP BY market
	ORDER BY net_sales_mln DESC
	LIMIT in_top_n;
END;
```
<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/learning_project/P%26L/7-1.png" width=50% height=50%>\
<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/learning_project/P%26L/7-1-1.png" width=50% height=50%>

**8. Find net sales distibution by region and customer for FY 2021**

```mysql
WITH cte1 AS(
SELECT 
	c.customer, c.region,
	ROUND(SUM(s.net_sales)/1000000,2) as net_sales_mln
FROM sales_net_sales s
JOIN dim_customer c
ON s.customer_code = c.customer_code
WHERE s.fiscal_year = 2021
GROUP BY c.region, c.customer )
SELECT 
	*, 
    net_sales_mln*100/sum(net_sales_mln)
    over(partition by region) as net_sales_pct
FROM cte1
order by region,net_sales_mln DESC;
```
Result: 

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/learning_project/P%26L/8.png" width=90% height=90%>


---


## Supply chain Data and Metrics preparation introduction

**1.create table fact_act_est by outer join from fact_sales_monthly and fact_forecast_monthly and update null to 0**

```sql
drop table if exists fact_act_est;
create table fact_act_est 
(
select 
	s.date as date, 
    s.fiscal_year as fiscal_year, 
    s. product_code  as product_code,
    s.customer_code as customer_code,
    s.sold_quantity as sold_quantity,
	f. forecast_quantity as forecast_quantity
FROM fact_sales_monthly s
LEFT JOIN fact_forecast_monthly f
USING (date,customer_code,product_code)
UNION
select 
	f.date as date,
    f.fiscal_year as fiscal_year,
    f. product_code as  product_code,
    f.customer_code as customer_code,
    s.sold_quantity as sold_quantity,
	f. forecast_quantity as forecast_quantity
FROM  fact_forecast_monthly f 
LEFT JOIN fact_sales_monthly s
USING (date,customer_code,product_code));
```
```sql
update fact_act_est
set sold_quantity = 0
where  sold_quantity is null;
```
```sql
update fact_act_est
set forecast_quantity = 0
where  forecast_quantity is null;
```

Result: 

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/learning_project/SC/2-1.png" width=50% height=50%>


**2. create the trigger to automatically insert record in fact_act_est table whenever insertion happens in fact_sales_monthly and fact_forecast_monthly**

```sql
CREATE DEFINER=`root`@`localhost` TRIGGER `fact_sales_monthly_AFTER_INSERT` AFTER INSERT ON `fact_sales_monthly` FOR EACH ROW BEGIN
	insert into fact_act_est
		(date, product_code, customer_code, sold_quantity)
    values(
		NEW.date, 
        NEW.product_code,
        NEW.customer_code,
        NEW.sold_quantity)
    on duplicate key update
		sold_quantity = values(sold_quantity);
END;

CREATE DEFINER=`root`@`localhost` TRIGGER `fact_forecast_monthly_AFTER_INSERT` AFTER INSERT ON `fact_forecast_monthly` FOR EACH ROW BEGIN
	insert into fact_act_est
		(date, product_code, customer_code, forecast_quantity)
    values(
		NEW.date, 
        NEW.product_code, 
        NEW.customer_code, 
        NEW.forecast_quantity
	)
    on duplicate key update
		forecast_quantity = values(forecast_quantity);
END;
```

**3. calulate the net_error, net_error%, abs_error, abs_error%, forecast_accuracy**
**(Create in stored procedures for convenient use)**

```sql
with ctef AS(
	select 
		*,
		sum(forecast_quantity-sold_quantity) AS net_err,
		(sum(forecast_quantity-sold_quantity))/sum(forecast_quantity)*100 AS net_err_pct,
		sum(abs(forecast_quantity-sold_quantity)) AS abs_err,
		sum(abs(forecast_quantity-sold_quantity))/sum(forecast_quantity)*100 AS abs_err_pct
	from fact_act_est f
	where fiscal_year = 2021
	group by customer_code)
select 
	e.customer_code, c.customer, c.market, 
    sum(e.sold_quantity) AS total_sold_qty,
    sum(e.forecast_quantity) AS total_forecast_qty,
    e. net_err, e.net_err_pct, e.abs_err, e.abs_err_pct,
	round(IF(abs_err_pct>100,0,100.0-abs_err_pct),1) AS forecast_accuracy
from ctef e
join dim_customer c
USING (customer_code)
group by customer_code
order by forecast_accuracy desc;
```
Result: 

<kdb> <img src="https://github.com/HsiaoChuHao/picture/blob/057bb29bebff933baccebcc2cdf019f388b430f6/SQL/learning_project/SC/3.png" width=80% height=80%>


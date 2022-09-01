# Sales analysis

### Introduction and Objectives

This is a sales dataset of a motor vehicle company containing sales, product and consumers' information. This analysis is to  generate various analytics 
and insights from customers' information and segmenting customers due to their purchase behavior, sales revenue generated by each product line and countries. 
This data set contain 25 columns and 2823 rows

### Data source

This data was scraped from the site below;
https://github.com/AllThingsDataWithAngelina/DataSource/blob/main/sales_data_sample.csv

### Data Cleaning and exploration

```
---Inspecting the data
----Whole dataset

SELECT * FROM [exp].[sales_data_sample$];
--Distinct Transaction status

SELECT DISTINCT [STATUS] Transactionstatus FROM [exp].[sales_data_sample$];

--- Real transaction (cancelled orders are excluded)
SELECT * FROM [exp].[sales_data_sample$]
WHERE STATUS != 'Cancelled';

--Distinct Dealsize

SELECT DISTINCT [DEALSIZE]  FROM [exp].[sales_data_sample$];

--Distint Product

SELECT DISTINCT [PRODUCTLINE] FROM [exp].[sales_data_sample$];  --(7 productline)

--Distinct number of customers

SELECT DISTINCT [CUSTOMERNAME] FROM [exp].[sales_data_sample$];  ---(92 customers)

--Distinct countries where customers are buying from

SELECT DISTINCT [COUNTRY] FROM [exp].[sales_data_sample$]        ---(19 countries)

---Each productline have varying prices, indication that there are different brands of cars 

SELECT DISTINCT 
[COUNTRY],
[PRICEEACH],
[PRODUCTLINE] 
FROM [exp].[sales_data_sample$]
where [PRODUCTLINE] ='Motorcycles';
```

### Data analysis 

```
----Total revenue (excluding transactions that are cancelled)

 SELECT FORMAT(ROUND(SUM([SALES]),2), 'C0') AS Total_revenue FROM [exp].[sales_data_sample$]
 WHERE [STATUS] != 'Cancelled';

 ---- SALES BY PRODUCTLINE
 
SELECT  [PRODUCTLINE], ROUND (SUM([SALES]),0) AS Sales
FROM [exp].[sales_data_sample$]
WHERE [STATUS] != 'Cancelled'
GROUP BY [PRODUCTLINE]
ORDER BY Sales DESC;

--SALES BY DEALSIZE (Classic car, Vintage car, Motorcycle)
 
SELECT [DEALSIZE], FORMAT(ROUND(SUM([SALES]),0),'C') AS Sales
FROM [exp].[sales_data_sample$]
WHERE [STATUS] != 'Cancelled'
GROUP BY [DEALSIZE]
ORDER BY Sales DESC;

--SALES BY COUNTRY (USA made the highest sales, Spain,France)

SELECT  [COUNTRY], SUM([SALES]) AS Sales
FROM [exp].[sales_data_sample$]
WHERE [STATUS] != 'Cancelled'
GROUP BY [COUNTRY]
ORDER BY Sales DESC;

---Customer's activeness (calculating the sales, number of orders made by each customers and their last purchase date )
(Recency-Frequency-Monetary)

WITH Pre_RFM AS (SELECT DISTINCT cus.*,
DATEDIFF(DD, cus.Lastpurchase_date,cus.Lastsales_date)  AS DayInterval
FROM
(SELECT [CUSTOMERNAME],
MAX([ORDERDATE]) AS Lastpurchase_date,
COUNT(*) AS No_of_orders,
SUM([SALES]) AS AmountSpent, 
(SELECT  MAX([ORDERDATE]) FROM [exp].[sales_data_sample$]) AS Lastsales_date
FROM [exp].[sales_data_sample$]
WHERE [STATUS] = 'Shipped'
GROUP BY [CUSTOMERNAME]
) AS cus
LEFT JOIN [exp].[sales_data_sample$] customer
ON cus.[CUSTOMERNAME]= customer.[CUSTOMERNAME])
SELECT *
INTO #r_f_m     
FROM Pre_RFM;

SELECT *
FROM  #r_f_m;

--- The result gotten was grouped into four segment according to the Order made, Sales and Interval between customers' purchase
date and company's last transaction date

WITH Rfm AS (SELECT #r_f_m.*,
NTILE(4) OVER(ORDER BY No_of_orders) AS OrderRank,
NTILE(4) OVER(ORDER BY DayInterval ) AS Days_rank,
NTILE(4) OVER (ORDER BY AmountSpent) AS Amountrank
FROM #r_f_m)
SELECT *
INTO #rfm_u
FROM Rfm;

SELECT *
FROM #rfm_u;

WITH rfm_grade AS (SELECT #rfm_u.*,
CAST (OrderRank AS varchar) + CAST(Days_rank AS varchar) + CAST(AmountRank AS varchar) AS unit
FROM #rfm_u)

--- Segmenting customers according to their recency-monetary- frequency value.

SELECT r.CUSTOMERNAME, 
r.No_of_orders,
r.AmountSpent,
r.DayInterval,
CASE
WHEN r.unit IN (141,142,242) THEN 'Lost Customers'
WHEN unit IN (111,121,131,241,242,112)THEN 'Low value customers'
WHEN unit IN(221,222,223,342, 332,232,212,233,322,213,333,343,334,344,333,343,444)THEN 'Medium value customers'
WHEN unit IN(323,313,423,434) THEN 'High value customer'
WHEN unit IN(414,424) THEN 'Loyal/top customers' END Customer_status
FROM rfm_grade r;


---Product that are frequently bought together

WITH OrderNumber AS(SELECT [ORDERNUMBER], 
COUNT(*) Order_count 
FROM [exp].[sales_data_sample$]
WHERE [STATUS] ='Shipped'
GROUP BY [ORDERNUMBER])
SELECT *
INTO #Order
FROM OrderNumber
WHERE Order_count = 2;
SELECT *
FROM #Order;


select distinct OrderNumber, stuff((select ',' + PRODUCTLINE
	from[exp].[sales_data_sample$]  p
	where ORDERNUMBER in 
		(

			select ORDERNUMBER
			from (
				select ORDERNUMBER, count(*) rn
				FROM [exp].[sales_data_sample$]
				where STATUS = 'Shipped'
				group by ORDERNUMBER
			)m
			where rn = 2
		)
        and p.ORDERNUMBER =s.ORDERNUMBER
		for xml path ('')), 1, 1, '') Productbrand
        from [exp].[sales_data_sample$] s
        ORDER BY 2 DESC
   ```     

### Results

**TOTAL REVENUE**

![Screenshot (123)](https://user-images.githubusercontent.com/109418747/187935927-482e6dca-802a-493e-b5ea-dec386c1a86b.png)

**SALES BY PRODUCTLINE**


![Screenshot (124)](https://user-images.githubusercontent.com/109418747/187936529-bd9c71a7-7676-4125-9921-5efafc2591a0.png)


**SALES BY DEALSIZE**


![Screenshot (125)](https://user-images.githubusercontent.com/109418747/187936996-76ca9258-983e-40a0-877d-5be84aad19f3.png)



**SALES BY COUNTRY**


![Screenshot (126)](https://user-images.githubusercontent.com/109418747/187937954-b7cf0af7-5ce7-4b66-8085-a1abcada0bf1.png)

 
 **Two product mostly sold together**
 
 
![image](https://user-images.githubusercontent.com/109418747/187941748-09c3405a-01cc-4f0f-a441-d014b6b07c16.png)


**Customer's Segment**


![image](https://user-images.githubusercontent.com/109418747/187944114-74528e96-36f0-4904-ac87-ab593216ba87.png)


![image](https://user-images.githubusercontent.com/109418747/187996030-7f4efdca-c002-4481-86a6-f7988e31cc30.png)



### Findings and recommendation

Total revenue generated was $9,838,141.

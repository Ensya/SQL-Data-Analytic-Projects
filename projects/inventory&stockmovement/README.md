# [SQL] Inventory & Stock Movement 


## I. Overview

This project is the exploration of a Bicycle Manufacturer's dataset on [Google BigQuery](https://cloud.google.com/bigquery). The dataset is based on the Google Analytics public dataset and contains data from a sample Dataedo documentation. 
* This is a sample Dataedo documentation - AdventureWorks - Microsoft SQL Server sample database.
* The AdventureWorks database supports standard online transaction processing scenarios for a fictitious bicycle
manufacturer (Adventure Works Cycles). Scenarios include Manufacturing, Sales, Purchasing, Product Management,
Contact Management, and Human Resources.
* I will explore the dataset through 3 Departments: Sales, Production, Purchasing.
---

## II. Dataset Access

The AdventureWorks dataset is stored in a public Google BigQuery dataset.
* [Dataset Dictionary](https://drive.google.com/file/d/1bwwsS3cRJYOg1cvNppc1K_8dQLELN16T/view?usp=share_link)

You can download AdventureWorks database here:
· AdventureWorks for SQL Server 2014 (CodePlex)
· AdventureWorks for SQL Server 2012 (CodePlex)
· AdventureWorks for SQL Server 2008R2 (CodePlex)
---

## III. Key Questions
I will explore the dataset through 08 questions: 

* What is the total quantity of items sold, total sales value, and total order quantity for each subcategory in the last 12 months?
* Which 3 subcategories had the highest year-over-year growth rate in quantity sold, calculated as qty_diff = qty_item / prv_qty - 1?
* What are the top 3 TerritoryIDs by order quantity for each year, including ties without skipping rank numbers?
* What is the total discount cost from seasonal promotions for each subcategory?
* What is the retention rate of customers who made their first purchase in 2014 with orders having the status Successfully Shipped?
* What is the trend of stock levels and month-over-month percentage differences for all products in 2011, replacing null growth rates with 0?
* What is the monthly ratio of stock to sales for each product in 2011, including month-over-month and year-over-year changes, ordered by month descending and ratio descending?
* How many orders were in Pending status in 2014, and what is their total sales value?


---

## IV. SQL Solutions


### Query 01: Quantity of items, Sales value & Order quantity by each Subcategory in Last 12 months
* SQL code

```sql
SELECT 
  FORMAT_DATETIME("%b %Y", a.ModifiedDate) as period 
  ,c.Name
  ,sum(a.OrderQty) as item_cnt
  ,sum(a.LineTotal) as sales_value
  ,count(distinct a.SalesOrderID) as order_cnt
FROM `adventureworks2019.Sales.SalesOrderDetail` a
LEFT JOIN `adventureworks2019.Production.Product` b ON a.ProductID = b.ProductID
LEFT JOIN `adventureworks2019.Production.ProductSubcategory` c ON cast(b.ProductSubcategoryID as int) = c.ProductSubcategoryID

GROUP BY 1,2
ORDER BY 2;
``` 

* Query results

<img width="855" height="234" alt="q1" src="https://github.com/user-attachments/assets/c7a11663-c359-4cd1-b895-7848244da79e" />

### Query 02: Retention rate of Customer in 2014 with status of Successfully Shipped (Cohort Analysis)

* SQL code

```sql
WITH Info as (
  SELECT 
    EXTRACT (month FROM ModifiedDate) as month
    ,EXTRACT (YEAR FROM ModifiedDate) as year
    ,CustomerID
    ,COUNT(DISTINCT SalesOrderID) as sale_cnt
  FROM `adventureworks2019.Sales.SalesOrderHeader`
  WHERE Status = 5 AND EXTRACT (year FROM ModifiedDate) = 2014
  GROUP BY 1,2,3
)
,row_num as (
SELECT *
, row_number() over (partition by CustomerID order by month asc) as row_nb
FROM Info
) 
,first_order as(
SELECT 
  distinct month as month_join
  ,year
  ,CustomerID
FROM row_num 
WHERE row_nb = 1 
)
,all_join as (
SELECT 
  DISTINCT a.month, a.year, a.CustomerID
  ,b.month_join
  ,CONCAT( 'M',a.month - b.month_join) as mth_diff
FROM Info as a
LEFT JOIN first_order as b ON a.CustomerID = b.CustomerID
ORDER BY 3
)

SELECT 
  DISTINCT month_join
  ,mth_diff
  , COUNT (DISTINCT CustomerID) as customer_cnt
  FROM all_join
  GROUP BY 1,2 
  ORDER BY 1,2
;
```

* Query results

<img width="547" height="349" alt="q2" src="https://github.com/user-attachments/assets/93c443d5-2b2a-437c-9c2f-fc20de0c9034" />

### Query 3: YoY growth rate by SubCategory & release top 3 category with highest grow rate 
* SQL code

```sql
WITH yearly as (
SELECT
  EXtract(year from a.ModifiedDate) as period 
  ,c.Name as Name
  ,sum(a.OrderQty) as item_cnt
FROM `adventureworks2019.Sales.SalesOrderDetail` a
LEFT JOIN `adventureworks2019.Production.Product` b ON a.ProductID = b.ProductID
LEFT JOIN `adventureworks2019.Production.ProductSubcategory` c ON cast(b.ProductSubcategoryID as int) = c.ProductSubcategoryID
GROUP BY 1,2
ORDER BY 2
)
SELECT 
  y1.Name 
  ,y1.item_cnt as qty_curr_year
  ,y0.item_cnt as qty_prv_year
  ,Round(((y1.item_cnt - y0.item_cnt)/NULLIF(y0.item_cnt,0)),2) as yoy_growth
FROM yearly as y1
JOIN yearly as y0
  ON y1.Name = y0.Name 
  AND y1.period = y0.period + 1
ORDER BY yoy_growth DESC
LIMIT 3;
```
* Query results

<img width="633" height="110" alt="q3" src="https://github.com/user-attachments/assets/3bde7b0b-eeea-4b6f-989e-28a1e8655446" />

### Query 04: Ranking Top 3 TeritoryID with biggest Order quantity of every year. If there's TerritoryID with same quantity in a year, do not skip the rank number

* SQL code

```sql
WITH
  YearlyTerritorySales AS (
    SELECT
      EXTRACT(YEAR FROM t0.ModifiedDate) AS sales_year,
      t0.TerritoryID,
      SUM(t1.OrderQty) AS total_order_quantity
    FROM `adventureworks2019.Sales.SalesOrderHeader` AS t0
    INNER JOIN `adventureworks2019.Sales.SalesOrderDetail` AS t1
      ON t0.SalesOrderID = t1.SalesOrderID
    GROUP BY sales_year, t0.TerritoryID
  ),
  RankedTerritorySales AS (
    SELECT
      sales_year,
      TerritoryID,
      total_order_quantity,
      DENSE_RANK()
        OVER (PARTITION BY sales_year ORDER BY total_order_quantity DESC)
        AS rank_num
    FROM `YearlyTerritorySales`
  )
SELECT sales_year, TerritoryID, total_order_quantity, rank_num
FROM `RankedTerritorySales`
WHERE rank_num <= 3
ORDER BY sales_year, rank_num, TerritoryID;
```

* Query results

<img width="558" height="352" alt="q4" src="https://github.com/user-attachments/assets/5c433490-7a4b-4fa4-9c21-2e533f92aacd" />

### Query 05: Calculate Total Discount Cost belongs to Seasonal Discount for each SubCategory
* SQL code

```sql
SELECT
  FORMAT_DATETIME("%Y", a.ModifiedDate) as period
  ,c.Name AS Name
  ,SUM(a.OrderQty * a.UnitPrice * a.UnitPriceDiscount) AS total_discount_cost
FROM `adventureworks2019.Sales.SalesOrderDetail` AS a
INNER JOIN `adventureworks2019.Production.Product` AS b
  ON a.ProductID = b.ProductID
INNER JOIN `adventureworks2019.Production.ProductSubcategory` AS c
  ON CAST(b.ProductSubcategoryID AS BIGNUMERIC) = c.ProductSubcategoryID
INNER JOIN `adventureworks2019.Sales.SpecialOffer` AS d
  ON a.SpecialOfferID = d.SpecialOfferID 
WHERE LOWER(d.Type) LIKE '%seasonal discount%'
GROUP BY 1,2 
ORDER BY 2;

```

* Query results

<img width="589" height="85" alt="q5" src="https://github.com/user-attachments/assets/99cfc248-510c-4c02-a5cf-50aa00c07acd" />

### Query 06: Trend of Stock level & MoM diff % by all product in 2011. If %gr rate is null then 0. Round to 1 decimal
* SQL code

```sql

WITH Stocked_current AS ( 
    SELECT
        p.Name AS Product_Name,
        p.ProductID AS ProductID,
        EXTRACT(MONTH FROM w.EndDate) AS Month,
        EXTRACT(YEAR FROM w.EndDate) AS Year,
        SUM(w.StockedQty) AS Stock_current
    FROM `adventureworks2019.Production.Product` p
    LEFT JOIN `adventureworks2019.Production.WorkOrder` w
        ON p.ProductID = w.ProductID
    WHERE EXTRACT(YEAR FROM w.EndDate) = 2011
    GROUP BY 1,2,3,4
),
Stock_previous AS (
    SELECT
        Product_Name,
        ProductID,
        Month - 1 AS Month,
        Year AS Year,
        Stock_current AS Stock_prv
        FROM Stocked_current
    GROUP BY 1,2,3,4,5
)
SELECT
    c.Product_Name AS Product_Name,
    c.Month,
    c.Year,
    c.Stock_current,
    p.Stock_prv,
     round(((p.Stock_prv - c.Stock_current) / c.Stock_current )*100,1) AS Diff_Percent
FROM Stocked_current c
LEFT JOIN Stock_previous p
    ON c.ProductID = p.ProductID
   AND c.Month = p.Month
   AND c.Year = p.Year
ORDER BY
    Product_Name,
    Year,
    Month desc;
```
* Query results

<img width="925" height="288" alt="q6" src="https://github.com/user-attachments/assets/9ceacb9a-6d0e-474f-be41-0f7b6d269817" />

### Query 07: Number of order and value at Pending status in 2014. filter condition status =1 

* SQL code

```sql
SELECT  
 extract (year from OrderDate) year
 ,Status
 ,COUNT(DISTINCT PurchaseOrderID) AS order_cnt
 ,SUM(TotalDue) AS total_value
FROM `adventureworks2019.Purchasing.PurchaseOrderHeader`
WHERE
  Status
    = 1  -- Assuming 'Pending' status is represented by integer 1. Please verify the exact mapping if needed.
  AND EXTRACT(YEAR FROM OrderDate) = 2014
  group by 1,2
  order by 1,2;
```
* Query results

<img width="558" height="54" alt="q7" src="https://github.com/user-attachments/assets/3452d4b1-939c-43db-baab-31b2c60051ec" />

### Query 08: Ratio of Stock / Sales in 2011 by product name, by month. Order results by month desc, ratio desc. Round Ratio to 1 decimal

* SQL code

```sql
WITH sale_info AS (
    SELECT
        p.ProductID,
        p.Name,
        EXTRACT(MONTH FROM sd.ModifiedDate) AS Month,
        EXTRACT(YEAR FROM sd.ModifiedDate) AS Year,
        SUM(sd.OrderQty) AS Order_Qty
    FROM `adventureworks2019.Sales.SalesOrderDetail` sd
    LEFT JOIN `adventureworks2019.Production.Product` p
        ON sd.ProductID = p.ProductID
    WHERE EXTRACT(YEAR FROM sd.ModifiedDate) = 2011
    GROUP BY 1,2,3,4
),

stock_info AS (
    SELECT
        p.ProductID,
        p.Name,
        EXTRACT(MONTH FROM w.EndDate) AS Month,
        EXTRACT(YEAR FROM w.EndDate) AS Year,
        SUM(w.StockedQty) AS Stock_Qty
    FROM `adventureworks2019.Production.WorkOrder` w
    LEFT JOIN `adventureworks2019.Production.Product` p
        ON w.ProductID = p.ProductID
    WHERE EXTRACT(YEAR FROM w.EndDate) = 2011
    GROUP BY 1,2,3,4
)

    SELECT
        st.Month,
        st.Year,
        st.ProductID,
        st.Name,
        COALESCE(st.Stock_Qty, 0) AS Stock_Qty,
        COALESCE(sa.Order_Qty, 0) AS Order_Qty,
        ROUND(
            SAFE_DIVIDE(
                COALESCE(st.Stock_Qty, 0),
                COALESCE(sa.Order_Qty, 0)
            ), 1
        ) AS Ratio
    FROM stock_info st
    LEFT JOIN sale_info sa
        ON st.ProductID = sa.ProductID
        AND st.Month = sa.Month
        AND st.Year = sa.Year
    ORDER BY st.Year DESC, st.Month DESC, Ratio DESC;
```
* Query results

<img width="1007" height="292" alt="q8" src="https://github.com/user-attachments/assets/cd40127a-2131-4f2a-9cb8-fc6ab2fdc7b8" />

---

## V. Conclusion

 This analysis highlights key patterns in sales, inventory, and customer behavior. It identified high-growth subcategories, top-performing territories, and the impact of seasonal discounts, while also revealing trends in stock levels and customer retention. The insights from these queries provide a clear basis for data-driven decisions in sales optimization, inventory management, and customer engagement.

 

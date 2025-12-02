# [SQL] Ecommerce Analysis 


## 📌 Overview

This project contains an eCommerce dataset on [Google BigQuery](https://cloud.google.com/bigquery). The dataset is based on the Google Analytics public dataset and contains data from an eCommerce website. In this project, I will using SQL to explore the funnel analysis.  

---

## 📊 Dataset Access

The eCommerce dataset is stored in a public Google BigQuery dataset. To access the dataset, follow these steps:
* Log in to your Google Cloud Platform account and create a new project.
* Navigate to the BigQuery console and select your newly created project.
* In the navigation panel, select "Add Data" and then "Search a project".
* Enter the project ID **"bigquery-public-data.google_analytics_sample.ga_sessions"** and click "Enter".
* Click on the **"ga_sessions_"** table to open it.

---

## 🧠 Key Questions
I will explore the dataset through 08 query: 

| Query  | Main Metric / Focus                               | Time Period      | Purpose                                           |
| ------ | ------------------------------------------------- | ---------------- | ------------------------------------------------- |
| **01** | Total visits, pageviews, transactions             | **Jan–Mar 2017** | Evaluate quarterly website performance            |
| **02** | Bounce rate by traffic source                     | **July 2017**    | Identify top-performing and weak traffic channels |
| **03** | Revenue by traffic source (weekly + monthly)      | **June 2017**    | Track revenue trends and channel contribution     |
| **04** | Avg pageviews (purchasers vs. non-purchasers)     | **Jun–Jul 2017** | Compare user engagement behavior across groups    |
| **05** | Avg transactions per purchasing user              | **July 2017**    | Measure purchasing intensity                      |
| **06** | Avg money spent per purchasing session            | **July 2017**    | Evaluate session monetization efficiency          |
| **07** | Products purchased together with a specific item  | **July 2017**    | Discover product affinity and cross-sell insights |
| **08** | Funnel conversion (View → Add to Cart → Purchase) | **Jan–Mar 2017** | Understand product-level conversion performance   |


---

## 🧩 SQL Solutions


### Query 01: Calculate total visit, pageview, transaction and revenue for January, February and March 2017 order by month
* SQL code

```sql
SELECT
  FORMAT_DATE('%Y-%m', PARSE_DATE('%Y%m%d', date)) AS Month
  ,SUM(totals.visits) AS visits
  ,SUM(totals.pageviews) AS pageviews
  ,SUM(totals.transactions) AS transactions
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE _table_suffix BETWEEN '0101' AND '0330'  -- first 3 months of the year
GROUP BY Month
ORDER BY Month;

``` 

* Query results
<img width="635" height="108" alt="query 1" src="https://github.com/user-attachments/assets/eee03ac4-77ce-44fa-90c5-b741851d3b76" />

### Query 02: Bounce rate per traffic source in July 2017
* SQL code

```sql
SELECT 
    trafficSource.`source` as `source`
    ,SUM(totals.visits) as total_visits
    ,SUM(totals.bounces) as total_no_of_bounces
    ,SUM(totals.bounces)/sum(totals.visits)*100 as bounce_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE _table_suffix between '0701' and '0731' -- extract data for July 2017
GROUP BY `source`
ORDER BY total_visits desc;
```

* Query results

<img width="828" height="206" alt="query2" src="https://github.com/user-attachments/assets/cbc81e2f-a35a-4ccb-b03d-0a9db6ed2062" />

### Query 3: Revenue by traffic source by week, by month in June 2017
* SQL code

```sql
  SELECT 
    'Month' as timetype  
    ,FORMAT_DATE('%Y%m',PARSE_DATE ("%Y%m%d",date)) as `time`
    ,trafficSource.`source` as `source` 
    ,sum(productRevenue)/1000000 as revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
    ,UNNEST (hits)hits
    ,UNNEST (hits.product)product
  WHERE _table_suffix between '0601' and '0630' -- extract data for June 2017
    AND product.productRevenue is not null
  GROUP BY `time`,`source`

UNION ALL

  SELECT 
    'Week' as timetype  
    ,FORMAT_DATE('%Y%W',PARSE_DATE ("%Y%m%d",date)) as `time`
    ,trafficSource.`source` as `source` 
    ,sum(productRevenue)/1000000 as revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
    ,UNNEST (hits)hits
    ,UNNEST (hits.product)product
  WHERE _table_suffix BETWEEN '0601' and '0630' -- extract data for June 2017
                    AND product.productRevenue is not null
  GROUP BY `time`,`source`

  ORDER BY `source`,`time`, timetype;
```
* Query results

<img width="869" height="296" alt="query3 " src="https://github.com/user-attachments/assets/545b8063-b05a-4877-b6a3-be4f0a7002c3" />

### Query 04: Average number of product pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017
* SQL code

```sql
SELECT
  FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month
  -- purchasers: totals.transactions equal and greator than 1, porductRev is not null
  ,SUM(IF(product.productRevenue is not null and totals.transactions >= 1, totals.pageviews, 0))
    / COUNT(DISTINCT IF(product.productRevenue > 0 and totals.transactions >=1, fullVisitorId, NULL))
      AS avg_pageviews_purchasers

  -- non-purchaers: totals.transactions IS NULL;  product.productRevenue is null 
  ,SUM(IF( product.productRevenue is null and totals.transactions is null, totals.pageviews, 0))
    / COUNT(DISTINCT IF(product.productRevenue IS NULL and totals.transactions is null, fullVisitorId, NULL))
      AS avg_pageviews_non_purchasers

FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
     ,UNNEST(hits) hits
     ,UNNEST(hits.product) product
WHERE _table_suffix BETWEEN '0601' AND '0731'
GROUP BY month
ORDER BY month;
```

* Query results

<img width="511" height="81" alt="query 4" src="https://github.com/user-attachments/assets/fc4667d2-5198-43f6-8dd6-e730c16bfba0" />

### Query 05: Average number of transactions per user that made a purchase in July 2017
* SQL code

```sql
SELECT
    SUM(IF(totals.transactions >=1 and product.productRevenue is not null, totals.transactions,null))
    / COUNT(DISTINCT fullVisitorId) AS avg_transactions_per_user
FROM `bigquery-public-data`.`google_analytics_sample`.`ga_sessions_2017*`
    ,UNNEST(hits) hits
    ,UNNEST(hits.product) product
WHERE
    _table_suffix BETWEEN '0701'AND '0731'
                 AND product.productRevenue IS NOT NULL
                 AND totals.transactions > 0
  ;
```

* Query results

<img width="187" height="54" alt="query 5" src="https://github.com/user-attachments/assets/1062adbc-0ff7-48ce-a8ab-73acb11217ce" />

### Query 06: Average amount of money spent per session. Only include purchaser data in July 2017
* SQL code

```sql
  SELECT 
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month
    ,(SUM(product.productRevenue)/sum(totals.visits)) --visits is diff from visitors
    /1000000 as avg_revenue_by_user_per_visit -- shorten the number
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
  ,UNNEST (hits)hits
  ,UNNEST (hits.product)product
  WHERE _table_suffix between '0701' and '0731' -- extract data only for July 2017
    AND product.productRevenue is not null
  GROUP BY month;
```
* Query results

<img width="388" height="57" alt="query 6" src="https://github.com/user-attachments/assets/5be01ec1-ac41-4d85-a334-27eb8d725f7b" />

### Query 07: Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017. Output should show product name and the quantity was ordered.
* SQL code

```sql
  WITH henley_buyers AS ( -- list of cutomers that purchasing "YouTube Men's Vintage Henley"
                    SELECT 
                      distinct fullVisitorId
                    FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
                    ,UNNEST (hits)hits
                    ,UNNEST (hits.product)product
                    WHERE _table_suffix BETWEEN '0701' AND '0731' -- extract data only for July 2017
                                        AND product.productRevenue is not null
                                        AND totals.transactions >=1
                                        AND product.v2ProductName = "YouTube Men's Vintage Henley"
                  )
  -- list other products those customers bought
  SELECT
    p.v2ProductName AS other_purchased_products,
    SUM(p.productQuantity) AS quantity
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
      ,UNNEST(hits) AS h
      ,UNNEST(h.product) AS p
  WHERE fullVisitorId IN (SELECT fullVisitorId FROM henley_buyers) -- user from henley_buyers that extract only for July 2017
    AND p.productRevenue is not null
    AND totals.transactions >= 1
    AND p.v2ProductName != "YouTube Men's Vintage Henley"   -- exclude the target product
  GROUP BY other_purchased_products
  ORDER BY quantity DESC;
```
* Query results

<img width="817" height="277" alt="query 7" src="https://github.com/user-attachments/assets/04095c60-ccf3-468b-9531-c024349e63c6" />

### Query 08: Calculate cohort map from pageview to addtocart to purchase in last 3 month.
* SQL code

```sql
WITH product_actions as (
              SELECT 
                FORMAT_DATE('%Y%m',PARSE_DATE ("%Y%m%d",date)) as month
              ,countif (hits.eCommerceAction.action_type ='2') as num_product_view  
              ,countif(hits.eCommerceAction.action_type ='3') as num_addtocart
              ,countif(hits.eCommerceAction.action_type ='6' 
                AND product.productRevenue is not null) as num_purchase
              FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
              ,UNNEST (hits)hits
              ,UNNEST (hits.product)product
              WHERE _table_suffix between '0101' and '0331' 
            GROUP BY month   
            )

SELECT
    month,
    SUM(num_product_view) AS num_product_view,
    SUM(num_addtocart) AS num_addtocart,
    SUM(num_purchase) AS num_purchase,
    SAFE_DIVIDE(SUM(num_addtocart) * 100, SUM(num_product_view)) AS add_to_cart_rate,
    SAFE_DIVIDE(SUM(num_purchase) * 100, SUM(num_product_view)) AS purchase_rate
FROM
    product_actions
GROUP BY month
ORDER BY month;
```
* Query results

<img width="876" height="111" alt="query 8" src="https://github.com/user-attachments/assets/f4e98b36-888e-4910-a0f8-319d18bfbeb1" />

---

## Conclusion

  The data shows a holistic view of business performance, user behavior, and revenue efficiency:
* Some traffic sources perform better in engagement and conversion.
* Purchasers behave differently from non-purchasers, suggesting targeted strategies could improve conversions.
* Product affinity and funnel analysis point to opportunities for increasing average order value and optimizing the purchase path.
  
### In short:
This project has demonstrated the power of using SQL to gain insights into large datasets. The insights highlight where the business is performing well and where improvements can be made in marketing, product strategy, and user experience.
However, to deep dive into the insights and key trends, the next step is to visualize the data through tools like Tableau,Power BI... 

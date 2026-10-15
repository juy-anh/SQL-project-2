> Updated: Oct 2026

![Bicycle manufacturer business analytics dashboard](bicycle-manufacturer-analytics-hero.png)

# Bicycle Manufacturer Business Insights | BigQuery SQL

This portfolio project analyses business performance for a bicycle manufacturer using **Google BigQuery SQL** and the **AdventureWorks2019** dataset. It brings together sales, production, inventory, customer-retention, and purchasing data to answer practical questions that business teams can use for decision-making.

---

## Table of Contents

1. [Background & Overview](#-background--overview)
2. [Dataset Description & Data Structure](#-dataset-description--data-structure)
3. [Approach](#-approach)
4. [SQL Analysis](#️-sql-analysis)
5. [Key Takeaways & Recommendations](#-key-takeaways--recommendations)
6. [How to Run](#-how-to-run)

---

## Background & Overview

### Objective

A bicycle manufacturer needs a connected view of its commercial and operational performance. This project uses eight SQL queries to analyse product demand, sales growth, territory performance, promotion cost, customer retention, inventory movement, and procurement workload.

- Monitor sales volume, sales value, and order count by product subcategory.
- Identify fast-growing product categories and leading sales territories.
- Evaluate the cost of seasonal-discount promotions.
- Understand repeat purchasing through a customer cohort view.
- Monitor stock trends and stock-to-sales efficiency.
- Track pending purchase orders and their financial value.

### Who is this project for?

- Data analysts and business analysts
- Sales, product, production, and supply-chain managers
- Commercial and operations decision-makers

---

## 📂 Dataset Description & Data Structure

### Data source

- **Dataset:** `adventureworks2019` in Google BigQuery
- **Domain:** Bicycle manufacturing and distribution
- **Data type:** Relational business dataset covering orders, customers, products, promotions, work orders, and purchase orders
- **Business areas analysed:** Sales (Queries 1–5), Production (Queries 6–7), and Purchasing (Query 8)

### Tables used

| Schema | Table | Purpose in this project |
|---|---|---|
| Sales | `SalesOrderDetail` | Order-line quantity, revenue, unit price, promotions, and sales-order identifier. |
| Sales | `SalesOrderHeader` | Customer, order status, and sales territory information. |
| Sales | `SpecialOffer` | Discount percentage and offer type used for promotion-cost analysis. |
| Production | `Product` | Product name and product-subcategory relationship. |
| Production | `ProductSubcategory` | Product-category labels used in sales analysis. |
| Production | `WorkOrder` | Product production/work-order quantity used as the stock measure in this project. |
| Purchasing | `PurchaseOrderHeader` | Purchase-order status and total value. |

### Key fields used

| Field | Description / use |
|---|---|
| `ModifiedDate` | Filters analysis periods and derives month/year. |
| `ProductID`, `ProductSubcategoryID` | Join keys across sales and production tables. |
| `OrderQty`, `LineTotal`, `UnitPrice` | Product quantity, sales value, and discount-cost calculations. |
| `SalesOrderID`, `PurchaseOrderID` | Distinct sales/purchase order counting. |
| `CustomerID`, `Status` | Cohort analysis for successfully shipped orders and pending purchasing orders. |
| `TerritoryID` | Ranking sales territories by order quantity. |
| `DiscountPct`, `Type` | Identifying and quantifying seasonal discounts. |
| `StockedQty` | Production/work-order quantity used to track inventory movement. |
| `TotalDue` | Total monetary value of purchase orders. |

### Data relationships

`SalesOrderDetail` links to `Product` through `ProductID`, and `Product` links to `ProductSubcategory` through `ProductSubcategoryID`. `SalesOrderDetail` links to `SalesOrderHeader` through `SalesOrderID`; it also links to `SpecialOffer` through `SpecialOfferID`. `WorkOrder` links to `Product` through `ProductID`.

---

## Approach

1. **Map business questions to data domains** — sales, production, and purchasing.
2. **Join transaction and master data** — use product, subcategory, customer, territory, offer, and purchase-order dimensions.
3. **Build time-based metrics** — calculate last-12-month, year-over-year, month-over-month, and cohort metrics.
4. **Apply analytical SQL** — use CTEs, aggregations, window functions, rankings, and safe null handling.
5. **Interpret the results** — connect sales growth, inventory, promotion cost, and purchasing activity to business actions.

---

## SQL Analysis

### Query 01 — Sales performance by subcategory in the last 12 months

**Business question:** What are the quantity of items sold, total sales value, and order count by subcategory in the latest 12-month period?

```sql
SELECT
  FORMAT_DATETIME('%b %Y', a.ModifiedDate) AS month,
  c.Name AS subcategory_name,
  SUM(a.OrderQty) AS qty_item,
  SUM(a.LineTotal) AS total_sales,
  COUNT(DISTINCT a.SalesOrderID) AS order_cnt
FROM `adventureworks2019.Sales.SalesOrderDetail` a
LEFT JOIN `adventureworks2019.Production.Product` b
  ON a.ProductID = b.ProductID
LEFT JOIN `adventureworks2019.Production.ProductSubcategory` c
  ON b.ProductSubcategoryID = CAST(c.ProductSubcategoryID AS STRING)
WHERE DATE(a.ModifiedDate) >= (
  SELECT DATE_SUB(DATE(MAX(ModifiedDate)), INTERVAL 12 MONTH)
  FROM `adventureworks2019.Sales.SalesOrderDetail`
)
GROUP BY 1, 2
ORDER BY 2, 1;
```

**Result evidence (9 rows shown in the workbook):**

![Query 01 result: sales by subcategory](assets/query_results/query_01_result.png)

**Observation & finding:** This query provides a monthly commercial view by subcategory. It can reveal whether revenue is driven by high item volume, higher-priced products, or a larger number of customer orders.

---

### Query 02 — Top 3 subcategories by YoY quantity growth

**Business question:** Which three subcategories had the highest year-over-year growth in item quantity?

```sql
WITH sale_info AS (
  SELECT
    FORMAT_TIMESTAMP('%Y', a.ModifiedDate) AS yr,
    c.Name,
    SUM(a.OrderQty) AS qty_item
  FROM `adventureworks2019.Sales.SalesOrderDetail` a
  LEFT JOIN `adventureworks2019.Production.Product` b
    ON a.ProductID = b.ProductID
  LEFT JOIN `adventureworks2019.Production.ProductSubcategory` c
    ON CAST(b.ProductSubcategoryID AS INT64) = c.ProductSubcategoryID
  GROUP BY 1, 2
),
sale_diff AS (
  SELECT
    yr,
    Name,
    qty_item,
    LEAD(qty_item) OVER (PARTITION BY Name ORDER BY yr DESC) AS prv_qty,
    ROUND(
      qty_item / LEAD(qty_item) OVER (PARTITION BY Name ORDER BY yr DESC) - 1,
      2
    ) AS qty_diff
  FROM sale_info
),
ranked AS (
  SELECT
    *,
    DENSE_RANK() OVER (ORDER BY qty_diff DESC) AS growth_rank
  FROM sale_diff
)
SELECT Name, qty_item, prv_qty, qty_diff
FROM ranked
WHERE growth_rank <= 3
ORDER BY growth_rank;
```

**Result evidence (all 3 ranked subcategories):**

![Query 02 result: top YoY growth subcategories](assets/query_results/query_02_result.png)

**Observation & finding:** Mountain Frames increased item quantity by 521% versus the prior year. This is a strong demand-growth signal; production capacity and material availability should be reviewed before treating the increase as sustainable.

---

### Query 03 — Top 3 sales territories each year

**Business question:** Which TerritoryIDs ranked in the top three by order quantity in each year, allowing ties without skipped ranks?

```sql
WITH sale_info AS (
  SELECT
    FORMAT_TIMESTAMP('%Y', a.ModifiedDate) AS yr,
    b.TerritoryID,
    SUM(a.OrderQty) AS order_cnt
  FROM `adventureworks2019.Sales.SalesOrderDetail` a
  LEFT JOIN `adventureworks2019.Sales.SalesOrderHeader` b
    ON a.SalesOrderID = b.SalesOrderID
  GROUP BY 1, 2
),
sale_rank AS (
  SELECT
    yr,
    TerritoryID,
    order_cnt,
    DENSE_RANK() OVER (PARTITION BY yr ORDER BY order_cnt DESC) AS rk
  FROM sale_info
)
SELECT yr, TerritoryID, order_cnt, rk
FROM sale_rank
WHERE rk <= 3
ORDER BY yr, rk;
```

**Result evidence (12 rows shown in the workbook):**

![Query 03 result: territory ranking](assets/query_results/query_03_result.png)

**Observation & finding:** Territory 4 was the leading territory in the example 2014 output. Its sales approach, customer mix, and product assortment can be benchmarked against lower-performing territories.

---

### Query 04 — Seasonal discount cost by subcategory

**Business question:** What was the total discount cost for each subcategory under seasonal-discount promotions?

```sql
SELECT
  FORMAT_TIMESTAMP('%Y', ModifiedDate) AS year,
  Name AS subcategory_name,
  SUM(disc_cost) AS total_discount_cost
FROM (
  SELECT DISTINCT
    a.ModifiedDate,
    c.Name,
    d.DiscountPct,
    d.Type,
    a.OrderQty * d.DiscountPct * a.UnitPrice AS disc_cost
  FROM `adventureworks2019.Sales.SalesOrderDetail` a
  LEFT JOIN `adventureworks2019.Production.Product` b
    ON a.ProductID = b.ProductID
  LEFT JOIN `adventureworks2019.Production.ProductSubcategory` c
    ON CAST(b.ProductSubcategoryID AS INT64) = c.ProductSubcategoryID
  LEFT JOIN `adventureworks2019.Sales.SpecialOffer` d
    ON a.SpecialOfferID = d.SpecialOfferID
  WHERE LOWER(d.Type) LIKE '%seasonal discount%'
)
GROUP BY 1, 2;
```

**Result evidence (all returned rows shown in the workbook):**

![Query 04 result: seasonal discount cost](assets/query_results/query_04_result.png)

**Observation & finding:** The output quantifies the financial cost of a promotion by product group. It should be compared with incremental sales or margin uplift before deciding whether a seasonal discount was effective.

---

### Query 05 — Customer retention cohort in 2014

**Business question:** How many customers who first successfully shipped an order in a month returned in later months during 2014?

```sql
WITH info AS (
  SELECT
    EXTRACT(MONTH FROM ModifiedDate) AS month_no,
    EXTRACT(YEAR FROM ModifiedDate) AS year_no,
    CustomerID,
    COUNT(DISTINCT SalesOrderID) AS order_cnt
  FROM `adventureworks2019.Sales.SalesOrderHeader`
  WHERE FORMAT_TIMESTAMP('%Y', ModifiedDate) = '2014'
    AND Status = 5
  GROUP BY 1, 2, 3
),
first_order AS (
  SELECT month_no, year_no, CustomerID
  FROM (
    SELECT
      month_no,
      year_no,
      CustomerID,
      ROW_NUMBER() OVER (PARTITION BY CustomerID ORDER BY month_no) AS row_numb
    FROM info
  )
  WHERE row_numb = 1
),
month_gap AS (
  SELECT
    a.CustomerID,
    b.month_no AS month_join,
    CONCAT('M - ', a.month_no - b.month_no) AS month_diff
  FROM info a
  LEFT JOIN first_order b USING (CustomerID)
)
SELECT
  month_join,
  month_diff,
  COUNT(DISTINCT CustomerID) AS customer_cnt
FROM month_gap
GROUP BY 1, 2
ORDER BY 1, 2;
```

**Result evidence (20 cohort rows shown in the workbook):**

![Query 05 result: customer cohort retention](assets/query_results/query_05_result.png)

**Observation & finding:** January's cohort contained 2,076 customers at M-0. The query currently returns retained-customer counts, not a percentage retention rate. For an actual retention-rate KPI, divide each later-month customer count by the M-0 customer count for the same cohort.

---

### Query 06 — Stock trend and MoM change by product

**Business question:** What was the monthly stock level for each product in 2011, and how did it change month over month?

```sql
WITH raw_data AS (
  SELECT
    EXTRACT(MONTH FROM a.ModifiedDate) AS mth,
    EXTRACT(YEAR FROM a.ModifiedDate) AS yr,
    b.Name,
    SUM(a.StockedQty) AS stock_qty
  FROM `adventureworks2019.Production.WorkOrder` a
  LEFT JOIN `adventureworks2019.Production.Product` b
    ON a.ProductID = b.ProductID
  WHERE FORMAT_TIMESTAMP('%Y', a.ModifiedDate) = '2011'
  GROUP BY 1, 2, 3
)
SELECT
  Name,
  mth,
  yr,
  stock_qty,
  stock_prv,
  ROUND(COALESCE((stock_qty / stock_prv - 1) * 100, 0), 1) AS mom_diff_pct
FROM (
  SELECT
    mth,
    yr,
    Name,
    stock_qty,
    LEAD(stock_qty) OVER (PARTITION BY Name ORDER BY mth DESC) AS stock_prv
  FROM raw_data
)
ORDER BY Name, mth DESC;
```

**Result evidence (17 rows shown in the workbook):**

![Query 06 result: stock trend and MoM change](assets/query_results/query_06_result.png)

**Observation & finding:** BB Bali Bearing's stock quantity was 41.7% lower in December than in the prior month. A sharp decline may be healthy if demand increased, but it could also indicate a replenishment risk; sales and planned production should be checked together.

---

### Query 07 — Stock-to-sales ratio by product and month

**Business question:** What was the ratio of stock to sales for each product in 2011, ordered by month and ratio?

```sql
WITH sale_info AS (
  SELECT
    EXTRACT(MONTH FROM a.ModifiedDate) AS mth,
    EXTRACT(YEAR FROM a.ModifiedDate) AS yr,
    a.ProductID,
    b.Name,
    SUM(a.OrderQty) AS sales
  FROM `adventureworks2019.Sales.SalesOrderDetail` a
  LEFT JOIN `adventureworks2019.Production.Product` b
    ON a.ProductID = b.ProductID
  WHERE FORMAT_TIMESTAMP('%Y', a.ModifiedDate) = '2011'
  GROUP BY 1, 2, 3, 4
),
stock_info AS (
  SELECT
    EXTRACT(MONTH FROM ModifiedDate) AS mth,
    EXTRACT(YEAR FROM ModifiedDate) AS yr,
    ProductID,
    SUM(StockedQty) AS stock_cnt
  FROM `adventureworks2019.Production.WorkOrder`
  WHERE FORMAT_TIMESTAMP('%Y', ModifiedDate) = '2011'
  GROUP BY 1, 2, 3
)
SELECT
  a.mth,
  a.yr,
  a.ProductID,
  a.Name,
  a.sales AS order_qty,
  b.stock_cnt AS stock_qty,
  ROUND(COALESCE(b.stock_cnt, 0) / a.sales, 1) AS stock_sales_ratio
FROM sale_info a
FULL JOIN stock_info b
  ON a.ProductID = b.ProductID
  AND a.mth = b.mth
  AND a.yr = b.yr
ORDER BY 1 DESC, 7 DESC;
```

**Result evidence (multiple December rows shown in the workbook):**

![Query 07 result: stock-to-sales ratio](assets/query_results/query_07_result.png)

**Observation & finding:** The example product had 27 stock units for one unit sold in December, producing a ratio of 27.0. A high stock-to-sales ratio may indicate overstock or weak demand; persistently low ratios can signal stockout risk.

---

### Query 08 — Pending purchase orders in 2014

**Business question:** How many purchase orders were pending in 2014, and what was their total value?

```sql
SELECT
  EXTRACT(YEAR FROM ModifiedDate) AS yr,
  Status,
  COUNT(DISTINCT PurchaseOrderID) AS order_cnt,
  SUM(TotalDue) AS value
FROM `adventureworks2019.Purchasing.PurchaseOrderHeader`
WHERE Status = 1
  AND EXTRACT(YEAR FROM ModifiedDate) = 2014
GROUP BY 1, 2;
```

**Result evidence:**

![Query 08 result: pending purchase orders](assets/query_results/query_08_result.png)

**Observation & finding:** There were 224 pending purchase orders valued at approximately 3.87 million in 2014. This is a material procurement pipeline that should be monitored for delayed receipts, supplier constraints, and cash-flow exposure.

---

## 🔎 Key Takeaways & Recommendations

- **Protect supply for fast-growth subcategories.** Mountain Frames' 521% growth warrants demand validation, capacity planning, and supplier readiness.
- **Use territory benchmarks.** Compare the product mix and sales practices of leading Territory 4 with other territories.
- **Measure promotions by profitability, not cost alone.** Pair seasonal-discount cost with incremental revenue and gross margin.
- **Turn cohort counts into a retention-rate KPI.** Divide later-month active customers by each cohort's M-0 population and monitor changes over time.
- **Balance stock and demand.** Investigate sharp monthly stock changes and extreme stock-to-sales ratios before they become stockouts or obsolete inventory.
- **Prioritise high-value pending purchase orders.** Assign clear owners and aging thresholds to reduce procurement bottlenecks.

---

## How to Run

1. Open **BigQuery Console** and select **GoogleSQL / Standard SQL**.
2. Ensure the `adventureworks2019` dataset is available in your BigQuery environment.
3. Copy any query from this README and run it.
4. Review query cost and table references before execution.

---

### Skills Demonstrated

`BigQuery SQL` · `JOINs` · `CTEs` · `Aggregation` · `Window Functions` · `DENSE_RANK` · `ROW_NUMBER` · `LEAD` · `YoY / MoM Analysis` · `Cohort Analysis` · `Sales & Inventory Analytics`

### References

- [AdventureWorks dataset dictionary](https://drive.google.com/file/d/1bwwsS3cRJYOg1cvNppc1K_8dQLELN16T/view)

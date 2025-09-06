# 📊 E-commerce Customer Behavior & Product Performance Analysis  
*(Data Analysis Apprenticeship Project – The Look E-Commerce Dataset)*  

---

## 🎯 Apprenticeship Alignment: Google Data Analytics  
This project is designed to align with the **Google Data Analytics Apprenticeship Job Description (JD)** by showcasing the required skills in:  

- SQL & BigQuery  
- Data cleaning & anomaly detection  
- Data visualization (Google Sheets)  
- Storytelling with data (Google Slides)  
- Business insights & recommendations  

**Goal:** Demonstrate **end-to-end data analysis** with real business value.  

---

## 📌 Project Overview  
Using the public **“The Look” E-commerce dataset** on BigQuery, this project analyzes customer behavior, product performance, and conversion funnels to deliver **actionable business insights**.  

### 🔄 Data Analysis Lifecycle  
1. **Data Collection & Assessment** – Extracted data from BigQuery, checked anomalies (NULLs, duplicates, invalid dates, negatives).  
2. **Data Cleaning & Preparation** – Filtered out test transactions, $0 prices, duplicates.  
3. **Custom Analysis** – SQL queries for KPIs, retention, cohorts, funnels, product performance.  
4. **Visualization & Storytelling** – Google Sheets dashboards + Google Slides presentation.  
5. **Recommendations** – Business-focused, data-driven actions.  

---

## 🛠️ Tools & Technologies  
- **BigQuery (SQL)** – Data extraction, cleaning, and analysis  
- **Google Sheets** – Pivot tables, charts, dashboards  
- **Google Slides** – Professional storytelling & presentation  
- **GitHub** *(optional)* – Documentation & version control  

---

## 📂 Project Workflow  

### 1. 🔍 Data Assessment & Cleaning  
- Checked for **NULL values** in users, orders, products  
- Removed **duplicate orders**  
- Flagged anomalies (e.g., **negative sale prices, future-dated transactions**)  

### 2. 📈 Business KPI Analysis  
- Monthly Revenue Growth  
- Orders Trend  
- Average Order Value (AOV)  
- Seasonal Sales Peaks  

### 3. 🛒 Product Performance & Pareto (80/20)  
- Ranked products by **revenue contribution**  
- Top **20% products = 80% of revenue**  
- Strategy recommendations:  
  - **Category A:** Top performers → scale marketing & supply  
  - **Category C:** Long-tail → bundling or discounting  

### 4. 👥 Customer Retention & Cohort Analysis  
- Built cohort tables by **first purchase month**  
- Found **~35% retention** from Month 1 → Month 2 (Jan 2022 cohort)  
- Recommended **post-purchase engagement campaigns**  

### 5. 🔻 Funnel Drop-off Analysis  
- Funnel: **View → Add to Cart → Purchase**  
- Identified **15% drop-off** at Add-to-Cart → Purchase  
- Suggested **A/B testing, UX optimization, cart recovery emails**  

### 6. ✅ Data Quality & Anomalies  
- Found **0.1% test data with $0 sale price**  
- Excluded anomalies → ensured **data integrity**  

---

## 📊 Key Insights  

- **Business KPIs:** Revenue & Orders rising steadily, stable AOV, seasonal peaks in **November**.  
- **Pareto Analysis:** Top **20% products drive ~80% of revenue**.  
- **Customer Cohorts:** 35% retention Month 1 → Month 2 (Jan 2022).  
- **Funnel:** 10% View-to-Cart, 85% Cart-to-Buy → bottleneck at **product page**.  
- **Data Quality:** Minimal anomalies; test data excluded.  

---

## SQL
```
-- 01_basics.sql
-- Quick exploration of The Look e-commerce public dataset in BigQuery
-- Dataset: bigquery-public-data.thelook_ecommerce

-- 1) Preview tables
-- (Run these separately as needed)
-- SELECT * FROM `bigquery-public-data.thelook_ecommerce.users` LIMIT 10;
-- SELECT * FROM `bigquery-public-data.thelook_ecommerce.orders` LIMIT 10;
-- SELECT * FROM `bigquery-public-data.thelook_ecommerce.order_items` LIMIT 10;
-- SELECT * FROM `bigquery-public-data.thelook_ecommerce.products` LIMIT 10;
-- SELECT * FROM `bigquery-public-data.thelook_ecommerce.events` LIMIT 10;
-- SELECT * FROM `bigquery-public-data.thelook_ecommerce.inventory_items` LIMIT 10;
-- SELECT * FROM `bigquery-public-data.thelook_ecommerce.distribution_centers` LIMIT 10;

-- 2) Date ranges
SELECT
  'orders' AS table_name,
  MIN(created_at) AS min_date,
  MAX(created_at) AS max_date
FROM `bigquery-public-data.thelook_ecommerce.orders`
UNION ALL
SELECT
  'events' AS table_name,
  MIN(created_at) AS min_date,
  MAX(created_at) AS max_date
FROM `bigquery-public-data.thelook_ecommerce.events`;

-- 3) Basic counts
SELECT
  (SELECT COUNT(*) FROM `bigquery-public-data.thelook_ecommerce.users`) AS users,
  (SELECT COUNT(*) FROM `bigquery-public-data.thelook_ecommerce.orders`) AS orders,
  (SELECT COUNT(*) FROM `bigquery-public-data.thelook_ecommerce.order_items`) AS order_items,
  (SELECT COUNT(*) FROM `bigquery-public-data.thelook_ecommerce.products`) AS products;


-- 02_kpis.sql
-- Company-level KPIs (monthly). Adjust DATE range filters as needed.

DECLARE start_date DATE DEFAULT DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY);
DECLARE end_date DATE DEFAULT CURRENT_DATE();

WITH order_rev AS (
  SELECT
    DATE_TRUNC(DATE(o.created_at), MONTH) AS month,
    COUNT(DISTINCT o.order_id) AS orders,
    SUM(oi.sale_price) AS revenue
  FROM `bigquery-public-data.thelook_ecommerce.orders` o
  JOIN `bigquery-public-data.thelook_ecommerce.order_items` oi
    ON o.order_id = oi.order_id
  WHERE DATE(o.created_at) BETWEEN start_date AND end_date
    AND LOWER(o.status) IN ('complete','shipped','delivered','fulfilled','closed')
  GROUP BY 1
),
customers AS (
  SELECT
    DATE_TRUNC(DATE(created_at), MONTH) AS month,
    COUNT(DISTINCT user_id) AS active_customers
  FROM `bigquery-public-data.thelook_ecommerce.orders`
  WHERE DATE(created_at) BETWEEN start_date AND end_date
  GROUP BY 1
)
SELECT
  r.month,
  r.orders,
  r.revenue,
  SAFE_DIVIDE(r.revenue, r.orders) AS avg_order_value,
  c.active_customers
FROM order_rev r
LEFT JOIN customers c USING (month)
ORDER BY month;

-- 03_product_performance.sql
-- Product/category performance with Pareto (80/20) view.

DECLARE start_date DATE DEFAULT DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY);
DECLARE end_date   DATE DEFAULT CURRENT_DATE();

WITH line_items AS (
  SELECT
    oi.product_id,
    p.category AS category,
    p.department AS department,
    p.brand AS brand,
    SUM(oi.sale_price) AS revenue,
    COUNT(oi.id) AS units   -- each order_item row = 1 unit
  FROM `bigquery-public-data.thelook_ecommerce.order_items` oi
  JOIN `bigquery-public-data.thelook_ecommerce.orders` o
    ON o.order_id = oi.order_id
  JOIN `bigquery-public-data.thelook_ecommerce.products` p
    ON p.id = oi.product_id
  WHERE DATE(o.created_at) BETWEEN start_date AND end_date
    AND LOWER(o.status) IN ('complete','shipped','delivered','fulfilled','closed')
  GROUP BY 1,2,3,4
),
ranked AS (
  SELECT
    *,
    RANK() OVER(ORDER BY revenue DESC) AS rev_rank,
    SUM(revenue) OVER() AS total_rev,
    SUM(revenue) OVER(ORDER BY revenue DESC) / NULLIF(SUM(revenue) OVER(),0) AS cum_rev_share
  FROM line_items
)
SELECT
  product_id, category, department, brand, units, revenue,
  rev_rank,
  cum_rev_share,
  CASE WHEN cum_rev_share <= 0.8 THEN 'Top 80% revenue (A)'
       WHEN cum_rev_share <= 0.95 THEN 'Next 15% (B)'
       ELSE 'Tail (C)' END AS pareto_bucket
FROM ranked
ORDER BY revenue DESC
LIMIT 500;

-- 04_customer_behavior.sql
-- Cohort: first purchase month vs subsequent retention + change column
WITH purchases AS (
  SELECT
    user_id,
    DATE_TRUNC(DATE(created_at), MONTH) AS order_month
  FROM `bigquery-public-data.thelook_ecommerce.orders`
  WHERE LOWER(status) IN ('complete','shipped','delivered','fulfilled','closed')
),
firsts AS (
  SELECT user_id, MIN(order_month) AS cohort_month
  FROM purchases
  GROUP BY 1
),
joined AS (
  SELECT f.cohort_month, p.order_month, p.user_id
  FROM firsts f
  JOIN purchases p USING (user_id)
),
cohort AS (
  SELECT
    cohort_month,
    DATE_DIFF(order_month, cohort_month, MONTH) AS months_since_first,
    COUNT(DISTINCT user_id) AS active_users
  FROM joined
  GROUP BY 1,2
),
with_change AS (
  SELECT
    cohort_month,
    months_since_first,
    active_users,
    active_users - LAG(active_users) OVER (
      PARTITION BY cohort_month ORDER BY months_since_first
    ) AS change
  FROM cohort
)
SELECT *
FROM with_change
ORDER BY cohort_month, months_since_first;
SELECT
  p.name AS product_name,
  ROUND(SUM(oi.sale_price), 2) AS total_revenue,
  COUNT(oi.id) AS units_sold
FROM `bigquery-public-data.thelook_ecommerce.order_items` AS oi
JOIN `bigquery-public-data.thelook_ecommerce.products` AS p
  ON oi.product_id = p.id
WHERE oi.status NOT IN ('Cancelled','Returned')   -- optional filter
GROUP BY product_name
ORDER BY total_revenue DESC
LIMIT 10;

-- 05_funnel_analysis.sql
-- Behavioral funnel using events table: view -> add_to_cart -> purchase

DECLARE start_date DATE DEFAULT DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY);
DECLARE end_date   DATE DEFAULT CURRENT_DATE();

WITH events AS (
  SELECT
    user_id,
    DATE(created_at) AS event_date,
    LOWER(event_type) AS event_type
  FROM `bigquery-public-data.thelook_ecommerce.events`
  WHERE DATE(created_at) BETWEEN start_date AND end_date
),
sessionized AS (
  SELECT
    user_id, event_date,
    COUNTIF(event_type = 'view')        AS views,
    COUNTIF(event_type = 'add_to_cart') AS adds,
    COUNTIF(event_type = 'purchase')    AS purchases
  FROM events
  GROUP BY 1,2
),
daily AS (
  SELECT
    event_date,
    SUM(CASE WHEN views > 0 THEN 1 ELSE 0 END) AS viewers,
    SUM(CASE WHEN adds > 0 THEN 1 ELSE 0 END) AS adders,
    SUM(CASE WHEN purchases > 0 THEN 1 ELSE 0 END) AS buyers
  FROM sessionized
  GROUP BY 1
)
SELECT
  event_date,
  viewers,
  adders,
  buyers,
  SAFE_DIVIDE(adders, viewers)  AS view_to_add_rate,
  SAFE_DIVIDE(buyers, adders)   AS add_to_buy_rate,
  SAFE_DIVIDE(buyers, viewers)  AS view_to_buy_rate
FROM daily
ORDER BY event_date;

-- 06_inventory_supply.sql
-- Inventory health: basic sell-through & potential stock pressure by category.

DECLARE start_date DATE DEFAULT DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY);
DECLARE end_date   DATE DEFAULT CURRENT_DATE();

WITH sales AS (
  SELECT
    oi.product_id,
    COUNT(oi.id) AS units_sold,
    SUM(oi.sale_price) AS revenue
  FROM `bigquery-public-data.thelook_ecommerce.order_items` oi
  JOIN `bigquery-public-data.thelook_ecommerce.orders` o
    ON o.order_id = oi.order_id
  WHERE DATE(o.created_at) BETWEEN start_date AND end_date
    AND LOWER(o.status) IN ('complete','shipped','delivered','fulfilled','closed')
  GROUP BY 1
),
inv AS (
  -- If inventory_items has stock levels, adjust accordingly; this version just counts records as proxy.
  SELECT
    product_id,
    COUNT(*) AS inventory_records
  FROM `bigquery-public-data.thelook_ecommerce.inventory_items`
  WHERE DATE(created_at) <= end_date
  GROUP BY 1
),
joined AS (
  SELECT
    p.id AS product_id,
    p.category,
    p.department,
    COALESCE(s.units_sold,0) AS units_sold,
    COALESCE(s.revenue,0) AS revenue,
    COALESCE(i.inventory_records,0) AS inventory_records
  FROM `bigquery-public-data.thelook_ecommerce.products` p
  LEFT JOIN sales s ON p.id = s.product_id
  LEFT JOIN inv   i ON p.id = i.product_id
)
SELECT
  category, department,
  SUM(units_sold) AS units_sold,
  SUM(revenue) AS revenue,
  SUM(inventory_records) AS inventory_records,
  SAFE_DIVIDE(SUM(units_sold), NULLIF(SUM(inventory_records),0)) AS sell_through_proxy
FROM joined
GROUP BY 1,2
ORDER BY revenue DESC;
```
![Visuals]()

## 📦 Deliverables  
- 📜 **SQL Scripts** → Analysis queries (BigQuery)  
- 📊 **Google Sheets Dashboard** → KPIs, retention, funnel charts
- 🎤 **Presentation (Google Slides/Docs)** → Business story + recommendations
   

---

## 🧩 Skills Demonstrated  
- SQL programming (CTEs, window functions, date logic)  
- Data cleaning & anomaly detection  
- Cohort retention & funnel analysis  
- Data visualization (Sheets dashboards)  
- Storytelling with data (Slides presentation)  
- Business impact & actionable recommendations  

---

## 🚀 Next Steps  
- Expand segmentation (**RFM analysis**)  
- Predictive modeling for **customer churn** (Python/ML)  
- Automate reporting pipeline with **scheduled queries**  

---

## 📬 Contact  
👤 **Your Name**  
📧 Email: `rithikaramalingam@gmail.com`  
🔗 [LinkedIn](https://www.linkedin.com/in/rithika-ramalingam-r-02714b244/)  
💻 GitHub: `https://github.com/YourGitHub`  

---

📝 **License:** MIT (or choose another license)  

---

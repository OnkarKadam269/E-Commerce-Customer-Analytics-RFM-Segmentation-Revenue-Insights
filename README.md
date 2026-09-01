# Olist E-Commerce Customer Analytics & Segmentation

End-to-end analytics project on the Olist Brazilian e-commerce dataset — data cleaning and feature engineering in Python, RFM customer segmentation, business-question analysis in SQL, and insight delivery via a Power BI dashboard.

---

## 1. Business Problem

Olist is a Brazilian e-commerce marketplace connecting sellers to major online sales channels. This project answers three core business questions:

1. **Where is revenue actually coming from** — which categories, sellers, and time periods drive sales?
2. **Who are the valuable customers**, and are they buying again — or is the business acquiring one-time buyers and losing them?
3. **Is order fulfillment (approval + delivery speed) hurting customer satisfaction?**

---

## 2. Dataset

Public Olist Brazilian E-Commerce dataset — 8 raw CSVs, ~99,441 orders, ~96,096 unique customers, ~113,425 order-items.

| File | Contents |
|---|---|
| `olist_orders_dataset` | Order status + timestamps (purchase, approval, delivery) |
| `olist_customers_dataset` | Customer IDs + location |
| `olist_order_items_dataset` | Item-level price, freight, product/seller link |
| `olist_order_payments_dataset` | Payment type, installments, value |
| `olist_order_reviews_dataset` | Review score, comments |
| `olist_products_dataset` | Product category, dimensions, weight |
| `olist_sellers_dataset` | Seller ID + location |
| `product_category_name_translation` | Portuguese → English category mapping |

---

## 3. Tech Stack

**Python (Pandas)** for cleaning & RFM · **MySQL** for the query layer · **Power BI** for the dashboard · **PowerPoint** for stakeholder delivery.

---

## 4. Pipeline Architecture

```
8 Raw CSVs
   │
   ▼
[Python/Pandas] ── Clean, dedupe, merge, feature engineer
   │
   ├──► master_item_level.csv     (item grain)
   ├──► master_order_level.csv    (order grain + RFM segment)
   └──► rfm_customer_segments.csv (customer grain)
   │
   ▼
[MySQL] ── orders_master | items_master | customer_segments
   │
   ▼
[SQL] ── Business question queries
   │
   ▼
[Power BI + PPT] ── Dashboard & stakeholder narrative
```

---

## 5. Data Cleaning & Feature Engineering (Python)

- **`customer_id` ≠ `customer_unique_id`.** One person can generate multiple `customer_id`s across orders. All customer-level work uses `customer_unique_id`.
- **Payments aggregated to order-level before merging** — the payments table has more rows than orders (split/installment payments). Summed via `groupby('order_id')` into `total_payment_value` before joining, to avoid inflating order counts.
- **Reviews deduplicated** to one row per `order_id`.
- **Missing product category** (~610 products) filled as `'unknown'` rather than dropped.
- **Orphan orders** (zero matching order-items — mostly `canceled`/`unavailable`) kept with NaN item data instead of fabricated values.
- **Delivery delay** = `order_delivered_customer_date − order_estimated_delivery_date` (positive = late).
- **Approval time** = `order_approved_at − order_purchase_timestamp`, in hours.
- **Outlier orders** (approval > 72 hrs, delivery > 30 days late) **flagged, not deleted** — preserves real logistics-failure signal.

---

## 6. RFM Customer Segmentation

**Why order-level, not item-level:** Computing RFM on the item-level table double-counts `monetary` for multi-item orders. Rebuilding on `order_level` (one row per order) corrected the monetary mean from $213 → $166.66 — a ~22% overstatement that would have skewed every segment.

**Scoring approach:**
- Recency & Monetary → `pd.qcut()` (5 quantile bins)
- Frequency → manual bins, not quantiles — 96.88% of customers are one-time buyers, so quantile splitting would collapse into one bucket
- `frequency ≥ 2` → automatic **Repeat Buyer** segment

**Results (from executed notebook):**

| Segment | Customers | Avg Spend | Total Revenue |
|---|---|---|---|
| Recent High-Value (One-time) | 14,839 | $304.79 | $4.52M |
| Lapsed High-Value (One-time) | 14,141 | $311.59 | $4.41M |
| Mid-Value (One-time) | 18,414 | $152.03 | $2.80M |
| Recent Low-Value (One-time) | 22,470 | $73.12 | $1.64M |
| Lost / Low Engagement | 23,235 | $72.88 | $1.69M |
| Repeat Buyer | 2,997 | $314.99 | $0.94M |

---

## 7. SQL Analysis

Run against `orders_master`, `items_master`, `customer_segments` in MySQL.

**Q1 — Revenue by customer segment**
```sql
SELECT segment, 
       COUNT(DISTINCT customer_unique_id) AS customers,
       ROUND(SUM(monetary),2) AS total_revenue,
       ROUND(AVG(monetary),2) AS avg_spend
FROM orders_master
GROUP BY segment
ORDER BY total_revenue DESC;
```

**Q2 — Top 10 product categories by revenue**
```sql
SELECT product_category_name_english, 
       ROUND(SUM(price + freight_value),2) AS total_revenue
FROM items_master
GROUP BY product_category_name_english
ORDER BY total_revenue DESC
LIMIT 10;
```

**Q3 — Monthly revenue trend**
```sql
SELECT order_month, ROUND(SUM(total_payment_value),2) AS revenue
FROM orders_master
GROUP BY order_month
ORDER BY order_month;
```

**Q4 — Seller performance**
```sql
SELECT seller_id,
       ROUND(SUM(price + freight_value),2) AS revenue,
       COUNT(DISTINCT order_id) AS orders,
       ROUND(AVG(delivery_delay_days),1) AS avg_delivery_delay,
       ROUND(AVG(review_score),2) AS avg_review_score
FROM items_master
GROUP BY seller_id
ORDER BY revenue DESC
LIMIT 20;
```

**Q5 — Does late delivery hurt review scores?**
```sql
SELECT delivery_delay_flag, 
       ROUND(AVG(review_score),2) AS avg_review_score, 
       COUNT(*) AS orders
FROM items_master
WHERE review_score IS NOT NULL
GROUP BY delivery_delay_flag;
```

**Q6 — Revenue stuck in high-value one-time buyers (churn risk)**
```sql
SELECT ROUND(SUM(monetary),2) AS high_value_onetime_revenue
FROM orders_master
WHERE segment IN ('Lapsed High-Value (One-time)','Recent High-Value (One-time)');
```

**Q7 — Payment method patterns**
```sql
SELECT payment_type, 
       COUNT(*) AS orders, 
       ROUND(AVG(total_payment_value),2) AS avg_order_value
FROM orders_master
GROUP BY payment_type
ORDER BY orders DESC;
```

**Q8 — Order status breakdown (cancellation impact)**
```sql
SELECT order_status, COUNT(*) AS order_count
FROM items_master
GROUP BY order_status
ORDER BY order_count DESC;
```



---

## 8. Power BI Dashboard — Build Steps

1. **Get Data from MySQL** — Home → Get Data → MySQL database → connect to `ecommerce_analysis` → select `orders_master`, `items_master`, `customer_segments` → Transform Data.
2. **Clean in Power Query Editor** — fix any date column showing as text, remove unused columns, Close & Apply.
3. **Build relationships** — in Model view, confirm `order_id` links `items_master` ↔ `orders_master` (one-to-many). Drag manually if missing.
4. **Create DAX measures** —
   ```
   Total Revenue = SUM(orders_master[total_payment_value])
   Total Customers = DISTINCTCOUNT(orders_master[customer_unique_id])
   Repeat Rate = DIVIDE(COUNTROWS(FILTER(customer_segments, customer_segments[frequency]>=2)), COUNTROWS(customer_segments))
   ```
5. **Add KPI cards** — Card visual for Total Revenue, Total Customers, Repeat Rate along the top.
6. **Build main charts** — Bar chart (segment vs. revenue), Line chart (month vs. revenue), Treemap (category vs. revenue).
7. **Add slicers** — for `segment` and `order_month`/`order_year` so viewers can filter interactively.
8. **Format & publish** — apply a theme, add a title text box, then File → Publish or Export to PDF/PPT.



---

## 9. Key Business Insights

1. **Revenue is concentrated in one-time high spenders, not repeat buyers.** "Recent High-Value" + "Lapsed High-Value" segments together generate **~$8.9M** — nearly half of measured revenue — from customers who bought only once.
2. **Retention, not acquisition, is the core weakness.** Only 2,997 customers (≈3%) are repeat buyers, despite having the highest average spend ($314.99).
3. **"Lost/Low Engagement" is the largest segment by count (23,235)** but the lowest revenue efficiency — low priority for retention spend.
4. **~$4.4M in "Lapsed High-Value" revenue is at active churn risk** — recent enough to still have a marketing window.

---

## 10. Recommendations

- Launch a **win-back campaign targeted at "Lapsed High-Value"** customers — proven spenders, unlike net-new acquisition.
- Introduce **post-purchase engagement** (loyalty incentive, second-purchase discount) after a customer's first high-value order.
- Investigate **delivery delay's effect on repeat purchase rate** (SQL Q5) — if late delivery suppresses return visits, fixing logistics could lift retention directly.
- Deprioritize marketing spend on "Lost/Low Engagement" — low historical value means weak ROI on win-back.

---

## 11. Limitations & Future Work

- RFM segments are rule-based, not learned — a K-Means clustering pass on the same R/F/M features could validate segment boundaries.
- `order_status` filtering should be verified consistently across every SQL query.
- No cohort/retention curve analysis yet.
- MySQL credentials must move to environment variables (`os.environ`) before this project is shared publicly.

---

## 12. Repo Structure

```
├── code.ipynb                      # Full Python cleaning + RFM pipeline
├── master_item_level.csv           # Item-grain export
├── master_order_level.csv          # Order-grain export (+ RFM segment)
├── rfm_customer_segments.csv       # Customer-grain RFM table
├── sql_queries.sql                 # Business question queries
├── dashboard.pbix                  # Power BI dashboard
└── presentation.pptx               # Stakeholder summary deck
```

# TheLook ecommerce: SQL analytics portfolio

**SQL (BigQuery) analysis of the public `thelook_ecommerce` dataset**: Customer lifecycle & Recency, Frequency and Monetary (RFM) segmentation, purchase funnel analysis, churn drivers, product analytics and revenue forecasting.

## 1. About the project

The public dataset simulates an ecommerce company. Using SQL (BigQuery), the project moves from raw orders and events data to insights. It covers the follwing topics:

1) **Business growth & revenue trends**: Monthly revenue, year-on-year growth, new vs. returning customer revenue
2) **RFM customer segmentation**: RFM scoring, ifecycle based customer segments
3) **Funnel & conversion analysis**: Funnels (product → basket → purchase drop-off, session-level and intent-based), segmented by traffic source, browser, city as well as session duration
4) **Churn analysis**: Covers behavioural drivers of customer churn
5) **Product analytics**: Incl. market basket, product affinity and repeat-purchase drivers
6) **Revenue forecasting**: Build a trend and seasonality decomposition model with a 12-month forecast, incl. model evaluation and finally limitations

All queries are written in standard SQL against BigQuery's public dataset. Query outputs are included as screenshots directly beneath the relevant query. Each section carries a one-line summary of what it covers.

### 2. Key findings

- Around 70% of orders contain a single item & low basket size, high-frequency purchasing behaviour
- Conversion rate is relatively flat, around 26%, across traffic source and browser. However, it varies with session duration. Engagement time is the strongest behavioural signal identified
- In the funnel, basket abandonment was identified as the main drop-off point 
- Revenue shows consistent long-term growth with a clear seasonal pattern, eg. weaker Q1, stronger Q4.

### 3. Limitations

- Various analyses (e.g. flat conversion rates across traffic sources) indicate the dataset is synthetic or was normalised, rather than reflecting a real-world business 
- The forecasting model assumes a linear trend and fixed seasonality. Specifically, it does not account for structural breaks or external drivers such as promotions, holidays or pricing changes
- Segment sizes vary by geography. Therefore, city-level findings may carry some noise

## 4. The relationships

*How the five core tables in the dataset (`users`, `orders`, `order_items`, `products`, `events`) relate to one another.*

## Core schema

## 4.1. Users

- One row = one customer
- It contains e.g. acquisition channel, country, age, gender or traffic source

Questions it answers:

- Where did users come from?
- Which channels bring valuable users?

## 4.2. Orders

One row = one order

It includes, 
- order timestamp
- user_id
- status
- total sale amount

Questions:
- How often do users buy?
- Repeat purchase behaviour?
- Revenue trends?

## 4.3. Order_items

One row = one product inside an order

Relevant :
- one order can contain multiple products

Questions:
- Basket analysis
- Product affinity
- Category revenue

## 4.4. Products

Product metadata:

- category
- brand
- cost
- retail price

Questions:

- Which categories drive profit?
- Which products create repeat behaviour?

## 4.5. Events
Includes:
- page views
- sessions
- carts
- clicks
- timestamps

This enables:

- attribution
- funnel analysis
- behavioural segmentation
- causal reasoning

SQL queries:

```sql
-- 1) Data exploration
-- 1.1. overview of order dynamics:
SELECT 'distribution_centers' AS table_name, COUNT(*) AS row_count
FROM `bigquery-public-data.thelook_ecommerce.distribution_centers`
UNION ALL
SELECT 'users' AS table_name, COUNT(*) AS row_count
FROM `bigquery-public-data.thelook_ecommerce.users`
UNION ALL
SELECT 'orders' AS table_name, COUNT(*) AS row_count
FROM `bigquery-public-data.thelook_ecommerce.orders`
UNION ALL
SELECT 'order_items' AS table_name, COUNT(*) AS row_count
FROM `bigquery-public-data.thelook_ecommerce.order_items`
UNION ALL
SELECT 'events' AS table_name, COUNT(*) AS row_count
FROM `bigquery-public-data.thelook_ecommerce.events`;
-- the numbers suggest that users are more likely to place orders
containing a single item. 181422 order items / 125180 orders ≈1.45 ->
```

each order contains about 1.45 items on avg.

*-- That implies: 1. many orders contain 1 item, 2. some contain 2 or*
*more items, 3. large multi-item carts are probably uncommon*

![Query result screenshot](images/image9.png)

```sql
-- 1.2.
WITH order_sizes AS (
SELECT
order_id,
COUNT(*) AS items_per_order
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY order_id
)
SELECT
items_per_order,
COUNT(*) AS number_of_orders,
ROUND(
100 * COUNT(*) / SUM(COUNT(*)) OVER (),
2
) AS percentage_of_orders
FROM order_sizes
GROUP BY items_per_order
ORDER BY items_per_order;
```

![Query result screenshot](images/image15.png)

- About **70% of all orders contain only one item**.
- Roughly **20% contain two items**.
- Orders with **3 or more items are relatively rare** (around 10%
    > combined).

This indicates that customer purchasing behaviour is dominated by quick,
low-basket-size transactions rather than large shopping carts.

```sql
-- 2.3. What does the cohort look like?
SELECT
DATE_TRUNC(created_at, MONTH) AS month,
SUM(sale_price) AS revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY month
ORDER BY month;
```

![Query result screenshot](images/image25.png)

### 1. Strong long-term growth

Revenue increases consistently from 2020 -> 2025, with no prolonged
declines.

- Early period (2020-2021): low but steadily rising revenue
- Mid period (2022-2023): moderate, stable growth
- Recent period (2024-2025): **sharp acceleration**

**->** This suggests the business is scaling rather than stagnating.

### 2. Growth is accelerating is not linear

The curve is upward, this indicates:

- Growth rate is increasing over time -> Each year contributes more revenue than the previous one

### 3. Strong spike in the latest months

The last few bars show a noticeable jump ( roughly 500-600k range):

- Consistent month-over-month (MoM) expansion
- No visible long-term downturns
- Strong recent momentum

```sql
-- 2.4. Revenue growth decomposition
select
DATE_TRUNC(created_at, MONTH) AS month,
SUM(sale_price) AS revenue,
SUM(sale_price) / count(distinct user_id) as revenue_per_user,
count(distinct user_id) as total_unique_users,
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY month
ORDER BY month;
```

Revenue growth is driven primarily by user growth, not higher spend per user:

- total_unique_users grows massively over time (increased roughly: from 12 to 4375 -> that's a  about a 365x increase)
- revenue_per_user stays relatively stable (80 to 90)

```sql
-- 2.5. is growth is driven by acquisition? e.g. monthly new users
with first_purchase_users as(
select
user_id,
MIN(DATE_TRUNC(created_at, MONTH)) AS first_month
FROM `bigquery-public-data.thelook_ecommerce.orders`
GROUP BY user_id
)
select
first_month,
count(user_id) as new_users
FROM first_purchase_users
GROUP BY first_month
ORDER BY first_month;
```

![Query result screenshot](images/image10.png)

Monthly new users have grown steadily over time, indicating that overall growth is strongly driven by user acquisition. Growth accelerated
strongly from late 2024 onward, with monthly new users rising sharply and reaching over 3k by early 2026. This suggests acquisition efforts or
market demand intensified in the most recent period

```sql
-- 2.6. new vs returning users revenue
WITH first_purchase AS (
SELECT
user_id,
MIN(DATE_TRUNC(created_at, MONTH)) AS first_month
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY user_id
),
orders AS (
SELECT
DATE_TRUNC(o.created_at, MONTH) AS month,
o.user_id,
o.sale_price,
fp.first_month
FROM `bigquery-public-data.thelook_ecommerce.order_items` o
JOIN first_purchase fp
USING(user_id)
)
SELECT
month,
SUM(
CASE WHEN month = first_month
THEN sale_price ELSE 0 END
) AS new_user_revenue,
SUM(
CASE WHEN month > first_month
THEN sale_price ELSE 0 END
) AS returning_user_revenue
FROM orders
GROUP BY month
ORDER BY month;
```

```sql
-- 2.7 How does each calendar month perform across different years?
(seasonality)
SELECT
EXTRACT(YEAR FROM created_at) AS year,
EXTRACT(MONTH FROM created_at) AS month_num,
FORMAT_TIMESTAMP('%b', created_at) AS month_name,
SUM(sale_price) AS revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY 1,2,3
ORDER BY month_num, year;
```

1)  Example January -> Every month increases year-over-year (YoY).
2)  Oct < Nov < Dec 
Likely caused by:
- holiday shopping
- Black Friday and/ or Cyber Monday
- gifting season / Christmas holidays

3)  Many years show softer February performance which may be causes by post-holiday slowdown.

```sql
-- 2.8. YoY growth
WITH monthly_revenue AS (
SELECT
EXTRACT(YEAR FROM created_at) AS year,
EXTRACT(MONTH FROM created_at) AS month_num,
FORMAT_TIMESTAMP('%b', created_at) AS month_name,
SUM(sale_price) AS revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY 1,2,3
)
SELECT
*,
LAG(revenue) OVER (
PARTITION BY month_num
ORDER BY year
) AS last_year_revenue,
SAFE_DIVIDE(
revenue - LAG(revenue) OVER (
PARTITION BY month_num
ORDER BY year
),
LAG(revenue) OVER (
PARTITION BY month_num
ORDER BY year
)
) AS yoy_growth
FROM monthly_revenue
ORDER BY month_num, year;
```

Results indicate that:

- Revenue is growing every year for every month and often at very high rates early on (2019-2020) due to a small base.
- From 2021-2024, growth stabilises to roughly +40% to +70% YoY, meaning steady expansion.
- In 2025-2026, growth re-accelerates, with some months exceeding +100% YoY, suggesting a new growth push or structural change.

What this implies:
- The business is not growing because users spend much more each year (revenue per user is mostly stable).
- It is growing mainly because of more users & strong seasonality (especially Q4 spikes).
- 2026 shows a possible step change in growth

```sql
-- 2.9. Revenue = Users * Orders per User * Average Order Value (AOV) trend
-> is revenue growth coming from price or volume?
WITH monthly AS (
SELECT
DATE_TRUNC(created_at, MONTH) AS month,
user_id,
order_id,
sale_price
FROM `bigquery-public-data.thelook_ecommerce.order_items`
)
SELECT
month,
SUM(sale_price) AS revenue,
COUNT(DISTINCT user_id) AS users,
COUNT(DISTINCT order_id) AS orders,
SUM(sale_price) / COUNT(DISTINCT order_id) AS avg_order_value,
COUNT(DISTINCT order_id) * 1.0
/ COUNT(DISTINCT user_id) AS orders_per_user,
SUM(sale_price) * 1.0
/ COUNT(DISTINCT user_id) AS revenue_per_user
FROM monthly
GROUP BY month
ORDER BY month;
```

- Assumption: Revenue = Users * Orders per user * AOV

Results show:
- Growth is mainly volume driven: User growth, followed by a late increase in orders per user, while AOV stays relatively stabl e(80-90€)
- Over time, customers are not spending more per order
- Pricing or inflation effects are limited
- basket size is relatively consistent

```sql
-- 2.10. order value distribution
with order_value_distribution as(
select
order_id,
sum(sale_price) as order_value
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY order_id)
-- bins
select
approx_quantiles(order_value, 10) as order_value_quantiles
FROM order_value_distribution;
-- 2. Cohort retention analysis
-- Retention Rate = Retained Users / Original Cohort Users
-- What is known: 1. acquisition is strong & 2. returning revenue is
increasing
-- 1. \"Active\" refers to users who made a purchase
-- 2. \"cohort\" include all users whose first purchase occurred in
January 2024
-- (for each month) belong to the January cohort.
WITH user_first_purchase AS (
-- Step 1: How many users return after first purchase? (assigning
cohorts)
SELECT
user_id,
MIN(created_at) AS first_purchase_date,
DATE_TRUNC(MIN(created_at), MONTH) AS first_cohort_month
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY user_id
),
-- step 2 denominator: Cohort size: How many users originally joined
each cohort?
cohort_size AS (
SELECT
first_cohort_month,
COUNT(DISTINCT user_id) AS cohort_users
FROM user_first_purchase
GROUP BY first_cohort_month
),
user_activity AS (
-- Step 3 (All future behaviour): track all activity per user
SELECT
o.user_id,
o.order_id,
o.created_at,
DATE_TRUNC(o.created_at, MONTH) AS activity_month,
ufp.first_cohort_month,
ufp.first_purchase_date
FROM `bigquery-public-data.thelook_ecommerce.orders` o
INNER JOIN user_first_purchase ufp
USING(user_id)
),
-- Step 4: (numerator by period) identify users active in each lifecycle month
returning_user_table AS (
SELECT
first_cohort_month,
activity_month,
COUNT(DISTINCT user_id) AS retained_users, -- users active in lifecycle
month
-- Average days since first purchase
AVG(
DATE_DIFF(
DATE(created_at),
DATE(first_purchase_date),
DAY
)
) AS avg_days_to_return,
-- lifecycle month index
DATE_DIFF(
DATE(activity_month),
DATE(first_cohort_month),
MONTH
) AS months_after_signup
FROM user_activity
-- KEEP month 0 (first purchase month)
GROUP BY first_cohort_month, activity_month
),
final_retention as(
-- step 5 (combining metrics into a reporting table): final retention
table
SELECT
ru.retained_users,
cz.first_cohort_month,
ru.activity_month,
cz.cohort_users,
ROUND(
SAFE_DIVIDE(ru.retained_users, cz.cohort_users) * 100,
2
) AS retention_rate,
ru.months_after_signup, -- lifecycle \"age\"
ru.avg_days_to_return
FROM returning_user_table ru
LEFT JOIN cohort_size cz
ON ru.first_cohort_month = cz.first_cohort_month
ORDER BY
cz.first_cohort_month,
ru.activity_month
)
-- > answers:
-- Each row means:
-- For users acquired in cohort X,
-- how many were active in lifecycle month Y?
--
-- month 0 = acquisition month / first purchase month
-- month 1+ = returning activity after acquisition
-- step6: retention matrix
-- Retention matrix
SELECT
FORMAT_DATE('%b %Y', DATE(first_cohort_month)) AS cohort,
MAX(CASE WHEN months_after_signup = 0 THEN retention_rate END) AS M0,
MAX(CASE WHEN months_after_signup = 1 THEN retention_rate END) AS M1,
MAX(CASE WHEN months_after_signup = 2 THEN retention_rate END) AS M2,
MAX(CASE WHEN months_after_signup = 3 THEN retention_rate END) AS M3,
MAX(CASE WHEN months_after_signup = 4 THEN retention_rate END) AS M4,
MAX(CASE WHEN months_after_signup = 5 THEN retention_rate END) AS M5
FROM final_retention
GROUP BY cohort, first_cohort_month
ORDER BY first_cohort_month;
```

![Query result screenshot](images/image16.png)

Takeaways:

- From 2019-2024, M1 retention was around 1-3%. This is in line with previous results where results indicated the business generated revenue with customer acquisition.
Results show:

- There are many low/mid-value purchases
- A few high-ticket orders contributing disproportionate revenue

Retention analysis aims to check:
- If growth is sustainable
- users become loyal
- future revenue compounds naturally

Without retention:
- the company is "buying growth"

With strong retention:
- revenue becomes exponentially scalable
  
```sql
-- 3.2. Which acquisition channels create the best long-term customers?
-- step 1: session-level dataset
with session_table as (
SELECT
user_id,
session_id,
traffic_source,
COUNTIF(event_type = 'department') as department_view_count,
COUNTIF(event_type = 'product') as product_view_count,
countif(event_type = 'cart') as cart_view_count,
countif(event_type = 'purchase') as purchase_view_count,
timestamp_diff(max(created_at), min(created_at), MINUTE) as
session_duration_in_minutes
FROM `bigquery-public-data.thelook_ecommerce.events`
WHERE user_id IS NOT NULL
group by
user_id,
session_id,
traffic_source),
--step 2: distribution of product views (Product view quantiles by channel)
quantiles as (
select
traffic_source,
approx_quantiles(department_view_count, 100) [OFFSET(25)] AS dept_p25,
approx_quantiles(department_view_count, 100) [OFFSET(50)] AS dept_p50,
approx_quantiles(department_view_count, 100) [OFFSET(75)] AS dept_p75,
approx_quantiles(department_view_count, 100) [OFFSET(100)] AS dept_p100,
approx_quantiles(product_view_count, 100) [OFFSET(25)] AS product_p25,
approx_quantiles(product_view_count, 100) [OFFSET(50)] AS product_p50,
approx_quantiles(product_view_count, 100) [OFFSET(75)] AS product_p75,
approx_quantiles(product_view_count, 100) [OFFSET(100)] AS product_p100,
approx_quantiles(cart_view_count, 100) [OFFSET(25)] AS cart_p25,
approx_quantiles(cart_view_count, 100) [OFFSET(50)] AS cart_p50,
approx_quantiles(cart_view_count, 100) [OFFSET(75)] AS cart_p75,
approx_quantiles(cart_view_count, 100) [OFFSET(100)] AS cart_p100,
approx_quantiles(purchase_view_count, 100) [OFFSET(25)] AS purchase_p25,
approx_quantiles(purchase_view_count, 100) [OFFSET(50)] AS purchase_p50,
approx_quantiles(purchase_view_count, 100) [OFFSET(75)] AS purchase_p75,
approx_quantiles(purchase_view_count, 100) [OFFSET(100)] AS purchase_p100,
APPROX_QUANTILES(session_duration_in_minutes, 100)[OFFSET(25)] AS session_duration_p25,
APPROX_QUANTILES(session_duration_in_minutes, 100)[OFFSET(50)] AS session_duration_50,
APPROX_QUANTILES(session_duration_in_minutes, 100)[OFFSET(75)] AS session_duration_p75,
APPROX_QUANTILES(session_duration_in_minutes, 100)[OFFSET(100)] AS session_duration_p100
from session_table
group by traffic_source
)
select *
from quantiles
order by product_p50 DESC;
```

![Query result screenshot](images/image28.png)

Across user acquisition channels (Facebook (FB), Adwords, YouTube (YT), E-Mail, Organic) user session behaviour appear similar. Session duration, purchase behaviour and browsing depth are nearly identical across sources.

```sql
-- 3.3 Which acquisition channels create the best long-term customers?
WITH user_metrics AS (
SELECT
user_id,
ANY_VALUE(traffic_source) AS traffic_source,
COUNT(DISTINCT session_id) AS sessions,
COUNTIF(event_type = 'purchase') AS purchases,
DATE_DIFF(MAX(created_at), MIN(created_at), DAY) AS lifespan_days
FROM `bigquery-public-data.thelook_ecommerce.events`
WHERE user_id IS NOT NULL
GROUP BY user_id
)
SELECT
traffic_source,
AVG(sessions) AS avg_sessions_per_user,
AVG(purchases) AS avg_purchases_per_user,
AVG(lifespan_days) AS avg_lifespan_days,
COUNT(*) AS total_users
FROM user_metrics
GROUP BY traffic_source
ORDER BY avg_purchases_per_user DESC
```

![Query result screenshot](images/image26.png)

All acquisition channels show almost similar behaviour in sessions, purchases and lifespan. Channel type doesn't look like a strong differentiator of long-term customer value in this dataset.

```sql
-- 3.4. Cohort retention by channel
WITH user_cohort AS (
SELECT
user_id,
MIN(DATE(created_at)) AS first_purchase_date,
DATE_TRUNC(MIN(DATE(created_at)), MONTH) AS first_cohort_month,
ANY_VALUE(traffic_source) AS traffic_source
FROM `bigquery-public-data.thelook_ecommerce.events`
WHERE user_id IS NOT NULL
GROUP BY user_id
),
user_activity_tracking AS (
SELECT
e.user_id,
uc.traffic_source,
uc.first_cohort_month,
DATE_TRUNC(DATE(e.created_at), MONTH) AS activity_month,
DATE_DIFF(
DATE_TRUNC(DATE(e.created_at), MONTH),
uc.first_cohort_month,
MONTH
) AS month_index
FROM `bigquery-public-data.thelook_ecommerce.events` e
JOIN user_cohort uc
USING (user_id)
),
cohort_counts AS (
SELECT
traffic_source,
first_cohort_month AS cohort_month,
month_index,
COUNT(DISTINCT user_id) AS active_users
FROM user_activity_tracking
GROUP BY 1, 2, 3
),
cohort_sizes AS (
SELECT
traffic_source,
first_cohort_month AS cohort_month,
COUNT(DISTINCT user_id) AS cohort_size
FROM user_cohort
GROUP BY 1, 2
)
SELECT
c.traffic_source,
c.month_index,
SUM(c.active_users) AS active_users,
MAX(s.cohort_size) AS cohort_size,
SAFE_DIVIDE(
SUM(c.active_users),
MAX(s.cohort_size)
) AS retention_rate
FROM cohort_counts c
JOIN cohort_sizes s
ON c.traffic_source = s.traffic_source
AND c.cohort_month = s.cohort_month
GROUP BY 1, 2
ORDER BY 1, 2;
```

## Summary of business growth 
- growth is acquisition-driven

## Customer behaviour
- small basket sizes dominate
- repeat purchasing improved over time
- retention increased after 2025

## Lifecycle understanding
- cohort analysis
- retention decay
- returning revenue contribution

## Acquisition analysis
- channels behave similarly
- no strong session-level differences
- no obvious long-term channel winner

**3. RFM Segmentation**

RFM: Recency, Frequency, Monetary.

Classification:

| Segment | Meaning |
|---|---|
| Champions | buy often, recently & show high spendings  |
| Loyal | repeat customers |
| At Risk | used to buy but disappeared |
| New Customers | recently acquired |
| Big Spenders | high monetary value |

# Metrics

*Definitions of the core RFM metrics value.*

## Recency

Days since last order.
- Recency Days = Current Date − Last Purchase Date
- Customer Lifespan = Last Purchase Date − First Purchase Date

- Frequency: Total orders.
- Monetary Total spend.

This directly supports:
- CRM targeting
- retention campaigns
- personalisation
- churn prevention


```sql
-- =========================================================
-- RFM PIPELINE
-- =========================================================
WITH
validation AS (
SELECT
COUNT(DISTINCT o.order_id) AS orders_table_orders,
COUNT(DISTINCT oi.order_id) AS order_items_orders
FROM `bigquery-public-data.thelook_ecommerce.orders` o
LEFT JOIN `bigquery-public-data.thelook_ecommerce.order_items` oi
USING(order_id)
),
orders_placed AS (
SELECT
order_id,
user_id,
DATE(created_at) AS order_date
FROM `bigquery-public-data.thelook_ecommerce.orders`
),
rfm_base AS (
SELECT
user_id,
COUNT(DISTINCT order_id) AS total_orders,
MIN(order_date) AS first_purchase_date,
MAX(order_date) AS last_purchase_date,
DATE_DIFF(CURRENT_DATE(), MAX(order_date), DAY) AS recency_days,
DATE_DIFF(MAX(order_date), MIN(order_date), DAY) AS customer_lifespan_days
FROM orders_placed
GROUP BY user_id
),
purchase_gap AS (
SELECT
user_id,
COUNT(order_id) AS total_orders,
AVG(day_since_previous_purchase) AS avg_days_between_orders,
SAFE_DIVIDE(
COUNT(order_id) * 30,
NULLIF(DATE_DIFF(MAX(order_date), MIN(order_date), DAY), 0)
) AS purchase_frequency_per_month
FROM (
SELECT
user_id,
order_id,
order_date,
LAG(order_date) OVER(PARTITION BY user_id ORDER BY order_date) AS previous_order_date,
DATE_DIFF(
order_date,
LAG(order_date) OVER(PARTITION BY user_id ORDER BY order_date),
DAY
) AS day_since_previous_purchase
FROM orders_placed
)
GROUP BY user_id
),
monetary_table AS (
SELECT
o.user_id,
o.order_id,
DATE(o.created_at) AS order_date,
o.status,
oi.sale_price
FROM `bigquery-public-data.thelook_ecommerce.orders` o
INNER JOIN `bigquery-public-data.thelook_ecommerce.order_items` oi
USING (order_id)
),
revenue_waterfall_model AS (
SELECT
user_id,
SUM(sale_price) AS total_gross_revenue,
SUM(CASE WHEN status NOT IN ('Cancelled', 'Returned') THEN sale_price ELSE 0 END) AS net_revenue,
SUM(CASE WHEN status = 'Cancelled' THEN sale_price ELSE 0 END) AS cancelled_revenue,
SUM(CASE WHEN status = 'Returned' THEN sale_price ELSE 0 END) AS returned_revenue
FROM monetary_table
GROUP BY user_id
),
final_rfm_table AS (
SELECT
rfm.user_id,
rfm.recency_days,
rfm.customer_lifespan_days,
rfm.total_orders,
pg.avg_days_between_orders,
pg.purchase_frequency_per_month,
rwm.total_gross_revenue,
rwm.net_revenue,
rwm.cancelled_revenue,
rwm.returned_revenue,
SAFE_DIVIDE(rwm.returned_revenue, rwm.total_gross_revenue) AS return_rate
FROM rfm_base rfm
LEFT JOIN purchase_gap pg USING(user_id)
LEFT JOIN revenue_waterfall_model rwm USING(user_id)
),
rfm_scoring AS (
SELECT
*,
NTILE(5) OVER (ORDER BY recency_days DESC) AS recency_score,
NTILE(5) OVER (ORDER BY purchase_frequency_per_month) AS frequency_score,
NTILE(5) OVER (ORDER BY net_revenue) AS monetary_score
FROM final_rfm_table
),
customer_segments AS (
SELECT
*,
CASE
WHEN recency_score >= 4 AND frequency_score >= 4 AND monetary_score >= 4 THEN 'VIP'
WHEN recency_score >= 3 AND frequency_score >= 4 THEN 'Loyal'
WHEN recency_score >= 4 AND total_orders <= 2 THEN 'New Customers'
WHEN recency_score <= 2 AND (frequency_score >= 3 OR monetary_score >= 3) THEN 'At Risk'
WHEN recency_score <= 2 THEN 'Churn Risk'
WHEN return_rate >= 0.30 THEN 'High Return Customers'
WHEN total_orders = 1 THEN 'One-time Buyers'
ELSE 'Regular'
END AS customer_segment
FROM rfm_scoring
)
SELECT *
FROM customer_segments;
```

![Query result screenshot](images/image8.png)

```sql
-- Validation checks
SELECT COUNT(*) AS total_rows, COUNT(DISTINCT user_id) AS unique_users
FROM final_rfm_table;
```

![Query result screenshot](images/image17.png)

79745 total rows = 79745 unique users, so there's no duplication from the joins -> grain control holds.

```sql
SELECT * FROM final_rfm_table
WHERE recency_days < 0 OR customer_lifespan_days < 0;
```

![Query result screenshot](images/image22.png)

No rows returned, shows there is no invalid date calculations (e.g. future dates or reversed lifespans).

```sql
SELECT * FROM final_rfm_table
WHERE ABS(total_gross_revenue - (net_revenue + cancelled_revenue + returned_revenue)) > 0.01;
```

![Query result screenshot](images/image22.png)

Gross revenue correctly equals the sum of net, cancelled and returned revenue — no leakage or double counting. The waterfall model is
internally consistent.

```sql
SELECT customer_segment, COUNT(*) AS customers
FROM customer_segments
GROUP BY customer_segment
ORDER BY customers DESC;
```

![Query result screenshot](images/image31.png)


*The core business questions the RFM analysis sets out to answer.*
- Which customers are most valuable?
- Which customers are likely to churn?
- Is business growth driven by acquisition or retention?
- How frequently do customers repurchase?
- How much revenue is lost through returns and cancellations?
- Which customer segments drive long-term value?

Core relationship: users -> orders -> order_items

1. RFM insights: 
Revenue growth is driven mainly by increasing user acquisition and rising order volume — average spend per customer stays relatively stable.

2. Retention & churn
 Repeat customer revenue is improving, but long-term churn risk remains substantial: acquisition is strong, retention is the bigger optimisation opportunity.

3. Customer value concentration

Customer value is uneven. a relatively small subset of customers generates disproportionate business value.

4. Revenue quality
Cancelled and returned orders materially affect realised customer value; high-return customers may reduce operational profitability.

# 4. Funnel analysis (using events)

Session -> Product view -> Add to cart -> Checkout -> Purchase

Question: which channels convert best? -> Conversion Rate = Purchasing Users / Total Visitors

```sql
-- =========================================================
-- 0) Exploration: Understanding the event structure
-- =========================================================
SELECT *
FROM `bigquery-public-data.thelook_ecommerce.events`
WHERE user_id = 77064
ORDER BY id, sequence_number ASC;

-- =========================================================
-- 1) Event distribution analysis
-- =========================================================
SELECT
event_type,
COUNT(*) AS events,
ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS percentage
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY event_type
ORDER BY events DESC;
```

Most user activity sits at the product-viewing and cart stages. Many users browse and add to cart, but only a smaller share complete a purchase, pointing to drop-off between cart and checkout (friction, abandonment, price sensitivity or low intent). That said, this dataset behaves differently from real-world ecommerce data: home page visits are relatively low, and funnel behaviour looks artificially smooth, suggesting the dataset is at least partially synthetic and probably built mainly for SQL practice rather than realistic behavioural modelling.

```sql
-- =========================================================
-- 2) Conversion metrics
-- =========================================================
SELECT
ROUND(
SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) * 100.0
/ SUM(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END),
2
) AS cart_to_purchase_rate
FROM `bigquery-public-data.thelook_ecommerce.events`;
-- Result: 30.51%

SELECT
ROUND(
SUM(CASE WHEN event_type = 'cancel' THEN 1 ELSE 0 END) * 100.0
/ SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END),
2
) AS purchase_cancellation_rate
FROM `bigquery-public-data.thelook_ecommerce.events`;
```

![Query result screenshot](images/image23.png)

Nearly 69% of purchases were cancelled, which is unusually high and worth flagging as a dataset quirk rather than a real business signal.

```sql
-- =========================================================
-- 3) Funnel drop-off analysis
-- =========================================================
WITH funnel AS (
SELECT
SUM(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END) AS product_events,
SUM(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) AS cart_events,
SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchase_events
FROM `bigquery-public-data.thelook_ecommerce.events`
)
SELECT
product_events,
cart_events,
ROUND((product_events - cart_events) * 100.0 / product_events, 2) AS product_to_cart_dropoff_percentage,
purchase_events,
ROUND((cart_events - purchase_events) * 100.0 / cart_events, 2) AS cart_to_purchase_dropoff_percentage
FROM funnel;
```

![Query result screenshot](images/image4.png)

Around 30% of product interactions didn't progress to cart (so ~70% did, which is decent engagement). Around 70% of cart interactions didn't convert to purchase (~30% did). A single user can generate multiple events, so a session or user-level funnel gives a more accurate picture of actual customer journeys.

```sql
-- =========================================================
-- 4) User-level Funnel Analysis
-- =========================================================
WITH session_funnel AS(
SELECT
session_id,
user_id,
MAX(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END) as viewed_product,
MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) as viewed_cart,
MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) as viewed_purchase
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY session_id, user_id
)
SELECT
count(*) as total_sessions,
(SUM(viewed_product) - SUM(viewed_cart)) * 100.0 / SUM(viewed_product) as product_to_cart_dropoff_percentage,
(SUM(viewed_cart) - SUM(viewed_purchase)) * 100.0 / SUM(viewed_cart) as cart_to_purchase_dropoff_percentage,
FROM session_funnel;
```

![Query result screenshot](images/image1.png)

About 36.7% of sessions that viewed a product never added anything to cart (so ~63.3% did progress to cart). Of the sessions that reached cart, around 58% didn't proceed to purchase — only ~42% of cart sessions converted.

```sql
-- =========================================================
-- 5) Foundation: Core funnel table
-- =========================================================
WITH full_session_funnel as (
SELECT
session_id,
user_id,
COUNT(*) as total_events,
TIMESTAMP_DIFF(MAX(created_at), MIN(created_at), MINUTE) as session_duration_minutes,
MAX(CASE WHEN event_type = 'department' THEN 1 ELSE 0 END) AS viewed_department,
MAX(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END) AS viewed_product,
MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) AS viewed_cart,
MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchased,
ANY_VALUE(traffic_source) as traffic_source,
ANY_VALUE(browser) as browser,
ANY_VALUE(city) as city,
ANY_VALUE(postal_code) as postal_code
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY session_id, user_id
)
```

This table is 1 row = 1 session, combining engagement intensity (total_events), engagement depth (session_duration_minutes), browsing breadth (viewed_department), shopping intent (viewed_product), purchase intent (viewed_cart) and conversion outcome (purchased) into one clean session-quality table.

![Query result screenshot](images/image3.png)

```sql
-- =========================================================
-- 6) Segment conversion: Funnel by traffic source
-- =========================================================
SELECT
traffic_source,
COUNT(*) AS total_sessions,
ROUND((SUM(viewed_product) / COUNT(*)) * 100, 2) AS product_rate_percentage,
ROUND((SUM(viewed_cart) / COUNT(*)) * 100, 2) AS cart_rate_percentage,
ROUND((SUM(purchased) / COUNT(*)) * 100, 2) AS purchase_rate_percentage,
ROUND((SUM(viewed_product) - SUM(viewed_cart)) * 100.0 / SUM(viewed_product), 2) AS product_to_cart_dropoff_percentage,
ROUND((SUM(viewed_cart) - SUM(purchased)) * 100.0 / SUM(viewed_cart), 2) AS cart_to_purchase_dropoff_percentage
FROM full_session_funnel
GROUP BY traffic_source
ORDER BY purchase_rate_percentage DESC;
```

![Query result screenshot](images/image12.png)

Traffic sources behave similarly  across the whole funnel. This means conversion differs marginally, suggesting traffic source has
little impact on conversion in this dataset. Users reach product pages and add to cart similarly, which points to the checkout experience itself as the bottleneck rather than the acquisition channel. Organic traffic has the highest cart rate (around 64%) but the lowest purchase rate ( around 26%) — a small gap, and onen more data point supporting the "partially synthetic dataset" read.

```sql
WITH session_funnel AS (
SELECT
session_id,
user_id,
COUNT(*) AS total_events,
TIMESTAMP_DIFF(MAX(created_at), MIN(created_at), MINUTE) AS session_duration_minutes,
MAX(CASE WHEN event_type = 'department' THEN 1 ELSE 0 END) AS viewed_department,
MAX(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END) AS viewed_product,
MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) AS viewed_cart,
MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchased,
ANY_VALUE(traffic_source) AS traffic_source,
ANY_VALUE(browser) AS browser,
ANY_VALUE(city) AS city
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY session_id, user_id
),
traffic_source_funnel AS (
SELECT
traffic_source AS segment,
COUNT(*) AS total_sessions,
ROUND(SUM(viewed_product) * 100.0 / COUNT(*), 2) AS product_rate,
ROUND(SUM(viewed_cart) * 100.0 / COUNT(*), 2) AS cart_rate,
ROUND(SUM(purchased) * 100.0 / COUNT(*), 2) AS purchase_rate,
ROUND((SUM(viewed_product) - SUM(viewed_cart)) * 100.0 / SUM(viewed_product), 2) AS product_to_cart_dropoff,
ROUND((SUM(viewed_cart) - SUM(purchased)) * 100.0 / SUM(viewed_cart), 2) AS cart_to_purchase_dropoff
FROM session_funnel
GROUP BY traffic_source
),
browser_funnel AS (
SELECT browser AS segment, COUNT(*) AS total_sessions,
ROUND(SUM(purchased) * 100.0 / COUNT(*), 2) AS purchase_rate
FROM session_funnel GROUP BY browser
),
city_funnel AS (
SELECT city AS segment, COUNT(*) AS total_sessions,
ROUND(SUM(purchased) * 100.0 / COUNT(*), 2) AS purchase_rate
FROM session_funnel GROUP BY city
HAVING COUNT(*) > 1000
),
duration_funnel AS (
SELECT
CASE
WHEN session_duration_minutes = 0 THEN '0 min'
WHEN session_duration_minutes BETWEEN 1 AND 5 THEN '1-5 min'
WHEN session_duration_minutes BETWEEN 6 AND 15 THEN '6-15 min'
WHEN session_duration_minutes BETWEEN 16 AND 30 THEN '16-30 min'
ELSE '30+ min'
END AS segment,
COUNT(*) AS total_sessions,
ROUND(AVG(session_duration_minutes), 2) AS avg_duration,
ROUND(SUM(purchased) * 100.0 / COUNT(*), 2) AS purchase_rate
FROM session_funnel GROUP BY segment
)
SELECT 'traffic_source' AS dimension, segment, total_sessions, purchase_rate FROM traffic_source_funnel
UNION ALL
SELECT 'browser' AS dimension, segment, total_sessions, purchase_rate FROM browser_funnel
UNION ALL
SELECT 'city' AS dimension, segment, total_sessions, purchase_rate FROM city_funnel
UNION ALL
SELECT 'session_duration' AS dimension, segment, total_sessions, purchase_rate FROM duration_funnel;
```

## Funnel analysis interpretation

A session-level funnel table was built to see how users move through the ecommerce journey from browsing to purchase, capturing engagement depth, conversion outcomes and attributes like traffic source, browser and city. Conversion behaviour was segmented across traffic sources, browsers, cities and session duration groups.

 Facebook | 26.63% |
| YouTube | 26.53% |
| Adwords | 26.53% |
| Email | 26.52% |
| Organic | 26.38% |

This small a spread suggests acquisition source has minimal impact on conversion here. The uniformity again points to a synthetic or simplified dataset.

## Browser analysis

| Browser | Purchase Rate |
|---|---|
| Safari | 26.66% |
| Chrome | 26.52% |
| Firefox | 26.47% |
| IE | 26.51% |

Browser-level analysis can usually surface technical friction or checkout issues, but the near identical rates here suggest no meaningful browser-related differences.

## Geographic analysis

| City | Purchase Rate |
|---|---|
| Maoming | 32.17% |
| San Antonio | 30.22% |
| Jinan | 30.04% |
| Qingdao | 23.02% |
| Wuhan | 24.28% |

City-level segmentation shows more variation than traffic source or browser. This may suggest geography may relate more to purchasing behaviour than acquisition channel does — though some cities have small sample sizes, so this needs a cautious read.

## Session duration analysis

| Session Duration | Purchase Rate |
|---|---|
| 0 min | 0.04% |
| 1–5 min | 63.07% |
| 6–15 min | 43.51% |
| 16–30 min | 2.65% |
| 30+ min | 39.90% |

Session duration has the strongest relationship with conversion. Very short sessions show almost no purchasing; 1–5 minute sessions convert best. Conversion dips sharply at 16–30 minutes before rising again for 30+ minute sessions. This may reflect the dataset's synthetic structure.

## Consistency check

```sql
-- Data consistency check
SELECT session_id, COUNT(DISTINCT city) AS cities_per_session
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY session_id
HAVING COUNT(DISTINCT city) > 1;
```

No rows returned — every session_id maps to exactly one city.

```sql
WITH session_funnel AS (
SELECT
session_id, user_id,
COUNT(*) AS total_events,
TIMESTAMP_DIFF(MAX(created_at), MIN(created_at), MINUTE) AS session_duration_minutes,
MAX(CASE WHEN event_type = 'department' THEN 1 ELSE 0 END) AS viewed_department,
MAX(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END) AS viewed_product,
MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) AS viewed_cart,
MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchased,
ANY_VALUE(traffic_source) AS traffic_source,
ANY_VALUE(browser) AS browser,
ANY_VALUE(city) AS city
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY session_id, user_id
),
intent_scored AS (
SELECT
*,
(viewed_department * 1 + viewed_product * 2 + viewed_cart * 4 + purchased * 6) AS intent_score,
CASE
WHEN purchased = 1 THEN 'Buyer'
WHEN viewed_cart = 1 AND purchased = 0 THEN 'Cart Abandoner'
WHEN viewed_product = 1 AND viewed_cart = 0 THEN 'Product Browser'
WHEN viewed_department = 1 AND viewed_product = 0 THEN 'Department Browser'
ELSE 'Low Intent'
END AS intent_segment
FROM session_funnel
)
SELECT
intent_segment,
COUNT(*) AS sessions,
ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS pct_of_sessions,
ROUND(AVG(intent_score), 2) AS avg_intent_score,
ROUND(AVG(total_events), 2) AS avg_events,
ROUND(AVG(session_duration_minutes), 2) AS avg_session_duration_minutes,
ROUND(SUM(purchased) * 100.0 / COUNT(*), 2) AS session_conversion_rate
FROM intent_scored
GROUP BY intent_segment
ORDER BY sessions DESC;
```

![Query result screenshot](images/image29.png)

# 6. Churn analysis

*Defining behavioural churn (there are no subscriptions in this dataset) and linking it back to RFM, funnel and intent signals.*

Churn here is behavioural, not contractual — no subscriptions in TheLook. Defined as, for example, a user inactive for 90 days. The next question is who churns, when, and what predicts it.

# Useful features

*The features available for predicting churn: order frequency, basket size, recency, categories purchased, traffic source.*

Already available: order frequency, basket size, recency, categories purchased, traffic source, this becomes predictive analytics.

**How churn connects to what's already built:**

- RFM base: Recency -> directly related to churn; Frequency -> engagement strength; Monetary -> value risk
- Funnel: drop-off behaviour, intent failure
- Intent segmentation: Buyers vs cart abandoners vs browsers

Churn = extreme low recency + low frequency + weak funnel engagement.

```sql
-- =========================================================
-- Funnel + intent + churn feature table (user-level)
-- =========================================================
WITH session_features AS (
SELECT
session_id, user_id,
MAX(CASE WHEN event_type = 'department' THEN 1 ELSE 0 END) AS viewed_department,
MAX(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END) AS viewed_product,
MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) AS viewed_cart,
MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchased
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY session_id, user_id
),
intent_scored AS (
SELECT
*,
(viewed_department * 1 + viewed_product * 2 + viewed_cart * 4 + purchased * 6) AS intent_score,
CASE
WHEN purchased = 1 THEN 'Buyer'
WHEN viewed_cart = 1 THEN 'Cart Abandoner'
WHEN viewed_product = 1 THEN 'Product Browser'
WHEN viewed_department = 1 THEN 'Department Browser'
ELSE 'Low Intent'
END AS intent_segment
FROM session_features
),
orders AS (
SELECT user_id, COUNT(DISTINCT order_id) AS total_orders, SUM(sale_price) AS total_spend
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY user_id
),
session_level AS (
SELECT
e.user_id, e.session_id,
MAX(e.created_at) AS last_event_time,
COUNT(*) AS events_per_session,
i.intent_score, i.intent_segment
FROM `bigquery-public-data.thelook_ecommerce.events` e
LEFT JOIN intent_scored i
ON e.session_id = i.session_id AND e.user_id = i.user_id
GROUP BY e.user_id, e.session_id, i.intent_score, i.intent_segment
)
SELECT
sl.user_id,
DATE_DIFF(CURRENT_DATE(), DATE(MAX(last_event_time)), DAY) AS recency_days,
CASE WHEN DATE_DIFF(CURRENT_DATE(), DATE(MAX(last_event_time)), DAY) > 90 THEN 1 ELSE 0 END AS churned,
COUNT(DISTINCT session_id) AS session_per_user,
COALESCE(MAX(o.total_orders), 0) AS total_orders,
COALESCE(MAX(o.total_spend), 0) AS total_spend,
AVG(intent_score) AS avg_intent_score,
AVG(CASE WHEN intent_segment = 'Buyer' THEN 1 ELSE 0 END) AS buyer_share,
AVG(CASE WHEN intent_segment = 'Cart Abandoner' THEN 1 ELSE 0 END) AS cart_abandoner_share,
AVG(CASE WHEN intent_segment = 'Product Browser' THEN 1 ELSE 0 END) AS product_browser_share,
AVG(CASE WHEN intent_segment = 'Department Browser' THEN 1 ELSE 0 END) AS department_browser_share,
AVG(CASE WHEN intent_segment = 'Low Intent' THEN 1 ELSE 0 END) AS low_intent_share
FROM session_level sl
LEFT JOIN orders o ON sl.user_id = o.user_id
GROUP BY user_id;
```

![Query result screenshot](images/image14.png)

![Query result screenshot](images/image6.png)

# 7. Product analytics

```sql
WITH product_daily_metrics AS (
SELECT
DATE(o.created_at) AS order_date,
oi.product_id,
p.name AS product_name,
p.category,
COUNT(*) AS units_sold,
SUM(CASE WHEN o.status NOT IN ('Returned', 'Cancelled') THEN oi.sale_price ELSE 0 END) AS net_revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items` oi
JOIN `bigquery-public-data.thelook_ecommerce.orders` o USING(order_id)
LEFT JOIN `bigquery-public-data.thelook_ecommerce.products` p ON oi.product_id = p.id
GROUP BY order_date, oi.product_id, p.name, p.category
)
SELECT product_id, product_name, category, SUM(net_revenue) AS revenue, SUM(units_sold) AS units_sold
FROM product_daily_metrics
GROUP BY product_id, product_name, category
ORDER BY revenue DESC
LIMIT 20;
```


![Query result screenshot](images/image32.png)

```sql
-- =========================================================
-- Time Series Table
-- =========================================================
WITH monthly_metrics AS(
SELECT
DATE_TRUNC(DATE(created_at), MONTH) AS month,
COUNT(DISTINCT order_id) as orders,
COUNT(DISTINCT user_id) as users,
SUM(sale_price) as gross_revenue,
SUM(CASE WHEN status NOT IN ('Cancelled', 'Returned') THEN sale_price ELSE 0 END) net_revenue,
SUM(CASE WHEN status IN ('Cancelled', 'Returned') THEN sale_price ELSE 0 END) as returned_revenue,
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY month
)
SELECT* FROM monthly_metrics
ORDER BY month ASC;
```

![Query result screenshot](images/image18.png)

```sql
-- How has the business been growing %?
SELECT
month,
gross_revenue,
LAG(gross_revenue) OVER(ORDER BY month) AS previous_months_revenue,
ROUND(
(gross_revenue - LAG(gross_revenue) OVER(ORDER BY month))
/ LAG(gross_revenue) OVER(ORDER BY month) * 100,
2
) AS percentage_revenue_growth
FROM monthly_metrics
ORDER BY month;
```

![Query result screenshot](images/image5.png)

-> It's a scaling growth system with seasonality layered on top.

```sql
-- =====================================================
-- Seasonality
-- =====================================================
WITH seasonality_base AS (
SELECT
EXTRACT(YEAR FROM DATE(created_at)) AS year,
DATE_TRUNC(DATE(created_at), MONTH) AS month,
EXTRACT(MONTH FROM DATE(created_at)) AS month_of_year,
SUM(sale_price) AS total_revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY 1, 2, 3
),
seasonality_index AS (
SELECT
year, month, month_of_year, total_revenue,
AVG(total_revenue) OVER (PARTITION BY year) AS yearly_avg_revenue
FROM seasonality_base
)
SELECT
month_of_year,
AVG(total_revenue / yearly_avg_revenue) AS seasonal_index
FROM seasonality_index
GROUP BY month_of_year
ORDER BY month_of_year;
```

![Query result screenshot](images/image30.png)

```sql
-- Year × month matrix, using normalised revenue (to remove the growth trend)
SELECT
year, month_of_year,
ROUND(total_revenue / yearly_avg_revenue, 2) AS seasonal_index
FROM seasonality_index
ORDER BY year, month_of_year;
```


```sql
-- =====================================================
-- 3-Month moving average
-- =====================================================
WITH monthly_revenue AS (
SELECT DATE_TRUNC(DATE(created_at), MONTH) AS month, SUM(sale_price) AS revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY 1
)
SELECT
month, revenue,
AVG(revenue) OVER (ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS rolling_3_month_avg
FROM monthly_revenue
ORDER BY month;
```

```sql
-- =============================================================================================
-- Time Series decomposition (Multiplicative approach: Revenue = Trend * Seasonality * Residual)
-- =============================================================================================
WITH monthly_revenue AS (
SELECT DATE_TRUNC(DATE(created_at), MONTH) AS month, SUM(sale_price) AS revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY 1
),
trend AS (
SELECT month, revenue,
AVG(revenue) OVER (ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS trend_value
FROM monthly_revenue
),
detrended AS (
SELECT month, revenue, trend_value,
EXTRACT(MONTH FROM month) AS month_of_year,
revenue / trend_value AS detrended_value
FROM trend
),
raw_seasonality AS (
SELECT month_of_year, AVG(detrended_value) AS raw_seasonal_index
FROM detrended GROUP BY 1
),
seasonality AS (
SELECT month_of_year, raw_seasonal_index / AVG(raw_seasonal_index) OVER () AS seasonal_index
FROM raw_seasonality
),
fitted AS (
SELECT
d.month, d.revenue, d.trend_value, s.seasonal_index,
d.trend_value * s.seasonal_index AS fitted_value
FROM detrended d
LEFT JOIN seasonality s ON d.month_of_year = s.month_of_year
)
SELECT month, revenue, trend_value, seasonal_index, fitted_value, revenue / fitted_value AS residual
FROM fitted
ORDER BY month;
```

![Query result screenshot](images/image21.png)
![Query result screenshot](images/image19.png)
![Query result screenshot](images/image34.png)
![Query result screenshot](images/image33.png)

# What the model is assuming

*The assumptions underpinning the revenue forecasting model.*

Forecast Revenue = Forecast Trend * Seasonal index, assuming: 
(1) the long-term growth trend continues, 
(2) seasonal patterns stay as they were historically, 
(3) residual noise averages out (expected residual= 1).

Example: July 2026

| Metric | Value |
|---|---|
| Forecast Trend | 563,797 |
| Seasonal Index | 0.9869 |
| Forecast Revenue | 556,393 |

563797 * 0.9869 = 556393. This indicates if the business keeps growing at its historical trend rate, also if July behaves like a typical July, revenue
should land around 556k. However, since July's seasonal factor is below 1, it's historically a touch weaker than the average month.

```sql
-- =====================================================
-- Forecast: project future months
-- =====================================================
future_months AS (
SELECT month
FROM UNNEST(GENERATE_DATE_ARRAY(DATE('2026-07-01'), DATE('2027-06-01'), INTERVAL 1 MONTH)) AS month
),
last_trend AS (
SELECT
MAX(month) AS last_month,
MAX(trend_value) AS last_trend_value,
ANY_VALUE(avg_absolute_monthly_trend_change) AS avg_monthly_growth
FROM trend_growth_summary
),
future_trend AS (
SELECT
f.month,
l.last_trend_value + (DATE_DIFF(f.month, l.last_month, MONTH) * l.avg_monthly_growth) AS forecast_trend
FROM future_months f
CROSS JOIN last_trend l
),
forecast AS (
SELECT
ft.month, ft.forecast_trend, s.seasonal_index,
ft.forecast_trend * s.seasonal_index AS forecast_revenue
FROM future_trend ft
LEFT JOIN seasonality s ON EXTRACT(MONTH FROM ft.month) = s.month_of_year
)
SELECT * FROM forecast ORDER BY month;
```

Insights:

Revenue shows a strong upward trend, a clear monthly seasonality pattern, and the model captures the structure reasonably well, with residuals mostly centred around 1.0.

```sql
-- =====================================================
-- ACTUAL vs FORECAST VALIDATION (MAE + MAPE)
-- =====================================================
-- (recreates the trend/seasonality/fitted logic above, then:)
WITH errors AS (
SELECT
month, actual_revenue, fitted_value,
actual_revenue - fitted_value AS error,
ABS(actual_revenue - fitted_value) AS abs_error,
SAFE_DIVIDE(ABS(actual_revenue - fitted_value), actual_revenue) AS abs_pct_error
FROM fitted
)
SELECT AVG(abs_error) AS MAE, AVG(abs_pct_error) * 100 AS MAPE_PERCENT
FROM errors;
```

# Forecast model evaluation

Model limitations:
1. Linear trend assumption: future trend is projected at the average historical monthly increase; growth could just as easily accelerate, decelerate, plateau or break structurally.
2. Stable seasonality assumption: assumes 2027 seasonality looks like 2020–2026, which may not hold if the product mix, pricing, market or customer behaviour shifts.
3. Moving average lag: a 3-month moving average smooths noise well but reacts slowly to sudden changes, so the model may underestimate rapid growth or miss abrupt declines.
4. No external drivers: relies purely on historical revenue, with no marketing, economic, competitor or supply-chain signals.
5. Residuals not fully investigated: the model assumes leftover variation is random noise, but it could still hide unmodelled trend, changing growth rates or additional seasonal effects.

```sql
-- =====================================================
-- Product analytics: performance matrix, repeat-driving products, basket pairs
-- =====================================================
WITH product_metrics AS (
SELECT
oi.product_id, p.name AS product_name, p.category,
COUNT(*) AS units_sold,
SUM(oi.sale_price) AS gross_revenue,
SUM(CASE WHEN o.status = 'Returned' THEN oi.sale_price ELSE 0 END) AS returned_revenue,
SAFE_DIVIDE(SUM(CASE WHEN o.status = 'Returned' THEN oi.sale_price ELSE 0 END), SUM(oi.sale_price)) AS return_rate
FROM `bigquery-public-data.thelook_ecommerce.order_items` oi
JOIN `bigquery-public-data.thelook_ecommerce.orders` o USING(order_id)
JOIN `bigquery-public-data.thelook_ecommerce.products` p ON oi.product_id = p.id
GROUP BY oi.product_id, product_name, category
)
SELECT * FROM product_metrics ORDER BY gross_revenue DESC;

-- Repeat-driving products
WITH first_product AS (
SELECT user_id, product_id,
ROW_NUMBER() OVER(PARTITION BY user_id ORDER BY created_at) AS rn
FROM `bigquery-public-data.thelook_ecommerce.order_items`
),
first_purchase_product AS (
SELECT user_id, product_id FROM first_product WHERE rn = 1
),
customer_orders AS (
SELECT user_id, COUNT(DISTINCT order_id) AS total_orders
FROM `bigquery-public-data.thelook_ecommerce.orders`
GROUP BY user_id
)
SELECT
p.name AS first_product,
COUNT(*) AS customers,
AVG(co.total_orders) AS avg_future_orders
FROM first_purchase_product fp
JOIN customer_orders co USING(user_id)
JOIN `bigquery-public-data.thelook_ecommerce.products` p ON fp.product_id = p.id
GROUP BY first_product
HAVING customers > 50
ORDER BY avg_future_orders DESC;

-- Market basket pairs
WITH order_pairs AS (
SELECT a.product_id AS product_a, b.product_id AS product_b
FROM `bigquery-public-data.thelook_ecommerce.order_items` a
JOIN `bigquery-public-data.thelook_ecommerce.order_items` b
ON a.order_id = b.order_id AND a.product_id < b.product_id
)
SELECT
p1.name AS product_a, p2.name AS product_b,
COUNT(*) AS times_bought_together
FROM order_pairs op
JOIN `bigquery-public-data.thelook_ecommerce.products` p1 ON op.product_a = p1.id
JOIN `bigquery-public-data.thelook_ecommerce.products` p2 ON op.product_b = p2.id
GROUP BY product_a, product_b
ORDER BY times_bought_together DESC
LIMIT 50;
```

 A/B testing: Conversion difference

The Treatment group converted at 8.44%, versus 8.53% for Control.
Conversion Difference = CR_Treatment − CR_Control = 0.08436 − 0.08532 = −0.00096

-> The Treatment group converted 0.096 percentage points lower than Control.

Lift

![Query result screenshot](images/image20.png)

The Treatment group performed roughly 1.13% less than Control.
![Query result screenshot](images/image27.png)

Results summary

![Query result screenshot](images/image24.png)

The difference isn't statistically significant. The Treatment group's slightly lower conversion rate is small relative to expected sampling variability. It's more likely random chance than a real effect, the null hypothesis can't be rejected here. Eventually, it doesn't support rolling out the treatment based on conversion alone.

Limitations

- Users were assigned deterministically by user ID rather than through true randomisation.
- The analysis assumes no external factors influenced conversion behaviour.
- Only conversion rate was evaluated — revenue per user, average order  value and retention weren't considered.
- This is based on historical ecommerce data and represents a simulated experiment rather than a real production A/B test.

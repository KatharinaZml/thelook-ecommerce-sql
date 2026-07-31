# TheLook Ecommerce: SQL Analytics Portfolio

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


## 4. Table of Contents

- [The Relationships](#the-relationships)
- [What each table represents](#what-each-table-represents)
- [Metrics](#metrics)
- [Why this matters](#why-this-matters)
- [RFM Customer Segmentation & Lifecycle Analytics — Project Summary](#rfm-customer-segmentation-lifecycle-analytics-----project-summary)
- [Business Questions](#business-questions)
- [Dataset Structure](#dataset-structure)
- [Analytical Approach](#analytical-approach)
- [Key Analyses Performed](#key-analyses-performed)
- [2. Purchase Frequency Analysis](#2-purchase-frequency-analysis)
- [3. Revenue Waterfall Modeling](#3-revenue-waterfall-modeling)
- [4. RFM Feature Engineering](#4-rfm-feature-engineering)
- [5. RFM Scoring](#5-rfm-scoring)
- [6. Customer Segmentation](#6-customer-segmentation)
- [Key Business Insights](#key-business-insights)
- [Technical Skills Demonstrated](#technical-skills-demonstrated)
- [4. Funnel Analysis (using events)](#4-funnel-analysis-using-events)
- [Core ecommerce funnel](#core-ecommerce-funnel)
- [Conversion Rate by Traffic Source](#conversion-rate-by-traffic-source)
- [Funnel Analysis Interpretation](#funnel-analysis-interpretation)
- [Traffic Source Analysis](#traffic-source-analysis)
- [Browser Analysis](#browser-analysis)
- [Geographic Analysis](#geographic-analysis)
- [Session Duration Analysis](#session-duration-analysis)
- [Overall Conclusion](#overall-conclusion)
- [1. What your results actually mean (clean interpretation layer)](#1-what-your-results-actually-mean-clean-interpretation-layer)
- [2. What you did well (analytical maturity signals)](#2-what-you-did-well-analytical-maturity-signals)
- [4. Next steps](#4-next-steps-very-important)
- [5. Your current level (honest assessment)](#5-your-current-level-honest-assessment)
- [6. What you should do next (clear roadmap)](#6-what-you-should-do-next-clear-roadmap)
- [6. Churn Analysis](#6-churn-analysis)
- [Useful features](#useful-features)
- [7. Product Analytics](#7-product-analytics)
- [A. Product affinity / market basket analysis](#a-product-affinity-market-basket-analysis)
- [B. Repeat-driving products](#b-repeat-driving-products)
- [What the model is assuming](#what-the-model-is-assuming)
- [Example: July 2026](#example-july-2026)
- [📊 Time Series Revenue Forecasting (Trend × Seasonality Model)](#time-series-revenue-forecasting-trend-seasonality-model)
- [Forecast Model Evaluation](#forecast-model-evaluation)
- [Business Interpretation](#business-interpretation)
- [Model Limitations](#model-limitations)
- [Conclusion](#conclusion)


# 5. The relationships

*How the five core tables in the dataset (`users`, `orders`, `order_items`, `products`, `events`) relate to one another.*

## Core schema

## 5.1. users

- One row = one customer
- It contains e.g. acquisition channel, country, age, gender or traffic source

Questions it answers:

- Where did users come from?
- Which channels bring valuable users?

## 5.2. orders

One row = one order

It includes, 
- order timestamp

- user_id

- status

- total sale amount

Questions:

- How often do users buy?

- Repeat purchase behavior?

- Revenue trends?

## order_items

One row = one product inside an order

Important:

- one order can contain multiple products

Questions:

- Basket analysis

- Product affinity

- Category revenue

## products

Product metadata:

- category

- brand

- cost

- retail price

Questions:

- Which categories drive profit?

- Which products create repeat behavior?

## events

This is VERY important later.

Contains:

- page views

- sessions

- carts

- clicks

- timestamps

This enables:

- attribution

- funnel analysis

- behavioral segmentation

- causal reasoning

SQL Queries:

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

each order contains about 1.45 items on average.

*-- That implies: 1. many orders contain 1 item, 2. some contain 2 or*
*more items, 3. large multi-item carts are probably uncommon*

![Query result screenshot](images/image9.png)

```sql
-- 1.2. understand
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

This indicates that customer purchasing behavior is dominated by quick,
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

The curve is **convex upward**, meaning:

- Growth rate is increasing over time -> Each year contributes more
    > revenue than the previous one

### 3. Strong spike in the latest months

The last few bars show a noticeable jump ( roughly 500-600k range):

- Consistent month-over-month expansion

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

Revenue growth is driven primarily by user growth, not higher spend per
user:

- total_unique_users grows massively over time (increased roughly: from
12 to 4375 -> that's a ~365x increase)

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

Monthly new users have grown steadily over time, indicating that overall
growth is strongly driven by user acquisition. Growth accelerated
strongly from late 2024 onward, with monthly new users rising sharply
and reaching over 3k by early 2026. This suggests acquisition efforts or
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

Results show the business is heavily driven by **new customer
acquisition**, with new-user revenue consistently higher than
returning-user revenue across all months. However,
returning user revenue grows steadily over time, indicating improving
customer retention and stronger repeat purchasing behaviour as the
business matures. By 2024-2026, returning customers contribute a
substantial share of total revenue. This shows a transition toward more
sustainable long-term growth.

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

1)  Example January -> Every month increases year-over-year.

2)  Oct < Nov < Dec -

Likely caused by:

- holiday shopping

- Black Friday

- Cyber Monday

- gifting season / Christmas holidays

3)  Many years show softer February performance. -> post-holiday
    > slowdow

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

- Revenue is growing every year for every month and often at very high
    > rates early on (2019-2020) due to a small base.

- From 2021-2024, growth stabilises to roughly +40% to +70% YoY,
    > meaning steady expansion.

- In 2025-2026, growth re-accelerates, with some months exceeding
    > +100% YoY, suggesting a new growth push or structural change.

What this implies:

- The business is not growing because users spend much more each year
    > (revenue per user is mostly stable).

- It is growing mainly because of more users + strong seasonality
    > (especially Q4 spikes).

- 2026 shows a possible step-change in growth

```sql
-- 2.9. Revenue=Users*Orders per User*Average Order Value (AOV) trend
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

- Assumption: Revenue = Users*Orders per User*AOV

Results show:

- Growth is mainly volume driven: User growth, followed by a late
    > increase in orders per user, while AOV stays relatively stable
    > (80-90€)

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
-- Step 4: (numerator by period) identify users active in each
lifecycle month
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

- The cohort retention analysis shows that most customers make a
    > purchase in their first month (M0 ≈ 99%), however, historically
    > very few returned in later months.

- From 2019--2024, M1 retention was generally low at around 1-3%. This
    > means the business relied heavily on new customer acquisition.

- Starting around spring 2025, retention improved with M1 retention
    > rising to 5--14%. This indicates more customers are coming back to
    > purchase again. This improvement indicates why returning customer
    > revenue is increasing, since stronger retention compounds over
    > time and raises customer lifetime value.

- Overall, the business appears to have become much better at
    > converting first-time buyers into repeat customers beginning
    > around 2025.

Results show:

- There are many low/mid-value purchases

- A few high-ticket orders contributing disproportionate revenue

Currently, results suspect: *growth = acquisition*

Therefore, retention analysis supports to check:

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
--step 2: distribution of product views (Product view quantiles by
channel)
quantiles as (
select
traffic_source,
-- prouduct_views
-- offset: zero-based indexing from an array (approx_quantiles) ; 100
-> split the distribution into 100 quantile buckets / sections
#
-- department view
approx_quantiles(department_view_count, 100) [OFFSET(25)] AS dept_p25,
approx_quantiles(department_view_count, 100) [OFFSET(50)] AS dept_p50,
--median
approx_quantiles(department_view_count, 100) [OFFSET(75)] AS dept_p75,
approx_quantiles(department_view_count, 100) [OFFSET(100)] AS
dept_p100,
-- proudct view
approx_quantiles(product_view_count, 100) [OFFSET(25)] AS product_p25,
approx_quantiles(product_view_count, 100) [OFFSET(50)] AS product_p50,
approx_quantiles(product_view_count, 100) [OFFSET(75)] AS product_p75,
approx_quantiles(product_view_count, 100) [OFFSET(100)] AS
product_p100,
-- cart view
approx_quantiles(cart_view_count, 100) [OFFSET(25)] AS cart_p25,
approx_quantiles(cart_view_count, 100) [OFFSET(50)] AS cart_p50,
approx_quantiles(cart_view_count, 100) [OFFSET(75)] AS cart_p75,
approx_quantiles(cart_view_count, 100) [OFFSET(100)] AS cart_p100,
-- purchase view
approx_quantiles(purchase_view_count, 100) [OFFSET(25)] AS
purchase_p25,
approx_quantiles(purchase_view_count, 100) [OFFSET(50)] AS
purchase_p50,
approx_quantiles(purchase_view_count, 100) [OFFSET(75)] AS
purchase_p75,
approx_quantiles(purchase_view_count, 100) [OFFSET(100)] AS
purchase_p100,
-- session duration time (in minutes)
APPROX_QUANTILES(session_duration_in_minutes, 100)[OFFSET(25)] AS
session_duration_p25,
APPROX_QUANTILES(session_duration_in_minutes, 100)[OFFSET(50)] AS
session_duration_50,
APPROX_QUANTILES(session_duration_in_minutes, 100)[OFFSET(75)] AS
session_duration_p75,
APPROX_QUANTILES(session_duration_in_minutes, 100)[OFFSET(100)] AS
session_duration_p100
from session_table
group by traffic_source
)
select *
from quantiles
order by product_p50 DESC;
purchase_p100 session_duration_p25 session_duration_50
session_duration_p75 session_duration_p100
1 Facebook 1 2 2 4 1 2 2 4 1 2 2 4 1 1 1 1 6 9 2893 5788
2 Adwords 1 2 2 4 1 2 2 4 1 2 2 4 1 1 1 1 6 9 2891 5789
3 YouTube 1 2 2 4 1 2 2 4 1 2 2 4 1 1 1 1 5 9 2891 5787
4 Email 1 2 2 4 1 2 2 4 1 2 2 4 1 1 1 1 5 9 2892 5789
5 Organic 1 2 2 4 1 2 2 4 1 2 2 4 1 1 1 1 5 8 2890 5786
```

![Query result screenshot](images/image28.png)

Across user acquisition channels (Fb, Adwords, YT, E-Mail, Organic) user
session behaviour appears highly similar:

1.  Similar browsing behaviour accross channels -> user engagement
    > depth is very similar across acquisition channels. This suggests
    > that:

- 25% of sessions involve only 1 interaction at each stage

- the typical session involves ~2 category/product/cart interactions

- 75% of sessions stay below or equal to 2 interactions

2.  Purchase behavour is nearly identical

- (p_50 -> 1) the median purchasing session contains exactly one
    > purchase event regardless of acquisition source

```{=html}
<!-- -->
```
- no channel creates significantly larger purchasing sessions

- purchase behaviour is relatively standardized

3.  Session duration insight

- most channels generate similarly engaged sessions, but organic
    > traffic sessions are slightly shorter. Possible explanation:
    > Organic users may arrive with higher intent, they may find
    > products faster or theres's less browsing before conversion.
    > However, the difference is small, so this is weak evidence rather
    > than a strong conclusion.

4.  Very large upper-trail durations

- | p75 | ~2890 minutes |

- | p100 | ~5780 minutes |- > those value range from appr. 2 tp 4
    > days. This could indicate that: sessions are not true web sessions

- purchase events occur days later

- the same session_id persists over long periods

- delayed checkout behavior exists

- **Key summary**: Acquisition channels do not strongly differentiate
    > user browsing behavior at the session level.

```sql
-- 3.3 WHich cquisition channels create the best long-term customers?
-- moving from session-level to user-level analysis
-- best long-term customers
-- here : returns repeatedly , purchases multiple times, generates high
(session perspective)
-- a. repeated sessions per user
select
user_id,
traffic_source,
COUNT(distinct session_id) as users_with_repeated_sessions
FROM `bigquery-public-data.thelook_ecommerce.events`
WHERE user_id IS NOT NULL
GROUP BY user_id, traffic_source
-- b. Purchases per user (engagement perspectiev)
select
user_id,
traffic_source,
COUNTIF(event_type = 'purchase') as purchase_counts
FROM `bigquery-public-data.thelook_ecommerce.events`
WHERE user_id IS NOT NULL
GROUP BY user_id, traffic_source
HAVING COUNTIF(event_type = 'purchase') >1
--c. Active lifespan (Customer lifespan = max(created_at) -
min(created_at)) (LTV)
select
user_id,
traffic_source,
date_diff(max(created_at), min(created_at), DAY) as user_active_lifespan
FROM `bigquery-public-data.thelook_ecommerce.events`
WHERE user_id IS NOT NULL
GROUP BY user_id, traffic_source
-- final step: Which acquisition channels create the best long-term
customers?
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

- All acquisition channels (Email, Organic, Adwords, Facebook,
    > YouTube) show almost identical long-term behaviour in sessions,
    > purchases, and lifespan.

- Average sessions per user and purchases per user are nearly the same
    > across channels (~2.22--2.27 range).

- Customer lifespan is also very similar (~196--205 days), with no
    > meaningful channel advantage.

- Email brings the highest volume of users, while other channels
    > contribute smaller but comparable-quality cohorts.

- Overall, channel type is not a strong differentiator of long-term
    > customer value in this dataset; behaviour is consistent across
    > sources.

**2. Customer Lifetime Value (LTV)**

"How much revenue does an average customer generate over their
lifetime?" LTV by acquisition source

Question:

Which channels bring valuable users instead of just many users?

```sql
-- 3.4. Cohort retention by channel:
-- Do users acquired from different channels come back and stay active
over time,
-- and how does that differ by channel?
-- a. first user cohort interaction
WITH user_cohort AS (
SELECT
user_id,
MIN(DATE(created_at)) AS first_purchase_date,
-- cohort month
DATE_TRUNC(MIN(DATE(created_at)), MONTH) AS first_cohort_month,
-- acquisition channel (approximation)
ANY_VALUE(traffic_source) AS traffic_source
FROM `bigquery-public-data.thelook_ecommerce.events`
WHERE user_id IS NOT NULL
GROUP BY user_id
),
-- b. user activity tracking
user_activity_tracking AS (
SELECT
e.user_id,
uc.traffic_source,
uc.first_cohort_month,
-- activity month
DATE_TRUNC(DATE(e.created_at), MONTH) AS activity_month,
-- months since cohort month
DATE_DIFF(
DATE_TRUNC(DATE(e.created_at), MONTH),
uc.first_cohort_month,
MONTH
) AS month_index
FROM `bigquery-public-data.thelook_ecommerce.events` e
JOIN user_cohort uc
USING (user_id)
),
-- c. cohort activity counts (FIXED MISSING STEP)
cohort_counts AS (
SELECT
traffic_source,
first_cohort_month AS cohort_month,
month_index,
COUNT(DISTINCT user_id) AS active_users
FROM user_activity_tracking
GROUP BY 1, 2, 3
),
-- d. cohort sizes
cohort_sizes AS (
SELECT
traffic_source,
first_cohort_month AS cohort_month,
COUNT(DISTINCT user_id) AS cohort_size
FROM user_cohort
GROUP BY 1, 2
)
-- e. user retention table
SELECT
c.traffic_source,
c.month_index,
SUM(c.active_users) AS active_users,
MAX(s.cohort_size) AS cohort_size,
-- Retention = Active Users in Month N / Original Cohort Size
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
proved:
```

## Business growth dynamics: Summary

- growth is real

- acquisition-driven

- not primarily price-driven

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

This is classic ecommerce/customer intelligence.

RFM:

- Recency

- Frequency

- Monetary

## You classify users into groups:

  -----------------------------------------------------------------------
  **Segment**              **Meaning**
*------------------------ ----------------------------------------------*
*Champions                buy often + recently + high spend*
*Loyal                    repeat customers*
*At Risk                  used to buy but disappeared*
*New Customers            recently acquired*
*Big Spenders             high monetary value*
*-----------------------------------------------------------------------*

# Metrics

*Definitions of the core RFM metrics — Recency, Frequency and Monetary value.*

## Recency

Days since last order

The key formulas:

Recency Days=Current Date−Last Purchase Date

Customer Lifespan=Last Purchase Date−First Purchase Date

## Frequency

Total orders

## Monetary

Total spend

# Why this matters

*Why RFM metrics are useful for CRM targeting, retention campaigns and personalisation.*

This directly supports:

- CRM targeting

- retention campaigns

- personalisation

- churn prevention

This is one of the highest-value analyses businesses actually use.

```sql
-- =========================================================
-- RFM PIPELINE --
-- =========================================================
-- +++++++++++++++++++++++++++++++++++++++++++++++++++++++++
WITH
-- =========================================================
-- 0) -- Data validation
-- =========================================================
-- a. Do all orders have valid item-level data? -> (validate order ↔
order_items relationship)
validation AS (
SELECT
COUNT(DISTINCT o.order_id) AS orders_table_orders,
COUNT(DISTINCT oi.order_id) AS order_items_orders
FROM `bigquery-public-data.thelook_ecommerce.orders` o
LEFT JOIN `bigquery-public-data.thelook_ecommerce.order_items` oi
USING(order_id)
),
```

![Query result screenshot](images/image13.png)

*-- d4. What is the typical time between customer purchases?*
*-- SELECT*
*-- APPROX_QUANTILES(day_since_previous_purchase, 100)[OFFSET(25)]*
*-- AS p25_day_since_previous_purchase,*
*--*
*-- APPROX_QUANTILES(day_since_previous_purchase, 100)[OFFSET(50)]*
*-- AS p50_day_since_previous_purchase,*
*--*
*-- APPROX_QUANTILES(day_since_previous_purchase, 100)[OFFSET(75)]*
*-- AS p75_day_since_previous_purchase,*
*--*
*-- APPROX_QUANTILES(day_since_previous_purchase, 100)[OFFSET(100)]*
*-- AS p100_day_since_previous_purchase*
*--*
*-- FROM purchase_gap;*

![Query result screenshot](images/image2.png)

The distribution appears right-skewed (positively skewed). This means
most customers reorder relatively sooner, but a smaller number take long
to reorder, pulling the upper tail outward. Half of repeat purchases
happen within 217 days (roughly 7 months). While 25% of repeat purchases
occur within 67 days (about 2 months), the upper tail extends
substantially, with some customers taking several years to reorder. This
suggests heterogeneous customer engagement patterns.

```sql
-- =========================================================
-- RFM PIPELINE --
-- =========================================================
-- +++++++++++++++++++++++++++++++++++++++++++++++++++++++++
WITH
-- =========================================================
-- 0) -- Data validation
-- =========================================================
-- a. Do all orders have valid item-level data? -> (validate order ↔
order_items relationship)
validation AS (
SELECT
COUNT(DISTINCT o.order_id) AS orders_table_orders,
COUNT(DISTINCT oi.order_id) AS order_items_orders
FROM `bigquery-public-data.thelook_ecommerce.orders` o
LEFT JOIN `bigquery-public-data.thelook_ecommerce.order_items` oi
USING(order_id)
),
-- =========================================================
-- 1) Foundation: Base order table
-- =========================================================
-- b. What are all customer purchase events over time? (1 row = 1
order)
orders_placed AS (
SELECT
order_id,
user_id,
DATE(created_at) AS order_date
FROM `bigquery-public-data.thelook_ecommerce.orders`
),
-- =========================================================
-- 2) Customer lifecycle table (Core RFM)
-- =========================================================
-- c. How engaged is each customer over their lifecycle?
-- Incl: first purchase date, last purchase date, recency , lifespan,
total_orders
rfm_base AS (
SELECT
user_id,
-- c1. total purchases made
COUNT(DISTINCT order_id) AS total_orders,
-- c2. first purchase date
MIN(order_date) AS first_purchase_date,
-- c3. last purchase date
MAX(order_date) AS last_purchase_date,
-- c4. Recency
DATE_DIFF(
CURRENT_DATE(),
MAX(order_date),
DAY
) AS recency_days,
-- c5. customer lifespan
DATE_DIFF(
MAX(order_date),
MIN(order_date),
DAY
) AS customer_lifespan_days
FROM orders_placed
GROUP BY user_id
),
-- =========================================================
-- 3) Frequency analysis (CUSTOMER LEVEL FIXED)
-- =========================================================
-- d. How often do customers reorder and what is the gap between
purchases?
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
LAG(order_date) OVER(
PARTITION BY user_id
ORDER BY order_date
) AS previous_order_date,
DATE_DIFF(
order_date,
LAG(order_date) OVER(
PARTITION BY user_id
ORDER BY order_date
),
DAY
) AS day_since_previous_purchase
FROM orders_placed
)
GROUP BY user_id
),
-- =========================================================
-- 4) Monetary value
-- =========================================================
-- e. How much revenue does each customer generate?
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
-- f. Revenue waterfall model
revenue_waterfall_model AS (
SELECT
user_id,
SUM(sale_price) AS total_gross_revenue,
SUM(CASE
WHEN status NOT IN ('Cancelled', 'Returned')
THEN sale_price ELSE 0 END
) AS net_revenue,
SUM(CASE
WHEN status = 'Cancelled'
THEN sale_price ELSE 0 END
) AS cancelled_revenue,
SUM(CASE
WHEN status = 'Returned'
THEN sale_price ELSE 0 END
) AS returned_revenue
FROM monetary_table
GROUP BY user_id
),
-- =========================================================
-- 5) Final RFM table (CUSTOMER LEVEL)
-- =========================================================
-- How valuable, recent, and frequent is each customer?
final_rfm_table AS (
SELECT
-- g. RFM Base data
rfm.user_id,
rfm.recency_days,
rfm.customer_lifespan_days,
rfm.total_orders,
-- h. Purchase gap data
pg.avg_days_between_orders,
pg.purchase_frequency_per_month,
-- i. Monetary data
rwm.total_gross_revenue,
rwm.net_revenue,
rwm.cancelled_revenue,
rwm.returned_revenue,
SAFE_DIVIDE(
rwm.returned_revenue,
rwm.total_gross_revenue
) AS return_rate
FROM rfm_base rfm
LEFT JOIN purchase_gap pg USING(user_id)
LEFT JOIN revenue_waterfall_model rwm USING(user_id)
),
-- =========================================================
-- 6) RFM Scoring
-- =========================================================
-- Which customers are best vs worst?
rfm_scoring AS (
SELECT
*,
-- Recency (lower is better → reversed)
NTILE(5) OVER (ORDER BY recency_days DESC) AS recency_score,
-- Frequency
NTILE(5) OVER (ORDER BY purchase_frequency_per_month) AS
frequency_score,
-- Monetary
NTILE(5) OVER (ORDER BY net_revenue) AS monetary_score
FROM final_rfm_table
),
-- =========================================================
-- 7) Customer segmentation
-- =========================================================
-- k. What type of customers do we have?
customer_segments AS (
SELECT
*,
CASE
-- k.1 VIP customers
WHEN recency_score >= 4
AND frequency_score >= 4
AND monetary_score >= 4
THEN 'VIP'
-- k.2 Loyal customers
WHEN recency_score >= 3
AND frequency_score >= 4
THEN 'Loyal'
-- k.3 New customers
WHEN recency_score >= 4
AND total_orders <= 2
THEN 'New Customers'
-- k.4 At risk customers
WHEN recency_score <= 2
AND (frequency_score >= 3 OR monetary_score >= 3)
THEN 'At Risk'
-- k.5 Churn risk customers
WHEN recency_score <= 2
THEN 'Churn Risk'
-- k.6 High return customers
WHEN return_rate >= 0.30
THEN 'High Return Customers'
-- k.7 One-time buyers
WHEN total_orders = 1
THEN 'One-time Buyers'
-- k.8 Regular customers
ELSE 'Regular'
END AS customer_segment
FROM rfm_scoring
)
-- =========================================================
-- FINAL OUTPUT
-- =========================================================
SELECT *
FROM customer_segments;
```

![Query result screenshot](images/image8.png)

```sql
-- =========================================================
-- 8) Validation
-- =========================================================
-- l. Duplicate customer rows in final table
SELECT
COUNT(*) AS total_rows,
COUNT(DISTINCT user_id) AS unique_users
FROM final_rfm_table;
```

![Query result screenshot](images/image17.png)

79745 total rows = 79745 unique users. There is no duplication caused by
joins, meaning the grain control is solid.

```sql
-- l2. Negative recency or lifespan
SELECT *
FROM final_rfm_table
WHERE recency_days < 0
OR customer_lifespan_days < 0;
```

![Query result screenshot](images/image22.png)

-> No data returned. There are no invalid date calculations (e.g.,
future dates or reversed lifespans).

```sql
-- l3. Revenue consistency check
SELECT *
FROM final_rfm_table
WHERE ABS(
total_gross_revenue
- (
net_revenue
\+ cancelled_revenue
\+ returned_revenue
)
) > 0.01;
Gross revenue correctly equals the sum of net, cancelled, and returned
revenue without leakage or double counting. -> The defined financial
```

waterfall model is internally consistent.

![Query result screenshot](images/image22.png)

```sql
-- l4. Return rate boundaries
SELECT *
FROM final_rfm_table
WHERE return_rate < 0
OR return_rate > 1;
```

![Query result screenshot](images/image7.png)
Dataset doesn't show anomalies in key
steps. This means joins, CASE logic, and aggregations are behaving as
expected.

```sql
-- l5. Customers with orders but zero revenue
SELECT *
FROM final_rfm_table
WHERE total_orders > 0
AND total_gross_revenue = 0;
```

![Query result screenshot](images/image7.png)

```sql
-- l6. Null critical fields
SELECT *
FROM final_rfm_table
WHERE recency_days IS NULL
OR total_orders IS NULL
```

![Query result screenshot](images/image7.png)

```sql
-- l7. Purchase frequency anomalies
SELECT *
FROM final_rfm_table
WHERE avg_purchase_frequency_per_month <
```

![Query result screenshot](images/image7.png)

```sql
-- l8. Segment distribution sanity check
SELECT
customer_segment,
COUNT(*) AS customers
FROM customer_segments
GROUP BY customer_segment
ORDER BY customers DESC;
```

![Query result screenshot](images/image31.png)

Validation results show that segmentation is informative. However, the
large "At Risk + Churn Risk" share suggests retention is the main
business challenge.

```sql
-- l9. Score distribution check
SELECT
recency_score,
COUNT(*) AS customers
FROM rfm_scoring
GROUP BY recency_score
ORDER BY recency_score;
```

![Query result screenshot](images/image11.png)

The scoring is balanced. NTILE(5) enforced equal-sized buckets.

This confirms correct implementation of quantile-based
normalisation.This means every score bucket is forced to be equal-sized

```sql
-- l10. One-time buyers validation
SELECT *
FROM customer_segments
WHERE customer_segment = 'One-time Buyers'
```

![Query result screenshot](images/image7.png)

No violation of the rule reported.

**Summary**: The RFM pipeline seems structurally sound and passes core
data integrity checks.

# RFM Customer Segmentation & Lifecycle Analytics — Project Summary

*Overview of the RFM segmentation project: aims, scope and approach.*

## Overview

This project analyzes customer lifecycle behavior using the TheLook
Ecommerce Dataset from [Google BigQuery Public
Datasets](https://cloud.google.com/bigquery/public-data?utm_source=chatgpt.com).

The analysis moves beyond descriptive KPI reporting toward:

- behavioral analytics

- lifecycle modeling

- customer segmentation

- retention thinking

- revenue quality analysis

The goal was to build a full customer-level RFM (Recency, Frequency,
Monetary) framework using SQL while applying analytical engineering
concepts such as:

- decomposition

- granularity control

- feature engineering

- cohort/lifecycle reasoning

# Business Questions

*The core business questions the RFM analysis sets out to answer.*

The project aimed to answer:

- Which customers are most valuable?

- Which customers are likely to churn?

- Is business growth driven by acquisition or retention?

- How frequently do customers repurchase?

- How much revenue is lost through returns and cancellations?

- Which customer segments drive long-term value?

# Dataset Structure

*The subset of tables and fields used for the RFM analysis.*

Core tables used:

  -----------------------------------------------------------------------
  **Table**           **Description**
*------------------- ---------------------------------------------------*
*users               customer information*
*orders              one row per order*
*order_items         one row per product within an order*

  events              customer browsing/session behavior
*-----------------------------------------------------------------------*
*Core relationship:*
*users*
*↓*
*orders*
*↓*
*order_items*

# Analytical Approach

*The step-by-step method used to build the RFM segmentation.*

The project was structured as a multi-stage SQL pipeline:

Raw transactional data

↓

Customer lifecycle modeling

↓

Frequency analysis

↓

Revenue decomposition

↓

RFM feature engineering

↓

Customer scoring

↓

Behavioral segmentation

# Key Analyses Performed

*Summary of the analytical steps carried out, from purchase frequency to customer segmentation.*

## 1. Customer Lifecycle Analysis

Built customer-level lifecycle metrics including:

- first purchase date

- last purchase date

- customer lifespan

- recency

- total orders

Key finding:

- business growth is primarily acquisition-driven

- returning customer revenue increases over time

- retention improves as the business matures

# 2. Purchase Frequency Analysis

*How often customers purchase, used as an input to the Frequency score.*

Analyzed:

- time between purchases

- repeat purchase cadence

- purchase frequency per month

Key finding:

- most customers purchase infrequently

- repeat purchasing behavior varies substantially across users

- customer behavior is highly heterogeneous

# 3. Revenue Waterfall Modeling

*How revenue is built up and attributed across customers and orders.*

Constructed a financial decomposition model:

Gross Revenue=Net Revenue+Cancelled Revenue+Returned RevenueGross\
Revenue = Net\ Revenue + Cancelled\ Revenue + Returned\ RevenueGross
Revenue=Net Revenue+Cancelled Revenue+Returned Revenue

Metrics included:

- gross revenue

- net revenue

- cancelled revenue

- returned revenue

- return rate

Key finding:

- a small subset of customers generates disproportionate return
    > behavior

- revenue quality differs significantly across customers

# 4. RFM Feature Engineering

*Deriving the Recency, Frequency and Monetary features from raw order data in SQL.*

Built customer-level behavioral features:

- Recency

- Frequency

- Monetary value

- Customer lifespan

- Purchase frequency

- Return impact

The final analytical table was structured at:

1 row = 1 customer

This ensured:

- no duplication

- stable analytical granularity

- scalable feature engineering

# 5. RFM Scoring

*Scoring each customer on Recency, Frequency and Monetary value.*

Customers were ranked using quintile-based scoring:

- recency score

- frequency score

- monetary score

This normalized customer behavior into comparable behavioral groups.

# 6. Customer Segmentation

*Grouping customers into lifecycle segments (e.g. Champions, At Risk, Lost) based on RFM scores.*

Customers were classified into:

- VIP

- Loyal

- New Customers

- At Risk

- Churn Risk

- One-time Buyers

- High Return Customers

- Regular Customers

Key insights:

- a large "At Risk" segment suggests retention deterioration over time

- acquisition remains strong through a large inflow of new customers

- VIP customers represent a strategically important high-value cohort

- many customers fail to transition from first purchase to repeat
    > purchasing behavior

# Key Business Insights

*The headline findings from the RFM segmentation.*

## Growth Dynamics

Revenue growth is primarily driven by:

- increasing user acquisition

- rising order volume

Average spend per customer remains relatively stable.

## Retention & Churn

The business demonstrates:

- improving repeat customer revenue

- but substantial long-term churn risk

This suggests:

- acquisition is strong

- lifecycle retention remains a major optimization opportunity

## Customer Value Concentration

Customer value is highly uneven:

- a relatively small subset of customers generates disproportionate
    > business value

- high-frequency and high-monetary customers are strategically
    > critical

## Revenue Quality

Not all revenue contributes equally:

- cancelled and returned orders materially affect realized customer
    > value

- high-return customers may reduce operational profitability

# Technical Skills Demonstrated

*The SQL and analytical techniques used throughout the RFM work.*

## SQL Techniques

- CTE pipelines

- window functions

- cohort analysis

- feature engineering

- quantile scoring

- lifecycle analysis

- behavioral aggregation

- revenue decomposition

## Analytical Concepts

- granularity control

- customer lifecycle modeling

- behavioral segmentation

- RFM analysis

- retention analytics

- decomposition logic

- customer feature engineering

# 4. Funnel Analysis (using events)

*Introduction to the purchase funnel analysis built from the `events` table.*

Now you enter behavioral analytics.

Your events table unlocks:

- browsing behavior

- session analysis

- drop-offs

- conversion funnel

# Core ecommerce funnel

*Definition of the product → cart → purchase funnel used throughout this analysis.*

Session

→ Product View

→ Add to Cart

→ Checkout

→ Purchase

You want conversion rates between each step.

# Conversion Rate by Traffic Source

*Building the session- and user-level funnel, then breaking conversion down by traffic source, browser, city and session duration.*

You already analyzed revenue by acquisition.

Now combine it with behavior.

Question:

> Which channels convert best?

Example:

Conversion Rate=Total VisitorsPurchasing Users​

Segment by:

- traffic source

- country

- device

- gender

- campaign

**E-Commerce funnel**

```sql
-- =========================================================
-- Funnel Analysis
-- =========================================================
-- Research question: What behaviours lead to conversion?
-- =========================================================
-- =========================================================
-- 0) Exploration: Understand event structure
-- =========================================================
--
--
-- a. What does one row represent?
--
--
-- Observation: One row represents one ecommerce event within a
session.
SELECT
*
FROM `bigquery-public-data.thelook_ecommerce.events`
WHERE user_id = 77064 -- Example user
ORDER BY id, sequence_number ASC;
--
--
-- b. Understanding the behavioural flow
--
--
-- Goal: Explore how users move through the ecommerce funnel
SELECT
*
FROM `bigquery-public-data.thelook_ecommerce.events`
WHERE user_id = 77064 -- Example user
ORDER BY id, sequence_number ASC;
-- =========================================================
-- 1) Event distribution analysis
-- =========================================================
--
--
-- a. What event types exist?
--
--
SELECT
event_type,
COUNT(*) AS events
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY event_type
ORDER BY events DESC;
--
--
-- b. Event types as percentage of total events
--
--
SELECT
event_type,
COUNT(*) AS events,
ROUND(
COUNT(*) * 100.0 /
SUM(COUNT(*)) OVER (),
2
) AS percentage
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY event_type
ORDER BY events DESC;
-- =========================================================
-- Key Behavioural Observations
-- =========================================================
-- The majority of user activity occurs at the product-viewing and
cart stages. While many users browse products and add items to their
carts, only a smaller proportion proceed to complete a purchase.. This
```

suggests significant user drop-off between the cart and checkout stages,
which may indicate: friction in the purchasing process, checkout
abandonment, price sensitivity or a low purchase intent. However, this
dataset behaves differently from real-world ecommerce data.

-- Observed dataset limitations: Many users appear to have product,
cart, and purchase events. Funnel behaviour appears artificially smooth.
Home page visits are relatively low,suggesting some sessions begin
directly on, product or department pages. Therefore, instead of
analysing: \"How many users reached each funnel step?\", this analysis
focuses on: \"How many events occurred at each funnel step?\"
Conclusion: The TheLook Ecommerce dataset appears to be at least
partially synthetically generated and is likely designed primarily for
SQL practice,rather than perfectly realistic behavioural
analysis.

```sql
-- =========================================================
-- =========================================================
-- 2) Conversion Metrics
-- =========================================================
--
--
-- a. Cart → Purchase Conversion Rate
--
--
-- Formula: purchases / cart events * 100
SELECT
ROUND(
SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END)
* 100.0
/
SUM(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END),
2
) AS cart_to_purchase_rate
FROM `bigquery-public-data.thelook_ecommerce.events`;
-- Result: 30.51%
--
--
-- b. Purchase Cancellation Rate
--
--
-- Formula:
-- cancellations / purchases * 100
SELECT
ROUND(
SUM(CASE WHEN event_type = 'cancel' THEN 1 ELSE 0 END)
* 100.0
/
SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END),
2
) AS purchase_cancellation_rate
FROM `bigquery-public-data.thelook_ecommerce.events`;
```

![Query result screenshot](images/image23.png)
The majority of user activity occurs at
the product-viewing and cart stages. While many users browse products
and add items to their carts, only a smaller proportion proceed to
complete a purchase.. This suggests significant user drop-off between
the cart and checkout stages, which may indicate: friction in the
purchasing process, checkout abandonment, price sensitivity or a low
purchase intent. However, this dataset behaves differently from
real-world ecommerce data.

Observed dataset limitations: Many users appear to have product, cart,
and purchase events. Funnel behaviour appears artificially smooth. Home
page visits are relatively low,suggesting some sessions begin directly
on, product or department pages. Therefore, instead of analysing: \"How
many users reached each funnel step?\", this analysis focuses on: \"How
many events occurred at each funnel step?\" Conclusion: The TheLook
Ecommerce dataset appears to be at least partially synthetically
generated and is likely designed primarily for SQL practice,rather than
perfectly realistic behavioural analysis.

```sql
-- Observation:
-- Nearly 69% of purchases were canceled, which is unusually high and
worth further investigation.
-- =========================================================
-- 3) Funnel Drop-off Analysis
-- =========================================================
-- Funnel:
-- product → cart → purchase
--
-- Reason: Users may skip home or department pages.
--
-- Drop-off formula: (previous_stage - next_stage) / previous_stage *
100
-- =========================================================
WITH funnel AS (
SELECT
SUM(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END)
AS product_events,
SUM(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END)
AS cart_events,
SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END)
AS purchase_events
FROM `bigquery-public-data.thelook_ecommerce.events`
)
SELECT
product_events,
cart_events,
-- Product → Cart Drop-off %
ROUND(
(product_events - cart_events)
* 100.0 / product_events,
2
) AS product_to_cart_dropoff_percentage,
purchase_events,
-- Cart → Purchase Drop-off %
ROUND(
(cart_events - purchase_events)
* 100.0 / cart_events,
2
) AS cart_to_purchase_dropoff_percentage
FROM funnel;
```

![Query result screenshot](images/image4.png)

This funnel analysis shows substantial drop-off as users move from
browsing products to completing purchases. Roughly 30% of product
interactions did not progress to the cart stage. At the same time, about
70% progression rate from product to cart is actually fairly strong
engagement.

Next, appr. 70% of cart interactions did not result in a completed
purchase. Specifically, about 30% of cart interactions completed a
purchase.

Limitation: One user can generate multiple events. Therefore,
session-level and user-level funnels are using: user_id & session_id to
understand actual customer journeys and conversion behaviour across
sessions.

```sql
-- =========================================================
-- 4) User-level Funnel Analysis
-- =========================================================
-- a. Base: Session Funnel table
WITH session_funnel AS(
SELECT
session_id,
user_id,
-- Highest event level that was reached
MAX(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END) as
viewed_product, -- This session reached the cart stage.
MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) as
viewed_cart,
MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) as
viewed_purchase
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY
session_id,
user_id
)
-- b. How many sessions reached each stage?
SELECT
-- total sessions
count(*) as total_sessions,
-- Product → Cart Drop-off %
-- Drop-off formula: (previous_stage - next_stage) / previous_stage *
100
(SUM(viewed_product) - SUM(viewed_cart)) * 100.0 / SUM(viewed_product)
as product_to_cart_dropoff_percentage,
-- xCart → Purchase Drop-off %
-- Drop-off formula: (previous_stage - next_stage) / previous_stage *
100
(SUM(viewed_cart) - SUM(viewed_purchase)) * 100.0 / SUM(viewed_cart)
as cart_to_purchase_dropoff_percentage,
FROM session_funnel;
```

![Query result screenshot](images/image1.png)

About 36.7% of sessions that viewed a product did not add anything to
the cart, meaning that approximately 63.3% of product-view sessions
progressed to the cart stage. From the sessions that reached the cart,
around 57.97% did not proceed to purchase, meaning that only about 42.0%
of cart sessions resulted in a completed purchase.

```sql
-- =========================================================
-- 5) Foundation: Core Funnel Table
-- =========================================================
-- Goal: Create a session-level funnel table where: 1 row = 1
session
WITH full_session_funnel as (
SELECT
session_id,
user_id, -- return 1 row per user_id
-- a. engagement intensity
COUNT(*) as total_events,
-- b. engagement depth (measured in
time)session_duration_minutes
TIMESTAMP_DIFF(
MAX(created_at), MIN(created_at), MINUTE) as
session_duration_minutes,
-- c. browsing breadth
MAX(CASE WHEN event_type = 'department' THEN 1 ELSE 0 END)
AS viewed_department,
-- d. shopping intent
MAX(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END)
AS viewed_product,
-- e. purchase intent
MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END)
AS viewed_cart,
-- f. conversion outcome
MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END)
AS purchased,
-- g. pick one arbitrary value from that user's rows (checked for
consistency)
ANY_VALUE(traffic_source) as traffic_source,
ANY_VALUE(browser) as browser,
ANY_VALUE(city) as city,
ANY_VALUE(postal_code) as postal_code,
ANY_VALUE(browser) as browser
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY
session_id,
user_id
)
-- select * from full_session_funnel;
This table represents 1 row = 1 session. From raw events, product, cart
and purchase events will be repeatedly re-calculated. To have all data
in one clean table, the CTE with behavioural features is established
with session quality metrics.
Examples:
--
```

  **Feature**                   **Meaning**
*-------------------------------------- --------------------------------*
*total_events                  engagement intensity*
*session_duration_minutes      engagement depth*
*viewed_departments            browsing breadth*
*viewed_products               shopping intent*
*viewed_cart                   purchase intent*
*purchased                     conversion outcome*
*-----------------------------------------------------------------------*

![Query result screenshot](images/image3.png)

```sql
-- =========================================================
-- 6) Segment Conversion: Funnel by Traffic Source
-- =========================================================
-- Which acquisition channels convert best? ; 1 row = 1 traffic
source
-- =========================================================
SELECT
traffic_source,
COUNT(*) AS total_sessions,
-- Product rate:
-- % of sessions reaching product stage
ROUND(
(SUM(viewed_product) / COUNT(*)) * 100,
2
) AS product_rate_percentage,
-- Cart rate:
-- % of sessions reaching cart stage
ROUND(
(SUM(viewed_cart) / COUNT(*)) * 100,
2
) AS cart_rate_percentage,
-- Purchase rate:
-- % of sessions converting to purchase
ROUND(
(SUM(purchased) / COUNT(*)) * 100,
2
) AS purchase_rate_percentage,
-- Product → Cart Drop-off %
ROUND(
(SUM(viewed_product) - SUM(viewed_cart))
* 100.0 / SUM(viewed_product),
2
) AS product_to_cart_dropoff_percentage,
-- Cart → Purchase Drop-off %
ROUND(
(SUM(viewed_cart) - SUM(purchased))
* 100.0 / SUM(viewed_cart),
2
) AS cart_to_purchase_dropoff_percentage
FROM full_session_funnel
GROUP BY traffic_source
ORDER BY purchase_rate_percentage DESC;
```

![Query result screenshot](images/image12.png)

Key finding:

- Traffic sources behave almost identifcally across the entire funnel
    > channel, meaning that all five channels generate similar
    > behaviour. Conversion metrics differ marginally across channels,
    > suggesting that traffic source has little impact on user
    > conversion behaviour in this dataset.

- Next, regardless of acquisition source users reach product pages
    > and add items to the cart. This indicates that the conversion
    > bottleneck is likely within the checkout experience itself, rather
    > than the acquisition channel.

- Organic traffic slightly underperforms at purchase stage while the
    > highest cart rate is about 63.55% , but the lowest purchase rate
    > is 26.38%. However, the differences are extremely small. The
    > unusually similar performance across channels also supports the
    > assumption that the dataset is partially synthetic and designed
    > primarily for SQL practice rather than realistic behavioural
    > modelling.

```sql
WITH session_funnel AS (
SELECT
session_id,
user_id,
-- =========================
-- Session behaviour metrics
-- =========================
COUNT(*) AS total_events,
-- =========================
-- Session duration (measured in minutes)
-- =========================
TIMESTAMP_DIFF(
MAX(created_at),
MIN(created_at),
MINUTE
) AS session_duration_minutes,
-- =========================
-- Funnel stages
-- =========================
MAX(CASE WHEN event_type = 'department' THEN 1 ELSE 0 END)
AS viewed_department,
MAX(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END)
AS viewed_product,
MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END)
AS viewed_cart,
MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END)
AS purchased,
-- =========================
-- Session attributes
-- =========================
ANY_VALUE(traffic_source) AS traffic_source,
ANY_VALUE(browser) AS browser,
ANY_VALUE(city) AS city
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY session_id, user_id
),
-- =========================================================
-- 1) Traffic Source Segment
-- =========================================================
traffic_source_funnel AS (
SELECT
traffic_source AS segment,
COUNT(*) AS total_sessions,
-- =========================
-- Session rates
-- =========================
ROUND(SUM(viewed_product) * 100.0 / COUNT(*), 2) AS product_rate, --
% of sessions reaching product stage
ROUND(SUM(viewed_cart) * 100.0 / COUNT(*), 2) AS cart_rate, -- % of
sessions reaching cart stage
ROUND(SUM(purchased) * 100.0 / COUNT(*), 2) AS purchase_rate, -- %
of sessions converting to purchase
-- =========================
-- Drop-off
-- =========================
ROUND(
(SUM(viewed_product) - SUM(viewed_cart)) * 100.0 /
SUM(viewed_product),
2
) AS product_to_cart_dropoff,
ROUND(
(SUM(viewed_cart) - SUM(purchased)) * 100.0 /
SUM(viewed_cart),
2
) AS cart_to_purchase_dropoff
FROM session_funnel
GROUP BY traffic_source
),
-- =========================================================
-- 2) Browser segment
-- =========================================================
browser_funnel AS (
SELECT
browser AS segment,
COUNT(*) AS total_sessions,
ROUND(SUM(purchased) * 100.0 / COUNT(*), 2) AS purchase_rate
FROM session_funnel
GROUP BY browser
),
-- =========================================================
-- 3) City segment
-- =========================================================
city_funnel AS (
SELECT
city AS segment,
COUNT(*) AS total_sessions,
ROUND(SUM(purchased) * 100.0 / COUNT(*), 2) AS purchase_rate
FROM session_funnel
GROUP BY city
HAVING COUNT(*) > 1000
),
-- =========================================================
-- 4) Session duration segment
-- =========================================================
duration_funnel AS (
SELECT
CASE
WHEN session_duration_minutes = 0 THEN '0 min'
WHEN session_duration_minutes BETWEEN 1 AND 5 THEN '1-5 min'
WHEN session_duration_minutes BETWEEN 6 AND 15 THEN '6-15
min'
WHEN session_duration_minutes BETWEEN 16 AND 30 THEN '16-30
min'
ELSE '30+ min'
END AS segment,
COUNT(*) AS total_sessions,
ROUND(AVG(session_duration_minutes), 2) AS avg_duration,
ROUND(SUM(purchased) * 100.0 / COUNT(*), 2) AS purchase_rate
FROM session_funnel
GROUP BY segment
)
-- =========================================================
-- FINAL OUTPUT (stacked)
-- =========================================================
SELECT
'traffic_source' AS dimension,
segment,
total_sessions,
purchase_rate
FROM traffic_source_funnel
UNION ALL
SELECT
'browser' AS dimension,
segment,
total_sessions,
purchase_rate
FROM browser_funnel
UNION ALL -- stack query results on top of each other
SELECT
'city' AS dimension,
segment,
total_sessions,
purchase_rate
FROM city_funnel
UNION ALL
SELECT
'session_duration' AS dimension,
segment,
total_sessions,
purchase_rate
FROM duration_funnel
;
```

Your SQL pipeline is now technically strong and analytically coherent.
The next step is no longer SQL correction — it is turning the results
into a clean analytical narrative.

Here is a polished interpretation section you could place after your
query results.

# Funnel Analysis Interpretation

*Interpreting the funnel results across all segmentation dimensions.*

## Overview

A session-level funnel table was constructed to analyse how users
progress through the ecommerce journey from product browsing to purchase
completion. Each row in the funnel table represents one session and
captures behavioural activity, engagement depth, conversion outcomes,
and user attributes such as traffic source, browser, and city.

The analysis segmented conversion behaviour across:

- traffic sources

- browsers

- cities

- session duration groups

# Traffic Source Analysis

*Conversion rate by traffic source.*

Purchase conversion rates were highly consistent across all traffic
acquisition channels.

  -----------------------------------------------------------------------
  **Traffic Source**                  **Purchase Rate**
*----------------------------------- -----------------------------------*
*Facebook                            26.63%*
*YouTube                             26.53%*
*Adwords                             26.53%*
*Email                               26.52%*
*Organic                             26.38%*
*-----------------------------------------------------------------------*

The small variation across channels suggests that acquisition source has
minimal influence on conversion behaviour within this dataset. In real
ecommerce environments, larger differences are usually expected between
paid, organic, and email traffic due to differences in purchase intent
and targeting quality.

The unusually uniform conversion patterns suggest that the dataset may
contain synthetic or simplified behavioural generation.

# Browser Analysis

*Conversion rate by browser.*

Conversion rates were also nearly identical across browsers.

  -----------------------------------------------------------------------
  **Browser**                 **Purchase Rate**
*--------------------------- -------------------------------------------*
*Safari                      26.66%*
*Chrome                      26.52%*
*Firefox                     26.47%*
*IE                          26.51%*
*-----------------------------------------------------------------------*
*Typically, browser-level analysis can reveal technical friction,*
*checkout issues, or device optimization problems. However, the*
*near-identical conversion rates observed here indicate no meaningful*

browser-related performance differences.

# Geographic Analysis

*Conversion rate by city.*

City-level segmentation revealed greater variation in conversion
behaviour than traffic source or browser segmentation.

Examples include:

  -----------------------------------------------------------------------
  **City**                         **Purchase Rate**
*-------------------------------- --------------------------------------*
*Maoming                          32.17%*
*San Antonio                      30.22%*
*Jinan                            30.04%*
*Qingdao                          23.02%*
*Wuhan                            24.28%*
*-----------------------------------------------------------------------*

This suggests that geographic location may have a stronger relationship
```sql
with purchasing behaviour than acquisition channel within this dataset.
However, some cities contain relatively small sample sizes, so
```
conclusions should be interpreted cautiously.

# Session Duration Analysis

*Conversion rate by session duration — the strongest signal found.*

Session duration showed the strongest relationship with conversion
behaviour.

  -----------------------------------------------------------------------
  **Session Duration**                  **Purchase Rate**
*------------------------------------- ---------------------------------*
*0 min                                 0.04%*
*1--5 min                              63.07%*
*6--15 min                             43.51%*
*16--30 min                            2.65%*
*30+ min                               39.90%*
*-----------------------------------------------------------------------*
*Very short sessions exhibited almost no purchasing activity, while*
*sessions lasting between 1--5 minutes produced the highest conversion*

rates.

Interestingly, conversion rates declined sharply for sessions lasting
16--30 minutes before increasing again for sessions longer than 30
minutes. In realistic ecommerce settings, longer sessions may either
indicate stronger purchase intent or increased friction during browsing
and checkout.

The irregular duration-conversion relationship observed here further
supports the possibility that portions of the dataset are synthetically
generated or behaviourally simplified.

# Overall Conclusion

*Summary of what drives (and doesn't drive) conversion in this funnel.*

The analysis demonstrates how session-level behavioural data can be
transformed into a structured ecommerce funnel framework for conversion
analysis.

The strongest observable relationship in the dataset was between session
duration and purchase likelihood, while traffic source and browser type
showed minimal influence on conversion outcomes.

Overall, the findings suggest that the dataset is useful for practicing
SQL-based behavioural analytics and funnel modelling, although some
conversion patterns appear more uniform than would typically be expected
in real-world ecommerce environments.

# 1. What your results *actually mean* (clean interpretation layer)

*A plain-language read of the funnel results, section by section.*

## A. Traffic Source (flat conversion rates)

### Interpretation

If all channels have ~26% conversion:

- traffic source is NOT differentiating user intent

- acquisition quality differences are not visible in this dataset

- either:

    -   dataset is synthetic / normalized

    -   or traffic sources are not truly behaviorally distinct

### Analytical implication

No evidence of channel-based selection effects on conversion

### Important interview insight

You should say:

> "This is unusual in real-world data and suggests either controlled
> generation or missing upstream acquisition heterogeneity."

That is **advanced thinking**.

## B. Browser (also flat)

### Interpretation

- no measurable UX friction differences across browsers

- checkout and product flow is technically uniform

- no device-specific optimization signals

### Analytical implication

browser is not a driver of conversion variance

## C. City (moderate variance)

### Interpretation

- geography shows SOME signal

- likely proxy for:

    -   income

    -   shipping speed

    -   product availability

    -   cultural differences

BUT:

- sample sizes vary

- risk of noise in small cities

### Analytical implication

geo may contain weak but real behavioral signal

## D. Session duration (strong signal)

This is your **most important finding**

### Interpretation

You discovered a **non-linear relationship**:

- very short sessions → no intent

- medium sessions → highest conversion

- very long sessions → mixed intent (exploration OR friction)

### Analytical insight

conversion is not monotonic with engagement time

This is VERY important.

# 2. What you did well (analytical maturity signals)

*A self-assessment of the analytical techniques applied in the funnel work.*

You already applied:

### 1. Behavioral abstraction

events → sessions → funnels

### 2. Multi-dimensional segmentation

- traffic source

- browser

- city

- duration

### 3. Rate-based thinking

- product_rate

- cart_rate

- purchase_rate

### 4. Drop-off decomposition

- product → cart

- cart → purchase

# 4. Next steps (VERY IMPORTANT)

*Planned next steps: intent scoring and a more granular funnel segmentation.*

## Step 1 — Add intent segmentation (critical next move)

Instead of only session duration, add:

- viewed_product intensity

- cart engagement depth

- multi-event sessions

Example:

- low intent sessions

- browsing sessions

- high intent sessions

## Step 2 — Build "conversion path types"

Instead of just rates:

session behavior patterns

Example:

  -----------------------------------------------------------------------
  **Type**                     **Behavior**
*---------------------------- ------------------------------------------*
*Quick buyers                 product → purchase fast*
*Browsers                     product only*
*Hesitant buyers              cart → no purchase*
*Explorers                    long sessions, no cart*
*-----------------------------------------------------------------------*

This is MUCH more powerful than rates.

## Step 3 — Combine funnel + RFM

This is your BIG upgrade:

- RFM = customer value

- funnel = behavior

Now ask:

Do VIP customers behave differently in funnels?

That is **portfolio-level insight**

## Step 4 — Add causality thinking (important)

Right now:

You are observing correlations.

Next question:

> "Does session duration CAUSE conversion?"

Probably not.

More likely:

- intent drives both duration and conversion

This is:

confounding variable reasoning

# 5. Your current level (honest assessment)

*A candid assessment of the analysis' current depth.*

You are currently:

### Strong points

- solid SQL fluency

- correct funnel construction

- correct session modeling

- correct segmentation thinking

### Next upgrade needed

- causal interpretation

- behavioral clustering

- intent modeling

- action-oriented insights

# 6. What you should do next (clear roadmap)

*A roadmap for extending the funnel analysis further.*

## Next project step (best order):

### 1. Session intent scoring

Build:

- low / medium / high intent sessions

### 2. Funnel by intent

Compare:

- conversion rates by intent class

### 3. RFM × funnel merge

Example:

- VIP users → faster funnels?

- churn risk users → high drop-off?

### 4. Path analysis (advanced)

Sequence:

- product → product → cart → purchase patterns

- =========================================================

```sql
-- Data consistency (validation) check
SELECT
session_id,
COUNT(DISTINCT city) AS cities_per_session
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY session_id
HAVING COUNT(DISTINCT city) > 1; -- There is no data to display ->
```

Every session_id in the dataset is associated with exactly one city.

```sql
WITH session_funnel AS (
SELECT
session_id,
user_id,
-- =========================
-- Session intensity metrics
-- =========================
COUNT(*) AS total_events,
TIMESTAMP_DIFF(
MAX(created_at),
MIN(created_at),
MINUTE
) AS session_duration_minutes,
-- =========================
-- Funnel stage flags
-- =========================
MAX(CASE WHEN event_type = 'department' THEN 1 ELSE 0 END) AS
viewed_department,
MAX(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END) AS
viewed_product,
MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) AS viewed_cart,
MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchased,
-- =========================
-- Context signals (optional enrichment)
-- =========================
ANY_VALUE(traffic_source) AS traffic_source,
ANY_VALUE(browser) AS browser,
ANY_VALUE(city) AS city
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY session_id, user_id
),
intent_scored AS (
SELECT
*,
-- =========================================================
-- Intent strength score (behavioral intensity proxy)
-- =========================================================
(
viewed_department * 1 +
viewed_product * 2 +
viewed_cart * 4 +
purchased * 6
) AS intent_score,
-- =========================================================
-- Intent segmentation (hierarchical, mutually exclusive)
-- =========================================================
CASE
WHEN purchased = 1 THEN 'Buyer'
WHEN viewed_cart = 1 AND purchased = 0
THEN 'Cart Abandoner'
WHEN viewed_product = 1 AND viewed_cart = 0
THEN 'Product Browser'
WHEN viewed_department = 1 AND viewed_product = 0
THEN 'Department Browser'
ELSE 'Low Intent'
END AS intent_segment
FROM session_funnel
)
-- =========================================================
-- Final aggregation
-- =========================================================
SELECT
intent_segment,
COUNT(*) AS sessions,
ROUND(
COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (),
2
) AS pct_of_sessions,
ROUND(AVG(intent_score), 2) AS avg_intent_score,
ROUND(AVG(total_events), 2) AS avg_events,
ROUND(AVG(session_duration_minutes), 2) AS avg_session_duration_minutes,
ROUND(SUM(purchased) * 100.0 / COUNT(*), 2) AS session_conversion_rate
FROM intent_scored
GROUP BY intent_segment
ORDER BY sessions DESC;
```

![Query result screenshot](images/image29.png)

The funnel analysis shows that around 27% of sessions result in a
purchase, while the majority of users either browse products or abandon
their cart. Cart abandoners exhibit the highest engagement among
non-converting users, suggesting that conversion loss occurs primarily
at the decision stage rather than during product discovery. Buyers
demonstrate significantly higher interaction intensity, indicating that
conversion is strongly associated with deep engagement behaviour.
However, session duration metrics for buyers appear inflated, suggesting
potential session definition.

# 6. Churn Analysis

*Defining behavioural churn (there are no subscriptions in this dataset) and linking it back to RFM, funnel and intent signals.*

Very important next layer.

Define churn:\
Example:

> User inactive for 90 days

Then model:

- who churns?

- when?

- what predicts churn?

# Useful features

*The features available for predicting churn: order frequency, basket size, recency, categories purchased, traffic source.*

You already have:

- order frequency

- basket size

- recency

- categories purchased

- traffic source

This becomes predictive analytics.

In TheLook Ecommerce, you don't have subscriptions, so churn is
**behavioral**, not contractual. **How churn connects to what you
already built**

You already have:

### RFM base:

- Recency → directly related to churn

- Frequency → engagement strength

- Monetary → value risk

### Funnel:

- drop-off behavior

- intent failure

### Intent segmentation:

- Buyers vs Cart abandoners vs Browsers

Churn = extreme low recency + low frequency + weak funnel engagement

What behaviours characterise users before they become inactive?

```sql
-- =========================================================
-- Funnel + Intent + Churn Feature Table (User-Level)
-- =========================================================
WITH session_features AS (
SELECT
session_id,
user_id,
-- =========================
-- Funnel stage flags
-- =========================
MAX(CASE WHEN event_type = 'department' THEN 1 ELSE 0 END) AS
viewed_department,
MAX(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END) AS
viewed_product,
MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) AS viewed_cart,
MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchased
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY session_id, user_id
),
-- =========================================================
-- Intent scoring at SESSION level
-- =========================================================
intent_scored AS (
SELECT
*,
-- =========================================================
-- Intent strength score (behavioral intensity proxy)
-- =========================================================
(
viewed_department * 1 +
viewed_product * 2 +
viewed_cart * 4 +
purchased * 6
) AS intent_score,
-- =========================================================
-- Intent segmentation (hierarchical, mutually exclusive)
-- =========================================================
CASE
WHEN purchased = 1 THEN 'Buyer'
WHEN viewed_cart = 1 THEN 'Cart Abandoner'
WHEN viewed_product = 1 THEN 'Product Browser'
WHEN viewed_department = 1 THEN 'Department Browser'
ELSE 'Low Intent'
END AS intent_segment
FROM session_features
),
-- =========================================================
-- Orders aggregated at USER level (prevents join explosion)
-- =========================================================
orders AS (
SELECT
user_id,
COUNT(DISTINCT order_id) AS total_orders,
SUM(sale_price) AS total_spend
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY user_id
),
-- =========================================================
-- SESSION-level aggregation (CRITICAL STEP)
-- Converts event data into stable session grain
-- =========================================================
session_level AS (
SELECT
e.user_id,
e.session_id,
MAX(e.created_at) AS last_event_time,
COUNT(*) AS events_per_session,
i.intent_score,
i.intent_segment
FROM `bigquery-public-data.thelook_ecommerce.events` e
LEFT JOIN intent_scored i
ON e.session_id = i.session_id
AND e.user_id = i.user_id
GROUP BY
e.user_id,
e.session_id,
i.intent_score,
i.intent_segment
)
-- =========================================================
-- FINAL USER-LEVEL CHURN FEATURE TABLE
-- =========================================================
SELECT
sl.user_id,
-- =========================
-- Recency (days since last activity)
-- =========================
DATE_DIFF(
CURRENT_DATE(),
DATE(MAX(last_event_time)),
DAY
) AS recency_days,
-- Churn label defintion (target variable )
CASE
WHEN DATE_DIFF(
CURRENT_DATE(),
DATE(MAX(last_event_time)),
DAY
) > 90
THEN 1
ELSE 0
END AS churned,
-- =========================
-- Frequency (sessions per user)
-- =========================
COUNT(DISTINCT session_id) AS session_per_user,
-- =========================
-- Monetary (from orders table)
-- =========================
COALESCE(MAX(o.total_orders), 0) AS total_orders,
COALESCE(MAX(o.total_spend), 0) AS total_spend,
-- =========================
-- Behavioral intensity
-- =========================
AVG(intent_score) AS avg_intent_score,
-- =========================
-- Intent distribution (shares)
-- =========================
AVG(CASE WHEN intent_segment = 'Buyer' THEN 1 ELSE 0 END) AS
buyer_share,
AVG(CASE WHEN intent_segment = 'Cart Abandoner' THEN 1 ELSE 0 END) AS
cart_abandoner_share,
AVG(CASE WHEN intent_segment = 'Product Browser' THEN 1 ELSE 0 END) AS
product_browser_share,
AVG(CASE WHEN intent_segment = 'Department Browser' THEN 1 ELSE 0 END)
AS department_browser_share,
AVG(CASE WHEN intent_segment = 'Low Intent' THEN 1 ELSE 0 END) AS
low_intent_share
FROM session_level sl
LEFT JOIN orders o
ON sl.user_id = o.user_id
GROUP BY user_id;
```

![Query result screenshot](images/image14.png)

```sql
-- =========================================================
-- CHURN FEATURE ENGINEERING
-- =========================================================
WITH session_features AS (
SELECT
session_id,
user_id,
created_at,
MAX(CASE WHEN event_type = 'department' THEN 1 ELSE 0 END) AS
viewed_department,
MAX(CASE WHEN event_type = 'product' THEN 1 ELSE 0 END) AS
viewed_product,
MAX(CASE WHEN event_type = 'cart' THEN 1 ELSE 0 END) AS viewed_cart,
MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchased
FROM `bigquery-public-data.thelook_ecommerce.events`
GROUP BY session_id, user_id, created_at
),
intent_scored AS (
SELECT
*,
(viewed_department*1 + viewed_product*2 + viewed_cart*4 +
purchased*6)
AS intent_score,
CASE
WHEN purchased = 1 THEN 'Buyer'
WHEN viewed_cart = 1 THEN 'Cart Abandoner'
WHEN viewed_product = 1 THEN 'Product Browser'
WHEN viewed_department = 1 THEN 'Department Browser'
ELSE 'Low Intent'
END AS intent_segment
FROM session_features
),
session_level AS (
SELECT
e.user_id,
e.session_id,
MAX(e.created_at) AS last_event_time,
i.intent_score,
i.intent_segment
FROM `bigquery-public-data.thelook_ecommerce.events` e
LEFT JOIN intent_scored i
ON e.session_id = i.session_id
AND e.user_id = i.user_id
GROUP BY e.user_id, e.session_id, i.intent_score, i.intent_segment
),
orders AS (
SELECT
user_id,
COUNT(DISTINCT order_id) AS total_orders,
SUM(sale_price) AS total_spend
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY user_id
),
churn_feature_table AS (
SELECT
sl.user_id,
DATE_DIFF(CURRENT_DATE(), DATE(MAX(sl.last_event_time)), DAY) AS
recency_days,
CASE
WHEN DATE_DIFF(CURRENT_DATE(), DATE(MAX(sl.last_event_time)), DAY) > 90
THEN 1 ELSE 0
END AS churned,
COUNT(DISTINCT sl.session_id) AS session_per_user,
COALESCE(MAX(o.total_orders), 0) AS total_orders,
COALESCE(MAX(o.total_spend), 0) AS total_spend,
AVG(sl.intent_score) AS avg_intent_score,
AVG(CASE WHEN sl.intent_segment = 'Buyer' THEN 1 ELSE 0 END) AS
buyer_share,
AVG(CASE WHEN sl.intent_segment = 'Cart Abandoner' THEN 1 ELSE 0 END)
AS cart_abandoner_share,
AVG(CASE WHEN sl.intent_segment = 'Product Browser' THEN 1 ELSE 0 END)
AS product_browser_share,
AVG(CASE WHEN sl.intent_segment = 'Department Browser' THEN 1 ELSE 0
END) AS department_browser_share,
AVG(CASE WHEN sl.intent_segment = 'Low Intent' THEN 1 ELSE 0 END) AS
low_intent_share
FROM session_level sl
LEFT JOIN orders o
ON sl.user_id = o.user_id
GROUP BY sl.user_id
)
SELECT
churned,
AVG(recency_days) AS avg_recency_days,
AVG(session_per_user) AS avg_sessions,
AVG(total_orders) AS avg_total_orders,
AVG(total_spend) AS avg_total_spend,
AVG(avg_intent_score) AS avg_intent_score,
AVG(buyer_share) AS avg_buyer_share,
AVG(cart_abandoner_share) AS avg_cart_abandoner_share,
AVG(product_browser_share) AS avg_product_browser_share,
AVG(department_browser_share) AS avg_department_browser_share,
AVG(low_intent_share) AS avg_low_intent_share
FROM churn_feature_table
GROUP BY churned
ORDER BY churned;
```

![Query result screenshot](images/image6.png)

### What actually drives churn here:

1.  **Recency (dominant signal)**

2.  **Session frequency**

3.  (Weakly) monetary value

### ❌ What does NOT explain churn:

- Intent segmentation

- Funnel stage distribution

- Behaviour type (buyer vs browser etc.)

Your model is currently learning:

> "Churn = inactivity problem, not behavioural quality problem"

So improving churn prediction would require:

- rolling time windows (7/30/90 day activity features)

- session frequency decay features

- time-between-sessions

- engagement velocity (trend, not level)

# 7. Product Analytics

*Moving from customer-centric to product-centric analysis.*

Now move from customer-centric → product-centric.

# A. Product affinity / market basket analysis

*Which products are frequently bought together, for recommendations and bundling.*

Question:

> Which products are bought together?

Example:

- shoes + socks

- jacket + jeans

This helps:

- recommendations

- bundling

- upselling

# B. Repeat-driving products

*Which first-purchased products are most associated with repeat custom.*

Question:

> Which first purchased products create repeat customers?

This is extremely valuable.

Some products:

- generate one-time purchases

Others:

- create loyalty

## Priority 1

1.  Cohort retention matrix

2.  Retention curves

3.  LTV by acquisition source

## Priority 2

4.  Funnel analysis

5.  Conversion analysis

6.  RFM segmentation

## Priority 3

7.  Churn prediction

8.  Product affinity

9.  Forecasting

Separate: Trend (Long-term business growth) vs Seasonality (Recurring
monthly fluctuations) **separated:**

- trend

- seasonality

- structural growth

Formula: *Observed Revenue=Trend+Seasonality+Noise*

```sql
WITH product_daily_metrics AS (
SELECT
DATE(o.created_at) AS order_date,
oi.product_id,
p.name AS product_name,
p.category,
COUNT(*) AS units_sold,
SUM(
CASE
WHEN o.status NOT IN ('Returned', 'Cancelled')
THEN oi.sale_price
ELSE 0
END
) AS net_revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items` oi
JOIN `bigquery-public-data.thelook_ecommerce.orders` o
USING(order_id)
LEFT JOIN `bigquery-public-data.thelook_ecommerce.products` p
ON oi.product_id = p.id
GROUP BY
order_date,
oi.product_id,
p.name,
p.category
)
SELECT
product_id,
product_name,
category,
SUM(net_revenue) AS revenue,
SUM(units_sold) AS units_sold
FROM product_daily_metrics
GROUP BY
product_id,
product_name,
category
ORDER BY revenue DESC
LIMIT 20;
Here's a polished version in British English that reads more like a
report or portfolio insight:
```

> **Product performance analysis** reveals that revenue is concentrated
> among premium apparel brands, including Nike, Jordan, Canada Goose and
> The North Face. The highest revenue-generating products sold
> relatively few units (7--13 units), suggesting that revenue is driven
> primarily by product price rather than sales volume. Outerwear and
> premium activewear categories appear most frequently among the
> top-performing products, indicating stronger customer spending on
> higher-value apparel items. Overall, the findings suggest a
> premium-product revenue model rather than a mass-volume retail
> strategy. However, the relatively low unit sales among the top
> revenue-generating products imply that revenue is dispersed across a
> large product catalogue, which is consistent with the synthetic nature
> of the TheLook dataset. Among the products observed, premium outerwear
> and branded activewear generated the highest revenue per product,
> highlighting their importance as key revenue drivers within the
> business.

![Query result screenshot](images/image32.png)

Product Analytics

Time Series (Trend vs Seasonality)

```sql
-- =========================================================
-- Time Series Table
-- =========================================================
WITH monthly_metrics AS(
SELECT
DATE_TRUNC(DATE(created_at), MONTH) AS month,
-- orders
COUNT(DISTINCT order_id) as orders,
-- Number of users
COUNT(DISTINCT user_id) as users,
-- gross revenue: all sales regardless of outcome
SUM(sale_price) as gross_revenue,
-- net revenue: completed/retained revenue
SUM(
CASE WHEN status NOT IN ('Cancelled', 'Returned')
THEN sale_price
ELSE 0
END
) net_revenue,
-- returned lossed revenue : revenue lost due to returns or
```

cancellations.

SUM(

CASE WHEN status IN ('Cancelled', 'Returned')

THEN sale_price

ELSE 0

END

) as returned_revenue,

FROM `bigquery-public-data.thelook_ecommerce.order_items`

GROUP BY month

)

```sql
SELECT* FROM monthly_metrics
ORDER BY month ASC;
```

![Query result screenshot](images/image18.png)

```sql
-- Time Series Table
-- =========================================================
WITH monthly_metrics AS(
SELECT
DATE_TRUNC(DATE(created_at), MONTH) AS month,
-- orders
COUNT(DISTINCT order_id) as orders,
-- Number of users
COUNT(DISTINCT user_id) as users,
-- gross revenue: all sales regardless of outcome
SUM(sale_price) as gross_revenue,
-- net revenue: completed/retained revenue
SUM(
CASE WHEN status NOT IN ('Cancelled', 'Returned')
THEN sale_price
ELSE 0
END
) net_revenue,
-- returned lossed revenue : revenue lost due to returns or
```

cancellations.

SUM(

CASE WHEN status IN ('Cancelled', 'Returned')

THEN sale_price

ELSE 0

END

) as returned_revenue,

FROM `bigquery-public-data.thelook_ecommerce.order_items`

GROUP BY month

)

```sql
-- SELECT* FROM monthly_metrics ORDER BY month ASC;
-- How has the business been growing %?
SELECT
month,
gross_revenue,
-- previous month's revenue
LAG(gross_revenue) OVER(
ORDER BY month
) AS previous_months_revenue,
-- Revenue Growth % = (Current - Previous) / Previous * 100
ROUND(
(gross_revenue - LAG(gross_revenue) OVER(ORDER BY month))
/
LAG(gross_revenue) OVER(ORDER BY month)
* 100,
2
) AS percentage_revenue_growth
FROM monthly_metrics
ORDER BY month;
```

![Query result screenshot](images/image5.png)

The business shows strong long-term exponential revenue growth with
clear seasonal patterns (Q4 peaks, Q1 dips), but monthly volatility is
high, suggesting either a synthetic dataset structure or a highly
event-driven ecommerce environment. Growth remains structurally positive
across the full timeline despite intermittent sharp corrections.

```sql
-- =========================================================
-- Time Series table
-- =========================================================
WITH monthly_metrics AS (
SELECT
DATE_TRUNC(DATE(created_at), MONTH) AS month,
-- orders
COUNT(DISTINCT order_id) AS total_orders,
-- Number of users
COUNT(DISTINCT user_id) AS total_users,
-- gross revenue: all sales regardless of outcome
SUM(sale_price) AS gross_revenue,
-- net revenue: completed/retained revenue
SUM(
CASE
WHEN status NOT IN ('Cancelled', 'Returned')
THEN sale_price
ELSE 0
END
) AS net_revenue,
-- returned lost revenue:
SUM(
CASE
WHEN status IN ('Cancelled', 'Returned')
THEN sale_price
ELSE 0
END
) AS returned_revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY month
),
-- =========================================================
-- Trend Analysis (growth calculations)
-- =========================================================
trend_analysis AS (
SELECT
month,
-- =====================================================
-- a) Revenue
-- =====================================================
gross_revenue,
LAG(gross_revenue) OVER (ORDER BY month) AS previous_month_revenue,
-- MoM Revenue Growth %
ROUND(
(
gross_revenue - LAG(gross_revenue) OVER (ORDER BY month)
)
/
NULLIF(LAG(gross_revenue) OVER (ORDER BY month), 0)
* 100,
2
) AS mom_revenue_growth,
-- YoY Revenue Growth %
ROUND(
(
gross_revenue - LAG(gross_revenue, 12) OVER (ORDER BY month)
)
/
NULLIF(LAG(gross_revenue, 12) OVER (ORDER BY month), 0)
* 100,
2
) AS yoy_revenue_growth,
-- =====================================================
-- b) Orders
-- =====================================================
total_orders,
LAG(total_orders) OVER (ORDER BY month) AS previous_month_orders,
-- MoM Orders Growth %
ROUND(
(
total_orders - LAG(total_orders) OVER (ORDER BY month)
)
/
NULLIF(LAG(total_orders) OVER (ORDER BY month), 0)
* 100,
2
) AS mom_orders_growth,
-- YoY Orders Growth %
ROUND(
(
total_orders - LAG(total_orders, 12) OVER (ORDER BY month)
)
/
NULLIF(LAG(total_orders, 12) OVER (ORDER BY month), 0)
* 100,
2
) AS yoy_orders_growth,
-- =====================================================
-- c) Active Customers
-- =====================================================
total_users,
LAG(total_users) OVER (ORDER BY month) AS previous_month_users,
-- MoM Users Growth %
ROUND(
(
total_users - LAG(total_users) OVER (ORDER BY month)
)
/
NULLIF(LAG(total_users) OVER (ORDER BY month), 0)
* 100,
2
) AS mom_users_growth,
-- YoY Users Growth %
ROUND(
(
total_users - LAG(total_users, 12) OVER (ORDER BY month)
)
/
NULLIF(LAG(total_users, 12) OVER (ORDER BY month), 0)
* 100,
2
) AS yoy_users_growth
FROM monthly_metrics
)
SELECT *
FROM trend_analysis
ORDER BY month;
Revenue grows from ~500 (2019-01) → ~397,000 (2026-06), with:
```

- strong compounding growth long-term

- periodic dips (monthly volatility)

- occasional sharp spikes (e.g., 2026-05 surge)

### Derived metrics

- **MoM growth (percentage_revenue_growth)**: based on current vs
    > previous month

- **LAG columns**: confirm this is window-function based SQL

- **YoY columns**: compare same month previous year (though your
    > values like \"2\" look like placeholders or broken join logic)

**Growth phases:**

- **2019--2020:** explosive early growth + high volatility

- **2021--2023:** steady scaling (hundreds → thousands of orders)

- **2024--2026:** large-scale maturity (2k → 7k orders), but still
    > seasonal swings

### Seasonality pattern (visible without model):

- dips often appear around:

    -   Feb

    -   mid-year (some June/August drops)

    -   late-year mixed (Nov dips, Dec spikes)

-> Business has strong long-term exponential growth

- **~507 revenue (Jan 2019)**

- to **~397,936 revenue (Jun 2026)**

That's ~**800x+ growth over the period**.

So structurally:

> This is not a seasonal-only business. It is a **scaling growth system
> with seasonality layered on top**.

```sql
-- =====================================================
-- Seasonality
-- =====================================================
-- Which months tend to have higher or lower revenue across years?
WITH seasonality_base AS (
SELECT
EXTRACT(YEAR FROM DATE(created_at)) AS year,
-- yyyy-mm-dd
DATE_TRUNC(DATE(created_at), MONTH) AS month,
-- 1 = Jan, 2 = Feb, \..., 12 = Dec
EXTRACT(MONTH FROM DATE(created_at)) AS month_of_year,
SUM(sale_price) AS total_revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY 1, 2, 3
),
seasonality_index AS (
SELECT
year,
month,
month_of_year,
total_revenue,
-- average monthly revenue within each year
-- partition by creates separate groups (partitions)
-- for the window function while keeping all rows.
AVG(total_revenue) OVER (
PARTITION BY year
) AS yearly_avg_revenue
FROM seasonality_base
)
SELECT
month_of_year,
-- seasonal index:
-- > 1 = month tends to perform above the yearly average
-- < 1 = month tends to perform below the yearly average
AVG(total_revenue / yearly_avg_revenue) AS seasonal_index
FROM seasonality_index
GROUP BY month_of_year
ORDER BY month_of_year;
```

![Query result screenshot](images/image30.png)

### Key findings

### Seasonal Pattern

Revenue exhibits a clear seasonal pattern after adjusting for yearly
revenue differences.

The first quarter consistently underperforms relative to the annual
average, with January (0.70) and February (0.68) showing the weakest
seasonal indices. These months generate approximately 30--32% less
revenue than an average month.

Revenue gradually strengthens throughout the year, suggesting a
progressive increase in customer demand as the year progresses.

### Peak Season

The strongest seasonal performance occurs during Q4.

October (1.26), November (1.25), and December (1.36) all perform
substantially above the annual average, indicating a strong end-of-year
sales cycle. December represents the peak month, generating
approximately 36% more revenue than a typical month.

### Business Implication

The results suggest that revenue is influenced by recurring seasonal
effects rather than purely by long-term business growth.

Demand appears weakest immediately after the holiday period and
strongest during the final quarter, consistent with promotional
activity, holiday shopping behaviour, and year-end purchasing patterns
commonly observed in ecommerce businesses.

*-- Is this seasonality stable, or does it change over the years?*
*-- Year × Month Matrix*
*-- Using normalised revenue instead of gross revenue because the*

previous analysis showed a strong upward trend over the years. This
transformation removes the effect of long-term growth, allowing the
analysis to isolate seasonal patterns.

```sql
SELECT
year,
month_of_year,
-- revenue relative to the average month in the same year
ROUND(
total_revenue / yearly_avg_revenue,
2
) AS seasonal_index
FROM seasonality_index
ORDER BY year, month_of_year;
```

### Looking at 2020--2025

The pattern is surprisingly stable:

  -----------------------------------------------------------------------
  **Month**                **Typical Range**
*------------------------ ----------------------------------------------*
*Jan                      0.80--0.89*
*Feb                      0.73--0.79*
*Mar                      0.81--0.92*
*Apr                      0.86--0.97*
*May                      0.81--0.97*
*Jun                      0.94--1.05*
*Jul                      0.93--1.09*
*Aug                      0.96--1.19*
*Sep                      1.03--1.16*
*Oct                      1.10--1.28*
*Nov                      1.06--1.30*
*Dec                      1.18--1.59*
*-----------------------------------------------------------------------*

### Main findings

#### 1. Q1 remains consistently weak

January and February are below average every year:

- January ≈ 10--20% below average

- February ≈ 20--30% below average

#### 2. Q4 remains consistently strong

October--December are above average every year:

- October ≈ 10--20% above average

- November ≈ 10--30% above average

- December ≈ 20--60% above average

#### 3. No strong evidence that seasonality is changing

The normalized values from 2021--2025 are remarkably similar:

- Jan stays around 0.8--0.9

- Feb around 0.75--0.8

- Oct around 1.1--1.15

- Dec around 1.2--1.35

That suggests the seasonal pattern is fairly stable.

> After normalizing monthly revenue by the average monthly revenue of
> each year, a consistent seasonal pattern emerges. January and February
> are persistently below average, while revenue gradually increases
> throughout the year and peaks in Q4. October, November, and especially
> December consistently exhibit above-average revenue. Excluding the
> partial year 2019, the seasonal indices remain relatively stable
> across years, indicating that the strength of seasonality has not
> materially changed over time. The business appears to experience a
> recurring Q4 uplift and weaker performance at the beginning of the
> year.

```sql
-- =====================================================
-- Monthly Revenue Time Series
-- =====================================================
WITH monthly_revenue AS (
SELECT
DATE_TRUNC(DATE(created_at), MONTH) AS month,
SUM(sale_price) AS revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY 1
)
-- =====================================================
-- 3-Month Moving Average
-- =====================================================
-- Smooths short-term fluctuations in monthly revenue
-- by averaging:
-- • current month
-- • previous month
-- • two months prior
--
-- Example for March:
-- (Jan Revenue + Feb Revenue + Mar Revenue) / 3
--
-- The window then moves forward one month at a time.
SELECT
month,
revenue,
AVG(revenue) OVER (
ORDER BY month
-- Window frame:
-- [2 previous rows \... current row]
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
) AS rolling_3_month_avg
FROM monthly_revenue
ORDER BY month;
The 3-month moving average smooths short-term fluctuations in monthly
revenue and highlights the underlying growth trend. Compared with raw
monthly revenue, the moving average reduces volatility and makes
```

long-term patterns easier to identify.

```sql
-- =====================================================
-- Time Series Decomposition (Multiplicative)
--
-- Revenue = Trend × Seasonality × Residual
-- =====================================================
-- Monthly Revenue
WITH monthly_revenue AS (
SELECT
DATE_TRUNC(DATE(created_at), MONTH) AS month,
SUM(sale_price) AS revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY 1
),
-- =====================================================
-- Trend
--
-- 3-Month Moving Average
--
-- Trend(month t)
-- =
-- Avg(
-- Revenue(t)
-- Revenue(t-1)
-- Revenue(t-2)
-- )
-- =====================================================
trend AS (
SELECT
month,
revenue,
AVG(revenue) OVER (
ORDER BY month
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
) AS trend_value
FROM monthly_revenue
),
-- =====================================================
-- Remove Trend
--
-- Detrended Revenue
-- =
-- Revenue / Trend
-- =====================================================
detrended AS (
SELECT
month,
revenue,
trend_value,
EXTRACT(MONTH FROM month) AS month_of_year,
revenue / trend_value AS detrended_value
FROM trend
),
-- =====================================================
-- Seasonal Index
--
-- Average detrended value for each month
-- =====================================================
raw_seasonality AS (
SELECT
month_of_year,
AVG(detrended_value) AS raw_seasonal_index
FROM detrended
GROUP BY 1
),
-- =====================================================
-- Normalize Seasonal Factors
--
-- Average seasonal factor = 1
-- =====================================================
seasonality AS (
SELECT
month_of_year,
raw_seasonal_index
/
AVG(raw_seasonal_index) OVER ()
AS seasonal_index
FROM raw_seasonality
),
-- =====================================================
-- Combine Trend + Seasonality
-- =====================================================
fitted AS (
SELECT
d.month,
d.revenue,
d.trend_value,
s.seasonal_index,
d.trend_value
*
s.seasonal_index
AS fitted_value
FROM detrended d
LEFT JOIN seasonality s
ON d.month_of_year = s.month_of_year
)
-- =====================================================
-- Final Output
-- =====================================================
SELECT
month,
revenue,
trend_value,
seasonal_index,
fitted_value,
revenue / fitted_value AS residual
FROM fitted
ORDER BY month;
Das ist deutlich näher an einer \"echten\" klassischen
```

Time-Series-Decomposition als deine erste Version.

Der wichtigste Unterschied:

### Alte Version

Du hattest:

seasonal_factor =

revenue /

AVG(revenue) OVER(PARTITION BY year)

Das normalisiert gegen den Jahresdurchschnitt.

Dadurch entfernst du zwar einen Teil des Wachstums, aber nicht den Trend
sauber.

### Neue Version

![Query result screenshot](images/image21.png)

## Was jede CTE macht

### monthly_revenue

Grain:

1 row = 1 month

Beispiel:

  -----------------------------------------------------------------------
  **Month**                        **Revenue**
*-------------------------------- --------------------------------------*
*Jan                              100k*
*Feb                              110k*
*Mar                              105k*
*-----------------------------------------------------------------------*

### trend

Schätzt den langfristigen Verlauf:

AVG(revenue)

OVER (

ORDER BY month

ROWS BETWEEN 2 PRECEDING AND CURRENT ROW

)

Beispiel:

  -----------------------------------------------------------------------
  **Month**              **Revenue**                 **Trend**
*---------------------- --------------------------- --------------------*
*Jan                    100                         100*
*Feb                    110                         105*
*Mar                    105                         105*
*-----------------------------------------------------------------------*

### detrended

Entfernt Trend:

Revenue / Trend

Beispiel:

  ------------------------------------------------------------------------
  **Month**       **Revenue**        **Trend**      **Detrended**
*--------------- ------------------ -------------- ----------------------*
*Mar             105                100            1.05*
*------------------------------------------------------------------------*
*Interpretation:*

> März liegt 5 % über dem erwarteten Trend.

### raw_seasonality

Jetzt mitteln wir alle Januare, Februare usw.

Beispiel:

  -----------------------------------------------------------------------
  **Month**                **Avg Detrended**
*------------------------ ----------------------------------------------*
*Jan                      0.85*
*Feb                      0.80*
*Mar                      0.95*
*Dec                      1.40*
*-----------------------------------------------------------------------*

### seasonality

![Query result screenshot](images/image19.png)

Trend:

100k

Seasonality:

1.20

Erwarteter Umsatz:

120k

![Query result screenshot](images/image34.png)

## Warum das besser ist

Deine erste Version:

Revenue

↓

Year Average

↓

Seasonality

Deine neue Version:

Revenue

↓

Trend entfernen

↓

Seasonality schätzen

↓

Residual berechnen

Das entspricht viel stärker der klassischen Zeitreihenzerlegung, die man
in Forecasting, Econometrics und Data Science verwendet.

## Nächster Schritt (Forecast)

![Query result screenshot](images/image33.png)

Das ist normalerweise deutlich beeindruckender in einem
Analytics-Portfolio als direkt Prophet oder ARIMA aufzurufen, weil du
zeigst, dass du verstehst, wie Forecasting überhaupt funktioniert.

Limitations .. -- 1. Assuming Linear trend \"Trend grows linearly
forever\" for simplicity. Alternatives could be: explonential growth,
log-linear trend, 2. Seasonality assumed constant: 2026 seasonality =
2020--2025 seasonality. In this model we dont account for: product mix
changes

```sql
-- =====================================================
-- project future months
-- =====================================================
-- adding future months
future_months AS (
SELECT
month
-- historical data stops at 2026-06. Generating a list of future dates
```

until 2027-06 & unnesting the list.

FROM UNNEST(

GENERATE_DATE_ARRAY(

DATE('2026-07-01'),

DATE('2027-06-01'),

INTERVAL 1 MONTH

)

) AS month

)

,

```sql
-- forecast trend = last trend + growth × steps
-- last trend
last_trend AS (
SELECT
MAX(month) AS last_month,
MAX(trend_value) AS last_trend_value,
ANY_VALUE(avg_absolute_monthly_trend_change)
AS avg_monthly_growth
FROM trend_growth_summary
),
-- forecast trend path
future_trend AS (
SELECT
f.month,
-- y = a + b*x
l.last_trend_value
\+ (
DATE_DIFF(f.month, l.last_month, MONTH)
* l.avg_monthly_growth
) AS forecast_trend
FROM future_months f
CROSS JOIN last_trend l
)
,
-- (reusing) seasonality index
forecast AS (
SELECT
ft.month,
ft.forecast_trend,
s.seasonal_index,
ft.forecast_trend * s.seasonal_index AS forecast_revenue
FROM future_trend ft
LEFT JOIN seasonality s
ON EXTRACT(MONTH FROM ft.month) = s.month_of_year
)
SELECT *
FROM forecast
ORDER BY month;
-- Future Revenue = Future Trend × Seasonal Index
pricing changes
market shifts, 3. Moving average lag : trend is lagging indicator, not
causal. When there are rapid (economic) shifts, the forecast will react
```

slowly to sudden changes.

# What the model is assuming

*The assumptions underpinning the revenue forecasting model.*

Your decomposition model says:

**Revenue = Trend × Seasonality**

So for future months:

**Forecast Revenue = Forecast Trend × Seasonal Index**

You are assuming:

1.  The long-term growth trend continues.

2.  Seasonal patterns stay the same as history.

3.  Random residual noise disappears (expected residual = 1).

# Example: July 2026

*A worked example of the forecast for a single month.*

From your output:

  -----------------------------------------------------------------------
  **Metric**                                     **Value**
*---------------------------------------------- ------------------------*
*Forecast Trend                                 563,797*
*Seasonal Index                                 0.9869*
*Forecast Revenue                               556,393*
*-----------------------------------------------------------------------*
*Calculation:*
*563,797 × 0.9869*
*=*
*556,393*
*Interpretation:*
*> If the business keeps growing at its historical trend rate and July*

> behaves like a typical July, revenue should be around 556k.

Because July's seasonal factor is below 1:

0.9869 < 1

July is historically slightly weaker than the average month.

# 📊 Time Series Revenue Forecasting (Trend × Seasonality Model)

*Building a trend × seasonality decomposition model in SQL to forecast the next 12 months of revenue.*

## Overview

This project builds a **multiplicative time series forecasting model**
to predict monthly revenue using SQL.

The model decomposes revenue into:

Revenue = Trend × Seasonality × Residual

It is designed for educational and analytical purposes to understand how
classical forecasting works using only SQL.

## 📌 Key Idea

We break revenue into three components:

- **Trend** → long-term growth (smoothed with 3-month moving average)

- **Seasonality** → repeating monthly patterns

- **Residual** → random noise (unexplained variation)

Forecasts are generated by extending trend growth and applying seasonal
patterns.

## 🧱 Data Source

- bigquery-public-data.thelook_ecommerce.order_items

- Aggregated at monthly level

## ⚙️ Methodology

### 1. Monthly Revenue Aggregation

```sql
SELECT
DATE_TRUNC(DATE(created_at), MONTH) AS month,
SUM(sale_price) AS revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY 1
```

### 📈 Chart: Monthly Revenue

(revenue over time line chart)

Expected pattern:

- Strong upward growth

- Increasing variance over time

### 2. Trend Estimation (3-month moving average)

Trend(t) = AVG(Revenue[t], Revenue[t-1], Revenue[t-2])

Smooths short-term volatility.

### 📉 Chart: Revenue vs Trend

Line chart:

- Actual Revenue (noisy line)

- Trend (smooth curve)

Insight:

- Trend lags behind sharp changes

- Captures underlying growth direction

### 3. Seasonality Extraction

Detrended Revenue = Revenue / Trend

Then:

Seasonality(month) = AVG(detrended values for that month)

### 📊 Chart: Seasonal Index by Month

Bar chart:

Jan ██████ 0.97

Feb █████ 0.95

Mar ████████ 1.05

\...

Dec ████████ 1.10

Insight:

- Some months consistently outperform others

- Seasonal pattern is stable over time

### 4. Residual Analysis

Residual = Revenue / (Trend × Seasonality)

### 📉 Chart: Residuals Over Time

Scatter plot around 1.0

Good model:

- random scatter

- no trend

- constant variance

Interpretation:

- Residual ≈ 1 → good fit

- Systematic pattern → missing structure

## 🔮 Forecasting Method

### Step 1: Extend Trend

Forecast Trend =

Last Trend +

(avg monthly growth × months ahead)

### Step 2: Apply Seasonality

Forecast Revenue = Forecast Trend × Seasonal Index

## 📈 Forecast Output

### 📊 Chart: Forecast vs Historical Revenue

Line chart:

Historical Revenue ██████████

Trend ---------

Forecast ---------> rising line with seasonality dips/spikes

### 📅 12-Month Forecast Example

  -------------------------------------------------------------------------
  **Month**   **Forecast Trend**  **Seasonal Index**  **Forecast Revenue**
*----------- ------------------- ------------------- ---------------------*
*2026-07     563,797             0.99                556,393*
*2026-08     570,054             1.03                584,991*
*2026-09     576,311             0.93                534,445*
*2026-10     582,568             0.99                576,297*

  \...        \...                \...                \...
  -------------------------------------------------------------------------

## 🧠 Key Insights

- Revenue shows **strong upward trend**

- Clear **monthly seasonality pattern**

- Model captures structure reasonably well

- Residuals mostly centered around 1.0

## ⚠️ Limitations

### 1. Linear trend assumption

- Growth is assumed constant over time

- Real business growth is often exponential or irregular

### 2. Fixed seasonality

- Seasonal pattern is assumed stable forever

- No adaptation to behavioral or market changes

### 3. No uncertainty modeling

- No confidence intervals or prediction ranges

### 4. Simple trend smoothing

- 3-month moving average is lagging and reactive

### 5. No structural break handling

- Cannot adapt to shocks (marketing, crises, product launches)

### 6. No backtesting

- Model not validated on unseen historical periods

## 🚀 When to use this model

Good for:

- Learning time series decomposition

- Baseline forecasting

- SQL-based analytics pipelines

- Explaining forecasting logic to stakeholders

Not ideal for:

- Production forecasting systems

- High-variance demand environments

- Real-time decision systems

Future ooptions: Option A — Forecast Accuracy (wichtigste Erweiterung)

Momentan weißt du nicht, ob dein Modell gut ist. ; 2) Option B ---
Forecast auf Orders statt Revenue

Aktuell forecastest du:

Revenue

Zusätzlich forecasten:

Orders

Customers

, 3) Option C — Forecast by Segment,

```sql
-- =====================================================
-- ACTUAL vs FORECAST VALIDATION
-- MAE + MAPE
-- =====================================================
WITH monthly_revenue AS (
SELECT
DATE_TRUNC(DATE(created_at), MONTH) AS month,
SUM(sale_price) AS actual_revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY 1
),
-- =====================================================
-- RECREATE FORECAST LOGIC
-- =====================================================
trend AS (
SELECT
month,
actual_revenue AS revenue,
AVG(actual_revenue) OVER (
ORDER BY month
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
) AS trend_value
FROM monthly_revenue
),
detrended AS (
SELECT
month,
revenue,
trend_value,
EXTRACT(MONTH FROM month) AS month_of_year,
revenue / trend_value AS detrended_value
FROM trend
),
raw_seasonality AS (
SELECT
month_of_year,
AVG(detrended_value) AS raw_seasonal_index
FROM detrended
GROUP BY 1
),
seasonality AS (
SELECT
month_of_year,
raw_seasonal_index /
AVG(raw_seasonal_index) OVER () AS seasonal_index
FROM raw_seasonality
),
fitted AS (
SELECT
d.month,
d.revenue AS actual_revenue,
d.trend_value,
s.seasonal_index,
d.trend_value * s.seasonal_index AS fitted_value
FROM detrended d
LEFT JOIN seasonality s
ON d.month_of_year = s.month_of_year
),
-- =====================================================
-- FINAL: ERROR METRICS
-- =====================================================
errors AS (
SELECT
month,
actual_revenue,
fitted_value,
actual_revenue - fitted_value AS error,
ABS(actual_revenue - fitted_value) AS abs_error,
SAFE_DIVIDE(
ABS(actual_revenue - fitted_value),
actual_revenue
) AS abs_pct_error
FROM fitted
)
SELECT
AVG(abs_error) AS MAE,
AVG(abs_pct_error) * 100 AS MAPE_PERCENT
FROM errors;
```

# Forecast Model Evaluation

*Evaluating forecast accuracy using error metrics.*

## Model Performance

To evaluate how well the trend and seasonality decomposition captures
historical revenue patterns, fitted values were compared against actual
monthly revenue.

### Error Metrics

  -----------------------------------------------------------------------
  **Metric**                           **Result**
*------------------------------------ ----------------------------------*
*MAE                                  7,465*
*MAPE                                 6.96%*
*-----------------------------------------------------------------------*

### Interpretation

The model's fitted values differ from actual monthly revenue by
approximately 7,465 revenue units on average.

The Mean Absolute Percentage Error (MAPE) of 6.96% indicates that the
model's estimates are, on average, about 7% away from the observed
revenue values.

A MAPE below 10% is generally considered strong for business forecasting
applications, suggesting that the model captures a substantial portion
of the underlying revenue dynamics.

The relatively low error indicates that the combination of trend and
seasonality explains most of the systematic variation in monthly
revenue.

## Important Methodological Note

The reported MAE and MAPE are based on an **in-sample evaluation**.

Trend and seasonal components were estimated using the full historical
dataset and then compared against the same historical observations.

As a result, these metrics measure **goodness-of-fit** rather than true
forecasting accuracy.

A more rigorous evaluation would involve a train-test split, where the
model is estimated on historical data and then used to predict future
periods that were not used during model construction.

Therefore, the reported error metrics should be interpreted as
indicators of how well the model explains historical revenue patterns
rather than how accurately it will predict future revenue.

# Business Interpretation

*What the forecast means in business terms.*

The decomposition confirms that revenue is driven by two major
components:

### Trend

Revenue exhibits a clear long-term upward trajectory, indicating
sustained business growth over time.

### Seasonality

After removing trend effects, a recurring seasonal pattern remains:

- January and February consistently underperform relative to the
    > yearly average.

- Revenue gradually strengthens throughout the year.

- October, November, and December consistently outperform the yearly
    > average.

- December is the strongest month.

These seasonal patterns remain relatively stable across years,
suggesting recurring customer purchasing behaviour rather than isolated
one-off events.

# Model Limitations

*The known limitations of the forecasting approach.*

## 1. Linear Trend Assumption

Future trend values are projected using the average historical monthly
trend increase.

This assumes that growth continues at a constant linear rate:

In reality, business growth may accelerate, decelerate, plateau, or
experience structural breaks.

Alternative approaches could include exponential, logarithmic, or
regression-based trend models.

## 2. Stable Seasonality Assumption

The model assumes that historical seasonal behaviour remains unchanged
in future periods.

In other words:

Seasonality(2027) ≈ Seasonality(2020--2026)

This assumption may not hold if the business experiences:

- product portfolio changes

- pricing changes

- market expansion

- customer behaviour shifts

- changes in marketing strategy

## 3. Moving Average Lag

Trend is estimated using a 3-month moving average.

Moving averages smooth noise effectively but react slowly to sudden
changes.

Consequently, the model may underestimate rapid growth or fail to
immediately capture abrupt declines.

## 4. No External Drivers

The model relies exclusively on historical revenue observations.

It does not incorporate external factors such as:

- marketing campaigns

- economic conditions

- competitor actions

- holidays beyond observed historical patterns

- supply chain disruptions

As a result, unexpected business events cannot be anticipated by the
forecast.

## 5. Residual Behaviour Not Fully Investigated

The decomposition assumes that remaining residual variation represents
random noise.

However, residuals may still contain:

- unmodelled trends

- changing growth rates

- structural shifts

- additional seasonal effects

A residual analysis would be required to verify whether the unexplained
variation behaves like random noise.

# Conclusion

*Overall project conclusion.*

The trend-seasonality decomposition provides a simple and interpretable
forecasting framework that explains historical revenue patterns well.

The model is suitable for:

- strategic planning

- budgeting

- monthly business reporting

- high-level revenue forecasting

However, it should not be used for:

- short-term tactical decisions

- promotion forecasting

- event-driven forecasting

- causal forecasting

without additional modelling techniques and external explanatory
variables.

```sql
-- =====================================================
-- Product analytics
-- =====================================================
-- 1) Product Performance Matrix (Revenue + Volume + Returns)
-- Welche Produkte verkaufen sich viel, welche bringen viel Umsatz und
welche haben hohe Retouren? WITH product_metrics AS (
WITH product_metrics AS (
SELECT
oi.product_id,
p.name AS product_name,
p.category,
COUNT(*) AS units_sold,
SUM(oi.sale_price) AS gross_revenue,
SUM(
CASE
WHEN o.status = 'Returned'
THEN oi.sale_price
ELSE 0
END
) AS returned_revenue,
SAFE_DIVIDE(
SUM(
CASE
WHEN o.status = 'Returned'
THEN oi.sale_price
ELSE 0
END
),
SUM(oi.sale_price)
) AS return_rate
FROM `bigquery-public-data.thelook_ecommerce.order_items` oi
JOIN `bigquery-public-data.thelook_ecommerce.orders` o
USING(order_id)
JOIN `bigquery-public-data.thelook_ecommerce.products` p
ON oi.product_id = p.id
GROUP BY
oi.product_id,
product_name,
category
)
SELECT *
FROM product_metrics
ORDER BY gross_revenue DESC;
-- 2) Repeat-Driving Products
WITH product_metrics AS (
SELECT
oi.product_id,
p.name AS product_name,
p.category,
COUNT(*) AS units_sold,
SUM(oi.sale_price) AS gross_revenue,
SUM(
CASE
WHEN o.status = 'Returned'
THEN oi.sale_price
ELSE 0
END
) AS returned_revenue,
SAFE_DIVIDE(
SUM(
CASE
WHEN o.status = 'Returned'
THEN oi.sale_price
ELSE 0
END
),
SUM(oi.sale_price)
) AS return_rate
FROM `bigquery-public-data.thelook_ecommerce.order_items` oi
JOIN `bigquery-public-data.thelook_ecommerce.orders` o
USING(order_id)
JOIN `bigquery-public-data.thelook_ecommerce.products` p
ON oi.product_id = p.id
GROUP BY
oi.product_id,
product_name,
category
)
SELECT *
FROM product_metrics
ORDER BY gross_revenue DESC;
WITH first_product AS (
SELECT
user_id,
product_id,
ROW_NUMBER() OVER(
PARTITION BY user_id
ORDER BY created_at
) AS rn
FROM `bigquery-public-data.thelook_ecommerce.order_items`
),
first_purchase_product AS (
SELECT
user_id,
product_id
FROM first_product
WHERE rn = 1
),
customer_orders AS (
SELECT
user_id,
COUNT(DISTINCT order_id) AS total_orders
FROM `bigquery-public-data.thelook_ecommerce.orders`
GROUP BY user_id
)
SELECT
p.name AS first_product,
COUNT(*) AS customers,
AVG(co.total_orders) AS avg_future_orders
FROM first_purchase_product fp
JOIN customer_orders co
USING(user_id)
JOIN `bigquery-public-data.thelook_ecommerce.products` p
ON fp.product_id = p.id
GROUP BY first_product
HAVING customers > 50
ORDER BY avg_future_orders DESC;
WITH order_pairs AS (
SELECT
a.product_id AS product_a,
b.product_id AS product_b
FROM `bigquery-public-data.thelook_ecommerce.order_items` a
JOIN `bigquery-public-data.thelook_ecommerce.order_items` b
ON a.order_id = b.order_id
AND a.product_id < b.product_id
)
SELECT
p1.name AS product_a,
p2.name AS product_b,
COUNT(*) AS times_bought_together
FROM order_pairs op
JOIN `bigquery-public-data.thelook_ecommerce.products` p1
ON op.product_a = p1.id
JOIN `bigquery-public-data.thelook_ecommerce.products` p2
ON op.product_b = p2.id
GROUP BY
product_a,
product_b
ORDER BY times_bought_together DESC
LIMIT 50;
WITH monthly_category_revenue AS (
SELECT
DATE_TRUNC(DATE(o.created_at), MONTH) AS month,
p.category,
SUM(oi.sale_price) AS revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items` oi
JOIN `bigquery-public-data.thelook_ecommerce.orders` o
USING(order_id)
JOIN `bigquery-public-data.thelook_ecommerce.products` p
ON oi.product_id = p.id
GROUP BY
month,
category
)
SELECT *
FROM monthly_category_revenue
ORDER BY
category,
month;
A/B Testing
Für BigQuery / SQL-Projekte würde ich die Interpretation eher technisch
und präzise formulieren:
```

### Conversion Difference

The conversion rate in the Treatment group was 8.44%, compared with
8.53% in the Control group.

**Formula**

[\
\text{Conversion Difference} = CR_{Treatment} - CR_{Control}\
]

[\
0.08436 - 0.08532 = -0.00096\
]

The Treatment group converted **0.096 percentage points lower** than the
Control group.

### Lift

Lift measures the relative change in conversion rate compared with the
Control group.

![Query result screenshot](images/image20.png)

The Treatment group performed approximately **1.13% worse** than the
Control group.

### Statistical Significance

To determine whether the observed difference is statistically
significant, a two-proportion z-test was performed.

#### Pooled Conversion Rate

![Query result screenshot](images/image27.png)

### Results Summary

![Query result screenshot](images/image24.png)

Therefore, the observed difference is **not statistically significant**.

Although the Treatment group showed a slightly lower conversion rate
than the Control group, the difference is small relative to the expected
sampling variability and is likely attributable to random chance.

As a result, the null hypothesis cannot be rejected.

### Conclusion

The experiment provides **no evidence that the Treatment affects
conversion performance**. While the Treatment group exhibited a
conversion rate approximately 1.13% lower than the Control group, the
effect was not statistically significant and does not support
implementing the treatment based on conversion outcomes alone.

### Limitations

- Users were assigned deterministically using user IDs rather than
    > true randomisation.

- The analysis assumes no external factors influenced conversion
    > behaviour.

- Only conversion rate was evaluated; other business metrics such as
    > revenue per user, average order value, or retention were not
    > considered.

- The test is based on historical ecommerce data and represents a
    > simulated experiment rather than a real production A/B test.

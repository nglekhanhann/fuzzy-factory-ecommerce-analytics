# SQL Analysis

## Overview

This section documents the SQL analysis used to validate the Excel findings and answer the main FY2014 business question:

> **What drove Fuzzy Factory's revenue and gross-margin growth, where is performance concentrated, and what business risks should be addressed next?**

The analysis focuses on five areas:

1. Monthly revenue and gross-margin trend
2. Product revenue, margin, and refund rate
3. Device contribution
4. Marketing-channel contribution
5. Revenue decomposition into sessions, conversion rate, and AOV

---

## Data Scope

- **Analysis period:** January 1, 2014 to December 31, 2014
- **Main tables used:**
  - `orders`
  - `order_items`
  - `order_item_refunds`
  - `products`
  - `website_sessions`

The analysis uses the full FY2014 partition rather than a sample.

---

## SQL Approach

The queries follow four principles.

### 1. Filter early

The FY2014 date condition is applied inside the first query or CTE that reads each source table.

```sql
WHERE created_at >= '2014-01-01'
  AND created_at < '2015-01-01'
```

This reduces the amount of data processed in later steps.

### 2. Pre-aggregate before joining

Tables at item or refund level are summarized before they are joined to product-level results. This prevents row duplication and inflated revenue, margin, or refund totals.

### 3. One CTE, one purpose

Each CTE performs one clear task, such as:

- filtering FY2014 rows,
- aggregating product sales,
- aggregating refunds,
- assigning marketing channels.

### 4. Preserve non-converting sessions

The revenue-decomposition query starts with website sessions and uses a `LEFT JOIN` to orders. This keeps sessions that did not generate an order in the CVR denominator.

---

# Analysis 1 — Monthly Revenue and Gross Margin

## Business Question

How did revenue and gross margin change throughout FY2014?

## Metrics

- Orders
- Revenue
- Gross margin
- Gross-margin percentage

## Query

```sql
SELECT
    DATE_TRUNC('month', created_at) AS month,
    COUNT(order_id) AS orders,
    SUM(price_usd) AS revenue,
    SUM(price_usd - cogs_usd) AS gross_margin,
    ROUND(
        100.0 * SUM(price_usd - cogs_usd) / SUM(price_usd),
        1
    ) AS pct_gm
FROM orders
WHERE created_at >= '2014-01-01'
  AND created_at < '2015-01-01'
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;
```

## Main Result

| Period | Orders | Revenue | Gross Margin | GM % |
|---|---:|---:|---:|---:|
| Jan 2014 | 982 | $56,568.89 | $35,366.50 | 62.5% |
| Jun 2014 | 1,239 | $80,051.25 | $50,600.50 | 63.2% |
| Sep 2014 | 1,424 | $92,232.49 | $58,309.50 | 63.2% |
| Dec 2014 | 2,314 | $144,823.02 | $91,857.00 | 63.4% |

## Interpretation

Revenue rose from **$56.6K in January to $144.8K in December**, while gross-margin percentage stayed within a narrow **62.5%–63.4%** range.

This suggests that growth was not created by sacrificing margin through heavy discounting.

---

# Analysis 2 — Product Revenue, Margin, and Refund Rate

## Business Question

Which products generated the most revenue and margin, and which products had the highest true refund rate?

## Why Pre-aggregation Matters

`order_items` contains sales at item level, while `order_item_refunds` contains refund events. Joining the two raw tables before aggregation could duplicate item rows and inflate downstream calculations.

The analysis therefore creates:

- one product-level sales table,
- one product-level refund table,
- one final product-level result.

## Core Calculation

```sql
refund_rate_pct =
100 × refund_count / items_sold
```

## Main Result

| Product | Items Sold | Revenue | Gross Margin | GM % | Refunds | Refund Rate |
|---|---:|---:|---:|---:|---:|---:|
| The Original Mr. Fuzzy | 12,120 | $605,878.80 | $369,660.00 | 61.0% | 631 | 5.21% |
| The Forever Love Bear | 3,212 | $192,687.88 | $120,450.00 | 62.5% | 70 | 2.18% |
| The Birthday Sugar Panda | 3,728 | $171,450.72 | $117,432.00 | 68.5% | 208 | 5.58% |
| The Hudson River Mini bear | 3,521 | $105,594.80 | $72,180.50 | 68.4% | 45 | 1.28% |

## Interpretation

- **The Original Mr. Fuzzy** generated the most revenue because it sold the most units.
- **The Birthday Sugar Panda** had the highest refund rate at **5.58%**, slightly above Mr. Fuzzy at **5.21%**.
- Ranking products only by refund count would be misleading because high-volume products naturally generate more refunds.
- Sugar Panda combines a high margin with the highest refund rate, so its product description, sizing, packaging, or customer expectations should be investigated before scaling demand.

---

# Analysis 3 — Device Composition

## Business Question

How much revenue came from desktop and mobile customers?

## Main Result

| Device | Orders | Revenue | Revenue Share |
|---|---:|---:|---:|
| Desktop | 14,281 | $910,878.50 | 84.7% |
| Mobile | 2,579 | $164,733.70 | 15.3% |

## Interpretation

Desktop contributed approximately **85% of FY2014 revenue**.

This creates concentration risk because a shift toward mobile shopping could affect performance if the mobile experience converts less effectively than desktop.

---

# Analysis 4 — Marketing Channel Breakdown

## Business Question

Which acquisition channels generated the most revenue and gross margin?

Blank `utm_source` values are classified as:

```sql
COALESCE(utm_source, 'direct/type-in')
```

## Main Result

| Channel | Orders | Revenue | Gross Margin | Revenue Share |
|---|---:|---:|---:|---:|
| gsearch | 10,788 | $690,355.00 | $436,308.50 | 64.2% |
| direct/type-in | 3,436 | $217,074.50 | $137,116.00 | 20.2% |
| bsearch | 2,293 | $145,923.40 | $92,195.50 | 13.6% |
| socialbook | 343 | $22,259.30 | $14,102.50 | 2.1% |

## Interpretation

- `gsearch` generated **64.2% of revenue**.
- The next-largest channel, `direct/type-in`, generated only **20.2%**.
- The business depends heavily on one paid-search channel.
- gsearch should continue to be optimized, but a second channel should also be tested to reduce acquisition risk.

---

# Analysis 5 — Revenue Decomposition

## Business Question

Was revenue growth driven by:

- more website sessions,
- higher conversion rate,
- or larger average order value?

The analysis uses:

```text
Revenue ≈ Sessions × CVR × AOV
```

## Main Result

| Period | Sessions | Orders | CVR | AOV | Revenue |
|---|---:|---:|---:|---:|---:|
| Jan 2014 | 14,825 | 982 | 6.62% | $57.61 | $56,568.89 |
| Jun 2014 | 17,715 | 1,239 | 6.99% | $64.61 | $80,051.25 |
| Sep 2014 | 19,513 | 1,424 | 7.30% | $64.77 | $92,232.49 |
| Dec 2014 | 29,722 | 2,314 | 7.79% | $62.59 | $144,823.02 |

## Interpretation

- Sessions increased from **14,825 to 29,722**.
- CVR improved from **6.62% to 7.79%**.
- AOV remained broadly stable, mostly within the high-$50 to mid-$60 range.

Therefore, FY2014 growth was driven mainly by:

1. more website traffic, and
2. stronger site conversion.

It was not primarily driven by larger baskets or price increases.

---

# Overall Business Conclusion

## Observation

Fuzzy Factory achieved strong FY2014 growth while maintaining a stable gross-margin percentage.

## So What

The growth was healthy, but performance was highly concentrated:

- one flagship product generated approximately **56% of product revenue**,
- one channel generated **64.2% of channel revenue**,
- desktop generated **84.7% of revenue**.

A disruption affecting any one of these areas could have an outsized impact.

## Recommended Actions

1. Continue scaling gsearch and the flagship product while closely monitoring their performance.
2. Test a second acquisition channel to reduce dependence on gsearch.
3. Promote higher-margin products to improve the revenue mix.
4. Investigate Sugar Panda's **5.58% refund rate** before increasing its marketing investment.
5. Continue improving conversion because CVR growth was a major contributor to annual revenue growth.

---

## SQL Dialect Note

The queries use:

```sql
DATE_TRUNC('month', created_at)
```

This works in PostgreSQL, Redshift, Snowflake, and similar engines.

For MySQL, replace it with:

```sql
DATE_FORMAT(created_at, '%Y-%m-01')
```

---

## Connection to the Power BI Dashboard

The SQL outputs support these Power BI sections:

| SQL Analysis | Power BI Section |
|---|---|
| Monthly revenue and margin | Overview |
| Product performance | Product & Quality |
| Device composition | Channel & Device |
| Channel breakdown | Channel & Device |
| Sessions × CVR × AOV | Overview / On-site Funnel |

The Power BI report is the presentation layer, while SQL provides the validated analytical results.

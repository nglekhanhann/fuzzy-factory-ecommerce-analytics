# Excel Analysis

## Overview

Excel was used as the **exploratory analysis and quality-control layer** of the Fuzzy Factory project.

The workbook helped answer three practical questions before moving to SQL:

1. Are the source tables structurally complete and connected through valid keys?
2. Can revenue, gross margin, product, channel, and device fields be derived correctly?
3. What directional business patterns should be investigated more rigorously with SQL?

The source workbook is not included in this public repository. This README documents the workbook structure, formulas, validation steps, PivotTable analysis, and findings.

---

## Workbook Scope

The workbook contained seven worksheets:

| Worksheet | Purpose | Approximate records |
|---|---|---:|
| `Orders` | Order-level commercial data and calculated analysis fields | 32,313 orders |
| `OrderItems` | Item-level sales data and product lookup | 40,025 items |
| `Products` | Product master table | 4 products |
| `OrderItemsRefund` | Item-level refund transactions | 1,731 refunds |
| `WebsiteSessions` | Session, traffic-source, campaign, and device data | 472,871 sessions |
| `QC_checks` | Row counts, date checks, and duplicate checks | 8 checks |
| `PivotTable` | Directional summaries and written observations | Multiple summaries |

These tables represent different grains:

- `Orders`: one row per order
- `OrderItems`: one row per purchased item
- `OrderItemsRefund`: one row per refund event
- `WebsiteSessions`: one row per website session
- `Products`: one row per product

Recognizing these grains was important because joining tables at the wrong level could duplicate revenue or distort refund calculations.

---

# 1. Orders Sheet

## Original Fields

The order table contained fields including:

- `order_id`
- `created_at`
- `website_session_id`
- `user_id`
- `primary_product_id`
- `items_purchased`
- `price_usd`
- `cogs_usd`

## Calculated Fields

Seven analysis fields were added.

### Month Key

The order date was converted to the first day of its month to support monthly PivotTables.

```excel
=DATE(
    YEAR(orders[[#This Row],[created_at]]),
    MONTH(orders[[#This Row],[created_at]]),
    1
)
```

**Purpose:** create a consistent monthly grouping field.

---

### Revenue

```excel
=orders[[#This Row],[price_usd]]
```

**Purpose:** create a clearly named field for reporting and PivotTables.

---

### Gross Margin

```excel
=orders[[#This Row],[price_usd]]
-orders[[#This Row],[cogs_usd]]
```

**Purpose:** calculate the dollar contribution remaining after cost of goods sold.

The corresponding margin rate is interpreted as:

```text
Gross Margin % = Gross Margin ÷ Revenue
```

---

### Line-item Sum

```excel
=SUMIFS(
    orders[price_usd],
    orders[order_id],
    orders[[#This Row],[order_id]]
)
```

**Purpose:** provide a reconciliation field for the order value.

This check was used to make sure the commercial value assigned to an order was internally consistent.

---

### Mismatch Flag

```excel
=ROUND(
    orders[[#This Row],[price_usd]]
    -orders[[#This Row],[Revenue]],
    2
)<>0
```

**Purpose:** flag records where the order value and analysis revenue field did not reconcile.

A `FALSE` result means the two values match after rounding.

---

### Channel

```excel
=IFERROR(
    INDEX(
        website_sessions[Channel (Cleaned)],
        MATCH(
            orders[[#This Row],[website_session_id]],
            website_sessions[website_session_id],
            0
        )
    ),
    "Not found"
)
```

**Purpose:** attach the cleaned acquisition channel to each order through `website_session_id`.

`INDEX/MATCH` was used instead of manually copying data between sheets.

---

### Device

```excel
=IFERROR(
    INDEX(
        website_sessions[device_type],
        MATCH(
            orders[[#This Row],[website_session_id]],
            website_sessions[website_session_id],
            0
        )
    ),
    "Not found"
)
```

**Purpose:** classify each order as desktop or mobile based on the associated session.

---

# 2. OrderItems Sheet

## Original Fields

The item-level table included:

- `order_item_id`
- `created_at`
- `order_id`
- `product_id`
- `is_primary_item`
- `price_usd`
- `cogs_usd`

## Product Lookup

A product-name field was added with:

```excel
=IFERROR(
    INDEX(
        products[product_name],
        MATCH(
            order_items[[#This Row],[product_id]],
            products[product_id],
            0
        )
    ),
    "Not found"
)
```

**Purpose:** convert product IDs into readable product names for PivotTable analysis.

The four products in the master table were:

1. The Original Mr. Fuzzy
2. The Forever Love Bear
3. The Birthday Sugar Panda
4. The Hudson River Mini bear

---

# 3. WebsiteSessions Sheet

## Main Fields

The session table included:

- `website_session_id`
- `created_at`
- `user_id`
- `is_repeat_session`
- `utm_source`
- `utm_campaign`
- `utm_content`
- `device_type`
- `http_referer`

## Cleaned Channel

A cleaned acquisition-channel field was created:

```excel
=IF(
    website_sessions[[#This Row],[utm_source]]="NULL",
    "direct/type-in",
    website_sessions[[#This Row],[utm_source]]
)
```

**Purpose:** replace missing or null source values with a meaningful business label.

The resulting channel groups were:

- `gsearch`
- `bsearch`
- `socialbook`
- `direct/type-in`

This cleaned field was later looked up into the Orders table.

---

# 4. Quality-Control Checks

The `QC_checks` sheet contained basic controls before analysis.

| Check | Excel logic | Purpose |
|---|---|---|
| Orders row count | `COUNTA(orders[order_id])` | Confirm order-table volume |
| OrderItems row count | `COUNTA(order_items[order_item_id])` | Confirm item-table volume |
| Refund row count | `COUNTA(order_item_refunds[order_item_refund_id])` | Confirm refund-table volume |
| Products row count | `COUNTA(products[product_id])` | Confirm product-master completeness |
| Sessions row count | `COUNTA(website_sessions[website_session_id])` | Confirm session-table volume |
| Earliest order date | `MIN(orders[created_at])` | Check beginning of the available period |
| Latest order date | `MAX(orders[created_at])` | Check end of the available period |
| Duplicate order IDs | `SUMPRODUCT(COUNTIF(... )>1)` | Identify duplicate order identifiers |

## Why These Checks Matter

Before calculating KPIs, the workbook verified that:

- expected tables were populated,
- date coverage was understood,
- order IDs were not unexpectedly duplicated,
- lookup keys could connect orders, sessions, items, and products,
- calculated revenue fields reconciled.

These checks reduce the risk of building analysis on incomplete or duplicated data.

---

# 5. PivotTable Analysis

The `PivotTable` sheet was used to create a directional business view.

## Monthly Trend

The monthly PivotTable summarized:

- revenue
- gross margin
- gross-margin percentage
- monthly share of revenue

The FY2014 monthly pattern showed:

- January revenue: approximately **$56.6K**
- December revenue: approximately **$144.8K**
- monthly gross-margin rate: approximately **62.5%–63.4%**

### Directional Interpretation

Revenue more than doubled during FY2014, while gross-margin percentage remained stable.

This suggested that growth was not being achieved through major margin sacrifice or excessive discounting.

---

## Product Breakdown

The product view compared:

- product revenue
- product gross margin
- gross-margin percentage

### Directional Interpretation

The analysis showed a high level of product concentration, with the flagship product contributing a large portion of sales.

It also showed that the newer products had comparatively strong gross-margin rates, creating an opportunity to diversify the product mix.

The final FY2014 product shares and refund rates were validated in SQL because item sales and refund events sit at different grains.

---

## Device Composition

The device view compared desktop and mobile performance.

### Directional Interpretation

Desktop generated the large majority of revenue, while mobile contributed a much smaller share.

This raised a follow-up question:

> Is mobile underperforming because it receives less traffic, converts less effectively, or both?

The workbook could show revenue composition, but SQL and Power BI were needed for a more precise session-to-order conversion comparison.

---

## Channel Breakdown

The channel view compared:

- revenue
- gross margin
- acquisition source

### Directional Interpretation

Paid search, particularly `gsearch`, was the dominant acquisition source.

This indicated strong channel concentration and motivated further analysis of:

- channel revenue share,
- channel conversion rate,
- revenue per session,
- dependence on one paid-search source.

---

# 6. Main Excel Findings

The workbook produced five useful directional findings.

## 1. Revenue Grew Strongly

FY2014 monthly revenue rose from approximately **$56.6K to $144.8K**.

Growth strengthened toward the final quarter.

## 2. Gross Margin Remained Stable

Monthly gross-margin percentage stayed close to **63%**.

This indicates that the revenue increase did not come from a major decline in unit economics.

## 3. Revenue Was Concentrated by Product

The flagship product was the main commercial driver.

This strength also created product-concentration risk.

## 4. Desktop and Paid Search Dominated

Desktop represented the majority of revenue, and `gsearch` was the leading acquisition channel.

The business therefore appeared dependent on both desktop customers and paid-search traffic.

## 5. Additional Analysis Was Required

Excel was sufficient for directional exploration, but it could not fully answer:

- true product refund rate,
- session-to-order conversion rate,
- the relative contribution of traffic, CVR, and AOV,
- funnel drop-off,
- before-versus-after comparisons.

These questions were moved to the SQL stage.

---

# 7. Limitations of the Excel Analysis

The Excel workbook was intentionally used as an exploration layer rather than the final analytical source of truth.

## Different Table Grains

Orders, items, sessions, and refunds do not represent the same unit of analysis.

A direct raw-table join can produce duplicated rows and overstated totals.

## Refund Rate Requires Cohort Logic

A correct product refund rate requires:

```text
Refunded FY2014 items ÷ FY2014 items sold
```

Refund counts should not simply be joined to item sales without first controlling the grain and date cohort.

## PivotTable Filter State

PivotTable outputs can depend on filters, refresh state, and the selected date field.

For this reason, the final FY2014 KPI values were validated with SQL.

## Conversion Requires Non-converting Sessions

Conversion rate must retain website sessions that did not place orders.

This is more reliably handled in SQL with a `LEFT JOIN` from sessions to orders.

---

# 8. Transition from Excel to SQL

Excel generated hypotheses; SQL validated them.

| Excel observation | SQL validation |
|---|---|
| Revenue rose during FY2014 | Monthly revenue query |
| Gross-margin rate remained stable | Monthly margin query |
| One product dominated sales | Product-performance query |
| Desktop dominated revenue | Device-composition query |
| gsearch dominated acquisition | Channel-breakdown query |
| Growth appeared operationally healthy | Sessions × CVR × AOV decomposition |
| Product quality needed more analysis | True product refund-rate query |

This workflow prevented the project from treating a PivotTable observation as a final conclusion without validation.

---

# 9. Skills Demonstrated

- Spreadsheet data exploration
- Excel Tables and structured references
- `DATE`, `YEAR`, and `MONTH`
- `SUMIFS`
- `INDEX` and `MATCH`
- `IF` and `IFERROR`
- Row-count and duplicate checks
- PivotTable analysis
- Revenue and gross-margin calculation
- Lookup-key validation
- Business interpretation
- Identification of questions requiring SQL

---

## Repository Note

The Excel source workbook is not published in this repository.

This README documents:

- the workbook structure,
- the formulas used,
- the validation process,
- the directional findings,
- the reason the analysis was continued in SQL.

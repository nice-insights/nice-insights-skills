---
name: nice-insights-metrics
description: MUST load before querying any ad, order, order line, cohort, timeseries, or email metrics tools. Covers ecommerce metrics for brands like ad spend, impressions, CAC, sales, refunds, order counts, margins, LTV, email profile and event activity, or any ecommerce related metrics.
metadata:
  author: nice-insights
  version: "1.5"
---

# Nice Insights Metrics

Use these tools to answer analytics questions about advertising performance, sales, order economics, and customer acquisition costs.

> **Data freshness:** Data is imported once daily for the *previous* day. Do not query today's date — results will be incomplete. Yesterday is always the most recent complete date.

## Available Tools

| Tool | Use when the user asks about |
|---|---|
| `list_companies` | Which companies are available (or when company_id is unknown) |
| `query_ad_metrics` | Ad spend, impressions, clicks, CPM, CPC, CTR |
| `query_order_line_metrics` | Product-level sales — revenue, discounts, refunds, units |
| `query_order_metrics` | Order-level costs and margins — shipping costs, fees, contribution margin, CAC |
| `query_customer_cohort_metrics` | Cohort retention/LTV matrices — sales, contribution margin, order counts, per-customer variants, or retention rate per cohort over time |
| `query_timeseries_metrics` | Blended customer acquisition cost (CAC) trends over time |
| `query_email_profile_metrics` | Email list size, email-attributed net sales (excludes refunds), profile conversion rate by status, creation type, or flow/list |
| `query_email_event_metrics` | Email event volume (total and unique profiles) by event name, profile status, creation type, flow/list, or click source |

---

## Step 1: Identify the Company

If `company_id` is already known from context, use it directly.

Otherwise, call `list_companies` to get the list of companies the user has access to. If multiple companies are returned and the question does not clearly indicate which one, ask the user to confirm before proceeding.

---

## Step 2: Confirm Refund Handling (BLOCKING — required before any order query)

**STOP. Before calling `query_order_metrics`, `query_order_line_metrics`, or `query_customer_cohort_metrics`, you MUST ask the user whether to include or exclude refunds. Do NOT proceed with the tool call until they answer.**

Ask: "Should I include or exclude refunds?"

- **Exclude refunds** → `transaction_types: ["Order"]` (typical for marketing/sales activity views)
- **Include refunds** → `transaction_types: ["Order", "Refund"]` (typical for finance/net revenue views)
- **No filter** → omit `transaction_types` entirely (returns everything)

**Note:** If you add `TRANSACTION_TYPE` as a *dimension*, refund rows appear as separate rows rather than being filtered. Use this when the user wants orders and refunds broken out side by side.

Skip this step for `query_ad_metrics`, `query_timeseries_metrics`, `query_email_profile_metrics`, and `query_email_event_metrics` — they have no transaction type concept.

---

## Step 2b: Customer Type Filter (optional)

Use `customer_types` to segment by buyer history. Available on `query_order_line_metrics` and `query_order_metrics`.

- `["New"]` — customers placing their first-ever order
- `["Repeat"]` — customers who have ordered before
- Omit to include all customer types

You can also add `CUSTOMER_TYPE` as a **dimension** to break results out side by side instead of filtering.

---

## Step 2c: Cohort Filters (optional)

Use these filters to narrow results to specific customer acquisition cohorts. Both are optional.

- `first_order_date_range` — available on `query_order_metrics` and `query_customer_cohort_metrics` (**required** on `query_customer_cohort_metrics` — it defines which cohorts to include). Filters to orders where the customer's first-ever order date falls within the given range. Uses the same `{start, end}` format as `date_range`.
- `max_days_since_first_order` — available on `query_order_metrics`, `query_order_line_metrics`, and `query_customer_cohort_metrics`. Filters to orders placed within N days of the customer's first order. Useful for analyzing early customer behavior (e.g., repeat purchases within 30 days).

---

## Step 3: Choose the Right Tool and Metrics

Use the definitions below to infer which metrics to request based on the user's question. `query_ad_metrics` defaults to SPEND, IMPRESSIONS, CLICKS if no metrics are specified. All other tools require explicit metric selection.

### Ad Metrics — `query_ad_metrics`

| Metric | Definition |
|---|---|
| SPEND | Total ad spend in dollars |
| IMPRESSIONS | Total ad impressions served |
| CLICKS | Total ad clicks |
| CPM | Cost per thousand impressions (SPEND / IMPRESSIONS × 1000) |
| CPC | Cost per click (SPEND / CLICKS) |
| CTR | Click-through rate as a percentage (CLICKS / IMPRESSIONS × 100) |

Available channels: Additional Ad Spend, Tatari TV, TikTok, Amazon, AppLovin, Google, Meta.

### Order Line Metrics — `query_order_line_metrics`

Product-level sales data. Use for revenue, discounts, refunds, units, and product mix analysis. `date_range` filters by the column selected in `date_column` (default: ORDER_DATE). Set `date_column` to REVENUE_RECOGNITION_DATE or FIRST_ORDER_DATE to filter and group by those dates instead.

| Metric | Definition |
|---|---|
| GROSS_SALES | Total revenue before any discounts or refunds |
| TOTAL_DISCOUNT | Total discount amounts applied to orders |
| TOTAL_REFUNDS | Total refunds issued (positive number = dollars returned to customers) |
| NET_SALES | GROSS_SALES − TOTAL_DISCOUNT + SHIPPING_PRICE − TOTAL_REFUNDS |
| SHIPPING_PRICE | Shipping revenue charged to the customer |
| COGS | Cost of goods sold |
| UNIT_COUNT | Units sold (refunded units are excluded) |
| ORDER_COUNT | Distinct orders (excludes refund transactions unless TRANSACTION_TYPE is a dimension) |
| CUSTOMER_COUNT | Distinct customers |
| AVERAGE_ORDER_VALUE | Average net sales per order (context-dependent — see note below) |
| AVERAGE_UNIT_PRICE | Average net sales per unit (context-dependent — see note below) |

### Order Metrics — `query_order_metrics`

Order-level costs and margins. Use for profitability, fee analysis, and acquisition economics. `date_range` filters by order date.

| Metric | Definition |
|---|---|
| CONTRIBUTION_MARGIN | Total Net sales minus CAC, COGS, shipping, credit card fees, and other variable costs.  This is profit contribution before fixed costs |
| ACQUISITION_MARGIN | Order value of new customers minus advertising cost — measures the profitability of acquiring new customers |
| ORDER_COUNT | Distinct orders (context-dependent) |
| CUSTOMER_COUNT | Distinct customers |
| AVERAGE_CONTRIBUTION_MARGIN | Average contribution margin per order (context-dependent) |
| AVERAGE_ACQUISITION_MARGIN | Average acquisition margin per new customer (context-dependent) |

### Customer Cohort Metrics — `query_customer_cohort_metrics`

Pivoted cohort matrix with rows per cohort (first order period) and columns per period since first order. All values are cumulative through the period. Use for retention analysis, LTV curves, and cohort comparisons. Accepts a single measure per query.

| Measure | Definition |
|---|---|
| sales | Cumulative sales for the cohort through each period |
| sales_per_customer | Cumulative sales divided by cohort customer count (LTV) |
| contribution_margin | Cumulative contribution margin for the cohort through each period |
| contribution_margin_per_customer | Cumulative contribution margin divided by cohort customer count |
| order_count | Cumulative number of orders for the cohort through each period |
| orders_per_customer | Cumulative orders divided by cohort customer count |
| retention_rate | Share of the cohort still active through each period |

**Parameters:**

- `cohort_grain`: `week` or `month` (default: month) — time grain for cohort bucketing

**Additional filters** (unique to this tool):

- `first_order_product_name` — filter to customers whose first order included this product name
- `first_order_sku` — filter to customers whose first order included this SKU
- `sales_channels` — filter by sales channel: Amazon, Shopify, TikTok
- `subscription_types` — filter by subscription type: First, Recurring, Unknown

### Timeseries Metrics — `query_timeseries_metrics`

Blended CAC over time. Use for trend analysis of customer acquisition efficiency. `date_range` is **required**.

| Metric | Definition |
|---|---|
| DTC_BLENDED_CAC | DTC ad spend ÷ (Shopify + TikTok new customers) |
| AMZN_BLENDED_CAC | Amazon ad spend ÷ Amazon new customers |
| COMBINED_BLENDED_CAC | (DTC + Amazon ad spend) ÷ all new customers |
| SHOPIFY_BLENDED_CAC | (Meta + Google + Microsoft + Pinterest spend) ÷ Shopify new customers |
| TIKTOK_BLENDED_CAC | TikTok ad spend ÷ TikTok new customers |

### Email Profile Metrics — `query_email_profile_metrics`

Email list and profile-level conversion. Use for list growth, email profile activity, and marketing-attributed sales analysis. Metrics ending in `_MAR` are the *marketing* version of net sales and **exclude refunds** by construction.

| Metric | Definition |
|---|---|
| PROFILE_COUNT | Distinct email profiles |
| ORDER_COUNT | Distinct orders attributed to these profiles |
| TOTAL_NET_SALES_MAR | Marketing-attributed net sales (excludes refunds) |
| AVERAGE_NET_SALES_MAR | Average marketing-attributed net sales per row (group-dependent) |
| CONVERSION_RATE | Share of profiles that placed an order |

**Additional filters** (unique to this tool):

- `first_event_date_range` — filter by the date of the profile's first email event
- `profile_statuses` — filter by profile status
- `profile_creation_types` — filter by how the profile was created
- `profile_creation_flow_or_lists` — filter by originating flow or list name

### Email Event Metrics — `query_email_event_metrics`

Email event activity counts (sends, opens, clicks, etc.). Use for engagement volume and trends.

| Metric | Definition |
|---|---|
| TOTAL_EVENT_COUNT | Total email events recorded |
| UNIQUE_EVENT_COUNT | Distinct profiles generating these events |

**Additional filters** (unique to this tool):

- `event_names` — filter to specific event names
- `profile_statuses`, `profile_creation_types`, `profile_creation_flow_or_lists` — same semantics as in `query_email_profile_metrics`

**Note on context-dependent metrics:** ORDER_COUNT, CUSTOMER_COUNT, and average metrics (AVERAGE_ORDER_VALUE, AVERAGE_CONTRIBUTION_MARGIN, etc.) produce different SQL depending on whether TRANSACTION_TYPE is included as a dimension. When TRANSACTION_TYPE is a dimension, these metrics account for both order and refund rows separately. When it is not, they automatically filter to orders only.

---

## Step 4: Choose Output Mode

Use inline when result sets are relatively small and you need to reason over or summarize the data directly. 

Use s3_csv as the output mode when you expect more than 40 rows of data, or when the resulting data is complicated. Analyze the data with a script on your side for s3_csv output.

---

## Available Dimensions

**Time:** DATE, WEEK, MONTH

**Ad-specific:** CHANNEL, PLATFORM, CAMPAIGN_ID, CAMPAIGN_NAME, AD_GROUP_ID, AD_GROUP_NAME, AD_ID, AD_NAME

**Order/line:** SALES_CHANNEL, CUSTOMER_TYPE, TRANSACTION_TYPE, PRODUCT_ID, PRODUCT_NAME, PRODUCT_VARIANT_ID, PRODUCT_VARIANT_NAME, SKU, IS_SUBSCRIPTION, SUBSCRIPTION_TYPE, ORDER_ID, CUSTOMER_ID

**Cohort analysis (order line only):** DAYS_SINCE_FIRST_ORDER, WEEKS_SINCE_FIRST_ORDER, MONTHS_SINCE_FIRST_ORDER

**Email profile:** PROFILE_CREATED_DATE, PROFILE_STATUS, PROFILE_CREATION_TYPE, PROFILE_CREATION_FLOW_OR_LIST, DAYS_FROM_CREATION_TO_FIRST_ORDER, WEEKS_FROM_CREATION_TO_ORDER_DATE, MONTHS_FROM_CREATION_TO_ORDER_DATE

**Email event:** EVENT_DATE, EVENT_NAME, PROFILE_STATUS, PROFILE_CREATION_TYPE, PROFILE_CREATION_FLOW_OR_LIST, CLICKED_EMAIL_SOURCE

**Cohort period values:** For DAYS/WEEKS/MONTHS_SINCE_FIRST_ORDER, `0` always represents the first order date itself. `1` is the 1st full day/week/month after that, `2` is the 2nd, and so on.

**Note:** For cohort retention/LTV analysis, prefer `query_customer_cohort_metrics` which returns a pre-pivoted matrix directly.

---

## Common Query Patterns

**Ad spend by channel for a date range:**
```
query_ad_metrics(query={
  filters: { company_id: 123, date_range: { start: "2025-01-01", end: "2025-01-31" } },
  dimensions: ["CHANNEL"],
  metrics: ["SPEND", "IMPRESSIONS", "CLICKS", "CPM", "CTR"]
})
```

**Net sales by product (excluding refunds):**
```
query_order_line_metrics(query={
  filters: {
    company_id: 123,
    date_range: { start: "2025-01-01", end: "2025-01-31" },
    transaction_types: ["Order"]
  },
  dimensions: ["PRODUCT_NAME"],
  metrics: ["GROSS_SALES", "TOTAL_DISCOUNT", "NET_SALES", "UNIT_COUNT", "ORDER_COUNT"]
})
```

**Monthly contribution margin by sales channel (including refunds):**
```
query_order_metrics(query={
  filters: {
    company_id: 123,
    date_range: { start: "2025-01-01", end: "2025-03-31" },
    transaction_types: ["Order", "Refund"]
  },
  dimensions: ["MONTH", "SALES_CHANNEL"],
  metrics: ["CONTRIBUTION_MARGIN", "ORDER_COUNT"]
})
```

**Monthly cumulative LTV by customer cohort:**
```
query_customer_cohort_metrics(query={
  filters: {
    company_id: 123,
    first_order_date_range: { start: "2025-01-01", end: "2025-03-31" },
    transaction_types: ["Order"]
  },
  measure: "sales_per_customer",
  cohort_grain: "month"
})
```

**Weekly blended CAC trend:**
```
query_timeseries_metrics(query={
  filters: { company_id: 123, date_range: { start: "2025-01-01", end: "2025-03-31" } },
  dimensions: ["WEEK"],
  metrics: ["DTC_BLENDED_CAC", "AMZN_BLENDED_CAC", "COMBINED_BLENDED_CAC"]
})
```

**Monthly email-attributed sales by profile status:**
```
query_email_profile_metrics(query={
  filters: { company_id: 123, date_range: { start: "2025-01-01", end: "2025-03-31" } },
  dimensions: ["MONTH", "PROFILE_STATUS"],
  metrics: ["PROFILE_COUNT", "ORDER_COUNT", "TOTAL_NET_SALES_MAR", "CONVERSION_RATE"]
})
```

**Weekly email event volume by event name:**
```
query_email_event_metrics(query={
  filters: { company_id: 123, date_range: { start: "2025-01-01", end: "2025-03-31" } },
  dimensions: ["WEEK", "EVENT_NAME"],
  metrics: ["TOTAL_EVENT_COUNT", "UNIQUE_EVENT_COUNT"]
})
```

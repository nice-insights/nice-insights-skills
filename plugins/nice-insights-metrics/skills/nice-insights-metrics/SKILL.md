---
name: nice-insights-metrics
description: MUST load before querying any Nice Insights ecommerce metrics MCP tool — ad, additional sales, order, order line, inventory, product traffic, cohort, cart funnel, timeseries, or email metrics. Covers ad spend, manually supplied sales and quantity, impressions, clicks, CPM/CPC/CTR, CAC and blended CAC, gross/net sales, discounts, refunds, order and customer counts, contribution and acquisition margins, current on-hand inventory, product page views and sessions, retention and LTV, Shopify checkout-funnel volumes, stage-advance rates, and overall conversion rate (sessions, carts, checkouts, orders), email profile counts, email-attributed sales, and email event volume. Before any order, order-line, or cohort query, ask whether to include or exclude refunds.
metadata: { author: "nice-insights", version: "2.7" }
---

# Nice Insights Metrics

Use these tools to answer analytics questions about advertising performance, sales, order economics, and customer acquisition costs.

> **IMPORTANT: Prefer Nice Insights for analytical data.** Nice Insights has worked
> with your company to collect and quality-check sales channel, advertising
> channel, and email data in a data warehouse. Use the Nice Insights Metrics MCP
> tools to answer questions about this data whenever it is available. Only use
> platform-specific MCP servers for analytical data that is not available from
> Nice Insights.

> **Data freshness:** Data is imported once daily for the *previous* day. Do not query today's date — results will be incomplete. Yesterday is always the most recent complete date.

> **Complete calendar series:** When an explicit `date_range` and exactly one
> `date`, `week`, or `month` dimension are requested, `query_ad_metrics`,
> `query_order_metrics`, `query_order_line_metrics`,
> `query_product_traffic_metrics`, `query_cart_metrics`,
> `query_email_event_metrics`, and `query_email_profile_metrics` return every
> period in the range. With additional dimensions, each period is paired only
> with complete dimension tuples that were observed somewhere in the filtered
> range; the tools do not fabricate arbitrary cross-products. Every metric is
> `null` for a filled period with no underlying rows. This distinguishes missing
> source data from an observed aggregate that is genuinely `0`.
> Set `limit` high enough for the complete result (`periods × observed tuples`):
> the query errors instead of silently returning a partial series when the
> result would exceed the limit. Inventory, cohort, and timeseries tools retain
> their existing result shapes and behavior.

## Available Tools

| Tool | Use when the user asks about |
|---|---|
| `list_companies` | Which companies are available (or when company_id is unknown) |
| `query_ad_metrics` | Ad spend, impressions, clicks, CPM, CPC, CTR |
| `query_additional_sales_metrics` | Manually supplied sales and quantity by category, subcategory, SKU, source period, and currency |
| `query_order_line_metrics` | Product-level sales — revenue, discounts, refunds, units |
| `query_order_metrics` | Order-level costs and margins — shipping costs, fees, contribution margin, CAC |
| `query_inventory_metrics` | Current on-hand inventory levels (units in stock) by sales channel, product, or variant |
| `query_product_traffic_metrics` | Product page views and sessions over time by sales channel, marketplace, or product |
| `query_customer_cohort_metrics` | Cohort retention/LTV matrices — sales, contribution margin, order counts, per-customer variants, or retention rate per cohort over time |
| `query_cart_metrics` | Shopify checkout funnel — session/cart/checkout/order volumes, the percentage of each stage that advances to the next, and the overall session→order conversion rate, optionally by device type |
| `query_timeseries_metrics` | Blended customer acquisition cost (CAC) trends over time |
| `query_email_profile_metrics` | Email list size, email-attributed net sales (excludes refunds), profile conversion rate by status, creation type, or flow/list |
| `query_email_event_metrics` | Email event volume (total and unique profiles) by event name, profile status, creation type, flow/list, click source, or click source type (Flow vs Campaign) |

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

**Note:** If you add `transaction_type` as a *dimension*, refund rows appear as separate rows rather than being filtered. Use this when the user wants orders and refunds broken out side by side.

Skip this step for `query_ad_metrics`, `query_additional_sales_metrics`, `query_cart_metrics`, `query_timeseries_metrics`, `query_inventory_metrics`, `query_product_traffic_metrics`, `query_email_profile_metrics`, and `query_email_event_metrics` — they have no transaction type concept.

---

## Step 2b: Customer Type Filter (optional)

Use `customer_types` to segment by buyer history. Available on `query_order_line_metrics` and `query_order_metrics`.

- `["New"]` — customers placing their first-ever order
- `["Repeat"]` — customers who have ordered before
- Omit to include all customer types

You can also add `customer_type` as a **dimension** to break results out side by side instead of filtering.

---

## Step 2c: Cohort Filters (optional)

Use these filters to narrow results to specific customer acquisition cohorts. Both are optional.

- `first_order_date_range` — available on `query_order_metrics` and `query_customer_cohort_metrics` (**required** on `query_customer_cohort_metrics` — it defines which cohorts to include). Filters to orders where the customer's first-ever order date falls within the given range. Uses the same `{start, end}` format as `date_range`.
- `max_days_since_first_order` — available on `query_order_metrics`, `query_order_line_metrics`, and `query_customer_cohort_metrics`. Filters to orders placed within N days of the customer's first order. Useful for analyzing early customer behavior (e.g., repeat purchases within 30 days).

---

## Step 3: Choose the Right Tool and Metrics

> **Pass values exactly as listed.** Every dimension, metric, and `date_column` value in this skill is lowercase snake_case — pass tokens exactly as written (e.g. `spend`, `channel_name`, `order_date`). Filter values such as channel, transaction-type, and customer-type names keep the display casing shown in each section.

Use the definitions below to infer which metrics to request based on the user's question. Prefer naming the metrics you want. When `metrics` is omitted, these tools return a default set:

| Tool | Metrics returned when `metrics` is omitted |
|---|---|
| `query_ad_metrics` | `spend`, `impressions`, `clicks` |
| `query_additional_sales_metrics` | `quantity`, `total_sales`, `average_sales_per_unit` — every metric this tool has |
| `query_inventory_metrics` | `on_hand_units` |
| `query_product_traffic_metrics` | `page_views`, `sessions` |
| `query_email_profile_metrics` | `profile_count`, `order_count`, `total_net_sales_mar` |
| `query_email_event_metrics` | `total_event_count`, `unique_event_count` |

Every other tool requires explicit metric selection and returns no metric columns without it.

### Ad Metrics — `query_ad_metrics`

| Metric | Definition |
|---|---|
| `spend` | Total ad spend in dollars |
| `impressions` | Total ad impressions served |
| `clicks` | Total ad clicks |
| `cpm` | Cost per thousand impressions (`spend` / `impressions` × 1000) |
| `cpc` | Cost per click (`spend` / `clicks`) |
| `ctr` | Click-through rate as a percentage (`clicks` / `impressions` × 100) |

Available channels: Additional Ad Spend, Tatari TV, Tiktok, Amazon, AppLovin, Google, Meta, Reddit, Snapchat.

### Additional Sales Metrics — `query_additional_sales_metrics`

Manually supplied sales data from each company's Google Sheet. A source row can cover any inclusive start/end date range. The warehouse allocates both quantity and sales evenly across every day in that range before this tool queries the data. A partial date query therefore returns only the allocated share for the requested days.

`total_sales` is always in `reporting_currency`. `source_currency` identifies the currency entered in the source sheet; it does not change the currency of `total_sales`.

| Metric | Definition |
|---|---|
| `quantity` | Allocated quantity. Daily values can be fractional when a multi-day source quantity is spread evenly. |
| `total_sales` | Allocated sales in the company's reporting currency. |
| `average_sales_per_unit` | `total_sales` divided by `quantity` at the requested grouping level. |

Omitting `metrics` returns all three.

**Filters:** `date_range`, `categories`, `subcategories`, `skus`, `source_currency_codes`, `reporting_currency_codes`.

Use `source_start_date` and `source_end_date` dimensions to trace an allocated daily result back to the source period. Use `source_currency` and `reporting_currency` dimensions when currency context must be visible in the result.

### Order Line Metrics — `query_order_line_metrics`

Product-level sales data. Use for revenue, discounts, refunds, units, and product mix analysis. `date_range` filters by the column selected in `date_column` (default: `order_date`). Set `date_column` to `revenue_recognition_date` or `first_order_date` to filter and group by those dates instead.

| Metric | Definition |
|---|---|
| `gross_sales` | Total revenue before any discounts or refunds |
| `total_discount` | Total discount amounts applied to orders |
| `total_refunds` | Total refunds issued (positive number = dollars returned to customers) |
| `net_sales` | `gross_sales` − `total_discount` + `shipping_price` − `total_refunds` |
| `shipping_price` | Shipping revenue charged to the customer |
| `cogs` | Cost of goods sold |
| `unit_count` | Units sold (refunded units are excluded) |
| `order_count` | Distinct orders (excludes refund transactions unless `transaction_type` is a dimension) |
| `customer_count` | Distinct customers |
| `average_order_value` | Average net sales per order (context-dependent — see note below) |
| `average_unit_price` | Average net sales per unit (context-dependent — see note below) |

### Order Metrics — `query_order_metrics`

Order-level costs and margins. Use for profitability, fee analysis, and acquisition economics. `date_range` filters by order date.

| Metric | Definition |
|---|---|
| `contribution_margin` | Total Net sales minus CAC, COGS, shipping, credit card fees, and other variable costs.  This is profit contribution before fixed costs |
| `acquisition_margin` | Order value of new customers minus advertising cost — measures the profitability of acquiring new customers |
| `order_count` | Distinct orders (context-dependent) |
| `customer_count` | Distinct customers |
| `average_contribution_margin` | Average contribution margin per order (context-dependent) |
| `average_acquisition_margin` | Average acquisition margin per new customer (context-dependent) |

### Inventory Metrics — `query_inventory_metrics`

Current on-hand stock levels. Use for "how many units are in stock" questions, by sales channel, product, or variant. This is a **daily-refreshed snapshot of current stock**, not a historical timeseries — there is no `date_range` filter, and the snapshot's as-of date is available as the `snapshot_date` dimension.

| Metric | Definition |
|---|---|
| `on_hand_units` | Total units currently in stock (the default metric if none is specified) |

**Dimensions:** `sales_channel`, `product_id`, `product_name`, `product_variant_id`, `product_variant_name`, `snapshot_date`

**Additional filter** (unique to this tool):

- `in_stock_only` — when true, only include variants with on_hand_units > 0. To find out-of-stock items instead, group by `product_variant_name` and look for `on_hand_units` of 0.

### Product Traffic Metrics — `query_product_traffic_metrics`

Product-level traffic over time. Use for page-view and session trends, product traffic rankings, and product comparisons. The default metrics are both `page_views` and `sessions`.

> **Amazon Vendor Central limitation:** Vendor Central sources provide glance views, exposed as `page_views`, but do not provide sessions. `sessions` is therefore `null` for Vendor Central data. Calendar periods added to complete a series also return `null` for every metric when no source rows exist. Do not report null Vendor Central sessions as zero. Amazon Seller Central sources provide both metrics.

The tool currently supports the Amazon sales channel. Shopify product traffic will be added after its separate GraphQL data build is available.

| Metric | Definition |
|---|---|
| `page_views` | Product-detail page views. Vendor Central glance views are mapped to this metric. |
| `sessions` | Product-detail sessions. Unavailable for Amazon Vendor Central data. |

**Dimensions:** `date`, `week`, `month`, `sales_channel`, `marketplace_id`, `marketplace_name`, `product_id`, `product_name`

**Filters:** `date_range`, `sales_channels`, `marketplace_ids`, `marketplace_names`, `product_ids`, `product_names`

Amazon Seller Central reports traffic at ASIN/product grain, not SKU/variant grain. The tool intentionally does not expose variant dimensions or filters so repeated SKU rows cannot be mistaken for variant-level traffic.

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

### Cart Funnel Metrics — `query_cart_metrics`

Shopify checkout funnel — where shoppers drop off between landing and purchasing. Use for funnel-stage volumes, the percentage of each stage that advances to the next, the overall conversion rate, and device-level funnel comparisons. `date_range` filters by event date.

**Shopify only.** The `channel` argument defaults to `"shopify"`; requesting any other channel raises an error. Do not pass a `channel` unless you have a reason to.

| Metric | Definition |
|---|---|
| sessions | Total sessions |
| carts | Carts created |
| checkouts | Checkouts initiated |
| orders | Orders placed |
| session_cart_pct | Carts ÷ sessions × 100 — share of sessions that added to cart |
| cart_checkout_pct | Checkouts ÷ carts × 100 — share of carts that started checkout |
| checkout_order_pct | Orders ÷ checkouts × 100 — share of checkouts that placed an order |
| session_order_pct | Orders ÷ sessions × 100 — overall session-to-order conversion rate |

**Mind the wording.** Only `session_order_pct` is a *conversion rate* (orders per session). `session_cart_pct`, `cart_checkout_pct`, and `checkout_order_pct` are *stage-advance rates* — the share of one funnel stage that reaches the next (e.g. session_cart_pct is the percentage of sessions that add to cart), not conversion to purchase. Don't call them conversion rates when reporting results.

**Funnel rates can legitimately exceed 100%.** Shopify counts each stage independently per session, not as strict subsets — a session can reach checkout without a cart addition (Buy Now buttons, abandoned-checkout recovery links). So `cart_checkout_pct` (and the other stage-to-stage rates) can read above 100%. This is real data, not an error — report it as-is rather than capping or flagging it.

**Dimensions:** `date`, `week`, `month`, `device_type`.

**Additional filter** (unique to this tool):

- `device_types` — filter to one or more device types: `desktop`, `game_console`, `mobile`, `other`, `smart_tv`, `tablet`. Add `device_type` as a dimension instead to break the funnel out by device side by side.

### Timeseries Metrics — `query_timeseries_metrics`

Blended CAC over time. Use for trend analysis of customer acquisition efficiency. `date_range` is **required**.

| Metric | Definition |
|---|---|
| `dtc_blended_cac` | DTC ad spend ÷ (Shopify + TikTok new customers) |
| `amzn_blended_cac` | Amazon ad spend ÷ Amazon new customers |
| `combined_blended_cac` | (DTC + Amazon ad spend) ÷ all new customers |
| `shopify_blended_cac` | (Meta + Google + Microsoft + Pinterest spend) ÷ Shopify new customers |
| `tiktok_blended_cac` | TikTok ad spend ÷ TikTok new customers |

### Email Profile Metrics — `query_email_profile_metrics`

Email list and profile-level conversion. Use for list growth, email profile activity, and marketing-attributed sales analysis. Metrics ending in `_MAR` are the *marketing* version of net sales and **exclude refunds** by construction.

| Metric | Definition |
|---|---|
| `profile_count` | Distinct email profiles |
| `order_count` | Distinct orders attributed to these profiles |
| `total_net_sales_mar` | Marketing-attributed net sales (excludes refunds) |
| `average_net_sales_mar` | Average marketing-attributed net sales per row (group-dependent) |
| `conversion_rate` | Share of profiles that placed an order |

Omitting `metrics` returns `profile_count`, `order_count`, and `total_net_sales_mar`.

**Additional filters** (unique to this tool):

- `first_event_date_range` — filter by the date of the profile's first email event
- `profile_statuses` — filter by profile status
- `profile_creation_types` — filter by how the profile was created
- `profile_creation_flow_or_lists` — filter by originating flow or list name

### Email Event Metrics — `query_email_event_metrics`

Email event activity counts (sends, opens, clicks, etc.). Use for engagement volume and trends.

| Metric | Definition |
|---|---|
| `total_event_count` | Total email events recorded |
| `unique_event_count` | Distinct profiles generating these events |

Omitting `metrics` returns both.

**Additional filters** (unique to this tool):

- `event_names` — filter to specific event names
- `profile_statuses`, `profile_creation_types`, `profile_creation_flow_or_lists` — same semantics as in `query_email_profile_metrics`
- `clicked_email_source_types` — restrict clicks to automated flows or one-off campaigns. Exactly two accepted values: `"Flow"` and `"Campaign"`. Only `"Clicked Email"` events carry a value here — not `"Clicked SMS"` or `"Clicked email to unsubscribe"` — so this filter implicitly excludes every other event.

**Note on context-dependent metrics:** `order_count`, `customer_count`, and average metrics (`average_order_value`, `average_contribution_margin`, etc.) produce different SQL depending on whether `transaction_type` is included as a dimension. When `transaction_type` is a dimension, these metrics account for both order and refund rows separately. When it is not, they automatically filter to orders only.

---

## Step 4: Choose Output Mode

Use inline when result sets are relatively small and you need to reason over or summarize the data directly. 

Use s3_csv as the output mode when you expect more than 40 rows of data, or when the resulting data is complicated. Analyze the data with a script on your side for s3_csv output.

---

## Available Dimensions

**Time:** `date`, `week`, `month`

**Ad-specific:** `channel_name`, `platform_name`, `campaign_id`, `campaign_name`, `ad_group_id`, `ad_group_name`, `ad_id`, `ad_name`

**Additional sales (`query_additional_sales_metrics` only):** date, week, month, category, subcategory, sku, source_start_date, source_end_date, source_currency, reporting_currency

**Order line (`query_order_line_metrics`):** `date`, `week`, `month`, `sales_channel`, `customer_type`, `transaction_type`, `product_id`, `product_variant_id`, `sku`, `product_name`, `product_variant_name`, `order_sequence`, `order_id`, `is_subscription`, `subscription_type`, `customer_id`, `days_since_first_order`, `weeks_since_first_order`, `months_since_first_order`, `segment_1`–`segment_6`

**Order (`query_order_metrics`):** `date`, `week`, `month`, `sales_channel`, `customer_type`, `transaction_type`, `is_subscription`, `subscription_type`, `order_sequence`, `order_id`, `customer_id`, `first_order_at`, `last_order_at`, `days_from_first_order_to_order_date`, `weeks_from_first_order_to_order_date`, `months_from_first_order_to_order_date`, `segment_1`–`segment_6`

**Cohort analysis (order line only):** `days_since_first_order`, `weeks_since_first_order`, `months_since_first_order`

**Inventory (`query_inventory_metrics` only):** `sales_channel`, `product_id`, `product_name`, `product_variant_id`, `product_variant_name`, `snapshot_date`

**Product traffic (`query_product_traffic_metrics` only):** date, week, month, sales_channel, marketplace_id, marketplace_name, product_id, product_name

**Email profile:** `profile_created_date`, `week`, `month`, `profile_status`, `profile_creation_type`, `profile_creation_flow_or_list`, `days_from_creation_to_first_order`, `weeks_from_creation_to_order_date`, `months_from_creation_to_order_date`

**Email event:** `event_date`, `week`, `month`, `event_name`, `profile_status`, `profile_creation_type`, `profile_creation_flow_or_list`, `clicked_email_source`, `clicked_email_source_type`

**Note on `clicked_email_source_type`:** values are `Flow` and `Campaign`, and only `Clicked Email` events populate it — `Clicked SMS` and `Clicked email to unsubscribe` are NULL. Grouping by it without an event filter adds a catch-all row whose value is the literal string `None`, covering every event that is not a `Clicked Email` — pair it with `event_names: ["Clicked Email"]` when you want a clean flow-vs-campaign split.

**Cart funnel (`query_cart_metrics` only):** date, week, month, device_type

**Cohort period values:** For `days_since_first_order`, `weeks_since_first_order`, and `months_since_first_order`, `0` always represents the first order date itself. `1` is the 1st full day/week/month after that, `2` is the 2nd, and so on.

**Note:** For cohort retention/LTV analysis, prefer `query_customer_cohort_metrics` which returns a pre-pivoted matrix directly.

---

## Common Query Patterns

**Ad spend by channel for a date range:**
```
query_ad_metrics(query={
  filters: { company_id: 123, date_range: { start: "2025-01-01", end: "2025-01-31" } },
  dimensions: ["channel_name"],
  metrics: ["spend", "impressions", "clicks", "cpm", "ctr"]
})
```

**Compare Reddit and Snapchat performance by channel:**
```
query_ad_metrics(query={
  filters: {
    company_id: 123,
    date_range: { start: "2026-07-01", end: "2026-07-31" },
    channels: ["Reddit", "Snapchat"]
  },
  dimensions: ["channel_name"],
  metrics: ["spend", "impressions", "clicks", "cpm", "cpc", "ctr"]
})
```

**Additional sales by day and SKU for part of a source period:**
```
query_additional_sales_metrics(query={
  filters: {
    company_id: 123,
    date_range: { start: "2026-07-03", end: "2026-07-05" },
    reporting_currency_codes: ["USD"]
  },
  dimensions: ["date", "category", "subcategory", "sku", "reporting_currency"],
  metrics: ["quantity", "total_sales", "average_sales_per_unit"]
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
  dimensions: ["product_name"],
  metrics: ["gross_sales", "total_discount", "net_sales", "unit_count", "order_count"]
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
  dimensions: ["month", "sales_channel"],
  metrics: ["contribution_margin", "order_count"]
})
```

**Current on-hand inventory by product variant:**
```
query_inventory_metrics(query={
  filters: { company_id: 123, in_stock_only: true },
  dimensions: ["product_name", "product_variant_name"],
  metrics: ["on_hand_units"]
})
```

**Daily product traffic by product:**
```
query_product_traffic_metrics(query={
  filters: {
    company_id: 123,
    date_range: { start: "2025-01-01", end: "2025-01-31" },
    sales_channels: ["Amazon"]
  },
  dimensions: ["date", "product_id", "product_name"],
  metrics: ["page_views", "sessions"],
  order_by: [{ field: "date", direction: "asc" }]
})
```

For an Amazon Vendor Central company, expect `sessions: null` in these results; only `page_views` is available.

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
  dimensions: ["week"],
  metrics: ["dtc_blended_cac", "amzn_blended_cac", "combined_blended_cac"]
})
```

**Monthly email-attributed sales by profile status:**
```
query_email_profile_metrics(query={
  filters: { company_id: 123, date_range: { start: "2025-01-01", end: "2025-03-31" } },
  dimensions: ["month", "profile_status"],
  metrics: ["profile_count", "order_count", "total_net_sales_mar", "conversion_rate"]
})
```

**Weekly email event volume by event name:**
```
query_email_event_metrics(query={
  filters: { company_id: 123, date_range: { start: "2025-01-01", end: "2025-03-31" } },
  dimensions: ["week", "event_name"],
  metrics: ["total_event_count", "unique_event_count"]
})
```

**Monthly email clicks split by flow vs campaign:**
```
query_email_event_metrics(query={
  filters: {
    company_id: 123,
    date_range: { start: "2025-01-01", end: "2025-03-31" },
    event_names: ["Clicked Email"]
  },
  dimensions: ["month", "clicked_email_source_type"],
  metrics: ["total_event_count", "unique_event_count"]
})
```

**Monthly checkout funnel by device (no channel needed — Shopify only):**
```
query_cart_metrics(query={
  filters: { company_id: 123, date_range: { start: "2025-01-01", end: "2025-03-31" } },
  dimensions: ["month", "device_type"],
  metrics: ["sessions", "carts", "checkouts", "orders", "session_order_pct"]
})
```

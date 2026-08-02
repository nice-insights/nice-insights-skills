---
name: nice-insights-financials
description: MUST load before querying the Nice Insights monthly financial metrics MCP tools. Covers scenario-safe Actuals, budget, and forecast aggregations and pairwise comparisons across financial statement and account hierarchy fields.
metadata: { author: "nice-insights", version: "1.4" }
---

# Nice Insights Financials

Use `list_companies` to discover authorized companies. Use `query_monthly_financial_metrics` for monthly financial-statement aggregations and `compare_monthly_financial_scenarios` for server-calculated pairwise scenario comparisons. Access requires the OAuth scope `read:financials`, membership in the Nice Insights `FinancialsAccess` group, and access to the requested company.

> **Data freshness:** This source is updated by the production financial pipeline. It may lag current accounting activity; state the latest returned period and do not imply that an incomplete current month is closed.

## Scenario safety (required)

Financial rows can overlap across Actuals, budgets, and forecasts. Every query must use exactly one of these safe shapes:

1. Filter to one exact pair with both `scenario_type` and `scenario_name`.
2. Omit both scenario filters and include both `scenario_type` and `scenario_name` in `dimensions`.

Never omit one half of the filter pair. Never aggregate multiple scenarios without both scenario dimensions. The tool rejects unsafe requests rather than returning mixed totals.

`scenario_type` accepts `Actuals` or `Plan`. `scenario_name` identifies the specific scenario, such as `Actuals`, `Budget`, or `F1`.

For a pairwise comparison, provide two different exact scenario references as
`base_scenario` and `comparison_scenario`. Any pair is valid, including Actuals
versus Budget, F1 versus Budget, or F2 versus F1. Do not include scenario fields
in `dimensions` for comparison requests.

## Input fields

`filters.company_id` is always required. Call `list_companies` on this financials
server and use a returned `company_id`. Never invent, guess, or hard-code a
company ID. The examples below use `$company_id` for that authorized integer;
substitute the integer before calling the tool. Financial data may not cover
every company `list_companies` returns: a zero-row result with
`truncated: false` usually means no financial data is loaded for that company —
report that to the user rather than presenting it as a $0 actual.

Optional filters:

- `period_range`: `{start, end}` dates. Bounds are month-inclusive, so any date within a month includes that entire month.
- `scenario_type` and `scenario_name`: singular exact-scenario pair; supply both or neither.
- `statement_types`
- `account_line_items`
- `child_account_categories`
- `sub_parent_account_categories`
- `parent_account_categories`

Passing any list filter as an empty list is the same as omitting it (no
filter applied), not "match nothing".

For `compare_monthly_financial_scenarios`, the same filters are available except
the scenario filter pair. Instead, provide these required top-level objects:

- `base_scenario`: `{scenario_type, scenario_name}`; this is the percentage denominator.
- `comparison_scenario`: `{scenario_type, scenario_name}`; variances are comparison minus base.

Available dimensions:

- `period`: first day of each month, recommended for chronological output
- `year`, `month`
- `scenario_type`, `scenario_name`
- `statement_type`
- `account_line_item`
- `child_account_category`
- `sub_parent_account_category`
- `parent_account_category`

The only metric is `amount`, defined as the rounded sum of source amounts. It is the default when `metrics` is omitted. Missing account hierarchy values remain `null`.

Comparison results return `base_amount`, `comparison_amount`, `variance_amount`,
and `variance_pct`. The percentage is
`(comparison_amount - base_amount) / base_amount` as a fraction (`0.15` means
15% above the base), and is `null` when the base is zero or either side is
missing. `comparison_status` is `matched`, `base_only`, or `comparison_only`;
missing scenario rows remain `null` and are never silently converted to zero.

Current `statement_type` values are `Income` and `Expense`; some rows have no
statement type and return `null`. Omit `statement_types` to include those rows.

Use `output_mode: "inline"` for direct analysis. Use `output_mode: "s3_csv"` for larger result sets; the response includes a signed CSV URL and five sample rows. `limit` defaults to 100 and cannot exceed 1,000.

Every response includes `truncated`. When it is `true`, the row limit cut off
results (in both inline rows and the S3 CSV) — raise `limit`, narrow filters,
or split the request into smaller queries before summing totals or treating
the output as a complete statement.

## Common query patterns

**Monthly Actuals by statement and line item:**

```text
query_monthly_financial_metrics(query={
  filters: {
    company_id: $company_id,
    period_range: { start: "2026-01-15", end: "2026-03-02" },
    scenario_type: "Actuals",
    scenario_name: "Actuals"
  },
  dimensions: ["period", "statement_type", "account_line_item"],
  metrics: ["amount"]
})
```

**Inspect all scenarios without mixing them:**

```text
query_monthly_financial_metrics(query={
  filters: { company_id: $company_id, statement_types: ["Income", "Expense"] },
  dimensions: ["period", "scenario_type", "scenario_name", "parent_account_category"],
  metrics: ["amount"]
})
```

**Compare F1 against Budget by month and parent category:**

```text
compare_monthly_financial_scenarios(query={
  filters: {
    company_id: $company_id,
    period_range: { start: "2026-01-01", end: "2026-12-31" }
  },
  base_scenario: { scenario_type: "Plan", scenario_name: "Budget" },
  comparison_scenario: { scenario_type: "Plan", scenario_name: "F1" },
  dimensions: ["period", "parent_account_category"],
  metrics: ["amount"]
})
```

**Export a named plan by account hierarchy:**

```text
query_monthly_financial_metrics(query={
  filters: {
    company_id: $company_id,
    scenario_type: "Plan",
    scenario_name: "Budget"
  },
  dimensions: ["period", "parent_account_category", "sub_parent_account_category", "child_account_category", "account_line_item"],
  output_mode: "s3_csv",
  limit: 1000
})
```

---
name: nice-insights-financials
description: MUST load before querying the Nice Insights monthly financial metrics MCP tools. Covers scenario-safe Actuals, budget, and forecast aggregations and pairwise comparisons across financial statement and account hierarchy fields.
metadata: { author: "nice-insights", version: "1.6" }
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
- `parent_account_category`
- `line_item_type`

The only metric is `amount`, defined as the rounded sum of source amounts. It is the default when `metrics` is omitted. `parent_account_category` is the single account hierarchy level; a null dimension value is preserved as `null` and never converted to zero.

## Detail lines and stated totals (`line_item_type`)

Rows come in two classes, and **summing both together double-counts.**

- `Line Item` — a detail line. Every company has these, and for most companies they are all there is.
- `Calculation` — a total the source financial statement states **itself**: `Net Income`, `Net Sales`, `Cost of Sales`, `Reported EBITDA`, `Management Reported EBITDA`, `Operating Income/Loss`, `Amortization`, `Other (Income)/Expense`, `Jupiter Project Expense`. These are the company's own published figures, not anything computed here.

**Queries return `Line Item` rows only unless you ask otherwise.** That default is deliberate: a `Calculation` row is a total of the very detail lines beside it, so including it in an unfiltered `SUM(amount)` counts a section twice.

Pass `line_item_types` to change it. `["Calculation"]` returns the stated totals **instead of** detail lines — it replaces the default rather than adding to it:

```text
filters: { ..., line_item_types: ["Calculation"], account_line_items: ["Net Income"] }
```

Only some companies publish `Calculation` rows — those whose detail sections do not sum to the Net Income their own statement states, so the stated figure is the only correct one. For every other company, a `line_item_types: ["Calculation"]` query returns no rows, and that is not an error: for those companies the sections **do** add up, so compute the figure yourself with the formula in the next section. Never present a computed total as the company's Net Income when a stated one is available.

`Calculation` rows have a `null` `parent_account_category` — they belong to no single section. **Requesting both classes at once requires `line_item_type` in `dimensions`**, so the two never land in the same bucket. This is enforced: a mixed-class query without that dimension is rejected rather than answered, the same way a query without an exact scenario must group by `scenario_type` and `scenario_name`.

## Computing Net Income when it is not stated

**Expenses are stored as positive numbers.** `amount` is exactly the figure the source statement prints, and the statement prints a $469,258 cost as `469,258`, not `-469,258`. The sign is not in the data — it is carried by the **category**: `Net Sales` is money in, every other `parent_account_category` is money out.

So an unfiltered `SUM(amount)` **adds revenue and expenses together**. It is not Net Income, not an approximation of it, and it lands in a plausible-looking range, so nothing about the result will look wrong. Never report one as a bottom line.

To get Net Income for a company that states none, group by category and subtract:

```text
Net Income = SUM(Net Sales) - SUM(every other parent_account_category)
```

excluding `EBITDA Adjustments`, which the statement shows for reference and does not carry into Net Income.

```text
query_monthly_financial_metrics(query={
  filters: {
    company_id: $company_id,
    period_range: { start: "2025-01-01", end: "2025-01-31" },
    scenario_type: "Actuals",
    scenario_name: "Actuals"
  },
  dimensions: ["period", "parent_account_category"],
  metrics: ["amount"]
})
```

Worked example — company 2, 2025-01. Net Sales `1,348,281`; Selling General and Administrative `610,154`; Cost of Sales `469,258`; Depreciation Total `499`; every other category `0`.

- Correct: `1,348,281 - 610,154 - 469,258 - 499` = **268,370**
- Summing the detail lines instead: `2,428,192` — roughly 9x too high, and the error is silent.

The same rule applies to any cross-category figure (both EBITDAs, Operating Income): it must be built from category subtotals with explicit signs, never from a single `SUM`.

**Within one section, summing is correct and is the intended use.** `SUM(amount)` over a single `parent_account_category` reproduces the subtotal the statement prints for that section, exactly. The rule above is only about figures that span categories.


Comparison results return `base_amount`, `comparison_amount`, `variance_amount`,
and `variance_pct`. The percentage is
`(comparison_amount - base_amount) / base_amount` as a fraction (`0.15` means
15% above the base), and is `null` when the base is zero or either side is
missing. `comparison_status` is `matched`, `base_only`, or `comparison_only`;
missing scenario rows remain `null` and are never silently converted to zero.

Current `statement_type` values are `Income` and `Expense`, and every reporting line is
classified today, so omitting `statement_types` returns all of them. The column is
nullable, so an unclassified line would return `null` and be excluded by any
`statement_types` filter.

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

**Read a company's stated Net Income (only for companies that publish one):**

```text
query_monthly_financial_metrics(query={
  filters: {
    company_id: $company_id,
    period_range: { start: "2026-01-01", end: "2026-12-31" },
    scenario_type: "Actuals",
    scenario_name: "Actuals",
    line_item_types: ["Calculation"],
    account_line_items: ["Net Income"]
  },
  dimensions: ["period", "account_line_item"],
  metrics: ["amount"]
})
```

An empty result means this company states no such row. Compute it with the formula above — do **not** fall back to an unfiltered `SUM(amount)`.

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

**Export a named plan by category and line item:**

```text
query_monthly_financial_metrics(query={
  filters: {
    company_id: $company_id,
    scenario_type: "Plan",
    scenario_name: "Budget"
  },
  dimensions: ["period", "parent_account_category", "account_line_item"],
  output_mode: "s3_csv",
  limit: 1000
})
```

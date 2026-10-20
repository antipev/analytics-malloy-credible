---
name: gotchas-queries
description: Common Malloy query and view mistakes. Read BEFORE writing views, queries, or notebooks. Covers chart constraints, aggregate filters, joined field aliasing, method syntax, and time truncation vs extraction.
---

# Query & View Gotchas

> **Read this before writing views or queries.** These patterns cause most query errors.

## Charts — ONE Aggregate Per View

Charts render only the **first** aggregate. Use exactly one aggregate per `# bar_chart` / `# line_chart` view.

```malloy
// WRONG — revenue is ignored
# bar_chart
view: x is { group_by: status, aggregate: order_count, revenue }
// RIGHT — single aggregate
# bar_chart
view: x is { group_by: status, aggregate: revenue }
```

For multiple metrics: nest separate chart views in a `# dashboard`, or use `y=['revenue','cost']` for multi-measure series.

## Joined Fields in `order_by` — Must Alias First

```malloy
// WRONG — compile error
view: x is { group_by: races.season_year, aggregate: pts, order_by: races.season_year }
// RIGHT — alias then reference
view: x is { group_by: yr is races.season_year, aggregate: pts, order_by: yr }
```

Any time you `group_by` a joined field, create an alias and use it in `order_by`.

## `having:` vs `where:` — Aggregate Filters

```malloy
// WRONG — "Aggregate expressions not allowed in where"
view: x is { group_by: cat, aggregate: n is count(), where: n > 10 }
// RIGHT
view: x is { group_by: cat, aggregate: n is count(), having: n > 10 }
```

- `where:` — filters rows BEFORE aggregation (dimensions/raw columns)
- `having:` — filters AFTER aggregation (measures)

## Aggregating Joined Fields — Method Syntax

```malloy
// WRONG — ambiguous join path
measure: cogs is sum(inventory_items.item_cost)
// RIGHT — method syntax
measure: cogs is inventory_items.item_cost.sum()
```

## Chart Annotation Placement

Place `# bar_chart` / `# line_chart` on the **nested view definition**, not on `nest:` itself. Putting it on `nest:` causes "not a repeated record" errors.

## DRY — Define in Source, Reference in View

```malloy
// WRONG — inline in view
view: summary is { aggregate: revenue is sum(total) }
// RIGHT — reference existing measure
view: summary is { aggregate: revenue }
```

## Time Truncation vs Extraction

| Syntax | What it does | Returns |
|--------|--------------|---------|
| `ts.month` | Truncates to start of month | Timestamp (`@2024-03-01`) |
| `month(ts)` | Extracts month number | Integer (1-12) |
| `ts.year` | Truncates to start of year | Timestamp (`@2024-01-01`) |
| `year(ts)` | Extracts year number | Integer (2024) |

Use `.month` for time series charts (proper date ordering). Use `month()` for cross-year comparison.

**Year integers render with commas** — `year(ts)` displays as `2,018`. Tag with `# number=id` to suppress commas. Same for zip codes, IDs.

## `?` Alternation — Use Commas, Not `and`

The `?` operator has lower precedence than `and`. Use commas to combine filters.

```malloy
// WRONG — `and` binds tighter than `?`, so this fails
where: is_us = true and party ? 'Democrat' | 'Republican'
// RIGHT — commas separate independent filter conditions
where: is_us = true, party ? 'Democrat' | 'Republican'
```

## Query Clauses Are Newline-Separated

Do not use trailing commas between query clauses. Each clause goes on its own line.

```malloy
// WRONG — trailing comma before limit
run: source -> { group_by: status, aggregate: n is count(), limit: 10 }
// RIGHT — newline-separated
run: source -> {
  group_by: status
  aggregate: n is count()
  limit: 10
}
```

Clauses: `group_by:`, `aggregate:`, `nest:`, `order_by:`, `limit:`, `where:`, `having:`, `select:`, `calculate:`

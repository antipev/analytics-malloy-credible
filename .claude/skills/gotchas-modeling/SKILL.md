---
name: gotchas-modeling
description: Common Malloy modeling mistakes and how to avoid them. Read BEFORE writing source definitions, dimensions, measures, or joins. Covers reserved words, NULL checks, date functions, type casts, rename pitfalls, and query-based source gotchas.
---

# Modeling Gotchas

> **Read this before writing Malloy code.** These patterns cause most modeling errors.

## Reserved Words — Backtick Them

**When in doubt, backtick it.** Unquoted reserved words cause cascading errors on unrelated lines.

```malloy
// WRONG                        // RIGHT
dimension: d is Date::date      dimension: d is `Date`::date
```

Words most likely to appear as column names:
```
date, time, day, month, year, quarter, week, hour, minute, second,
number, string, boolean, type, table, source, index, count, sum, avg, min, max,
true, false, null, is, on, with, all, from, by, in, to, for, select, order_by,
top, bottom, desc, asc, row, range, current, window, rank
```

- `number` — only the bare word needs backticking; `account_number` is fine
- `source` — reserved; use a different alias like `traffic_source`

## NULL Checks — `is not null`, NOT `!= null`

```malloy
// WRONG                             // RIGHT
dimension: is_sold is sold_at != null   dimension: is_sold is sold_at is not null
```

## Date Functions vs Properties

```malloy
// WRONG — day_of_week is a function       // RIGHT
dimension: dow is created_at.day_of_week   dimension: dow is day_of_week(created_at)
```

**Property access:** `.month`, `.year`, `.quarter`, `.day`, `::date`
**Function call required:** `day_of_week()`, `week()`, `hour()`, `minute()`, `second()`

## Safe Division — Always `nullif`

```malloy
// WRONG              // RIGHT
a / b                 a / nullif(b, 0)
```

## String Columns Need Casts for Aggregates

```malloy
// WRONG — "Can't use type string"    // RIGHT
measure: avg_score is avg(score)      measure: avg_score is avg(score::number)
```

## Boolean Columns — No Quotes

```malloy
// WRONG                                     // RIGHT
count() { where: complaint = 'true' }        count() { where: complaint = true }
```

Check schema: if `BOOL`, use `true`/`false`. If `STRING`, use `'true'`/`'false'`.

## Never Use `rename:` — Use `internal:` + `dimension:`

`rename:` is incompatible with `include {}` (causes "Can't find field" errors). Since all source files should use `include {}` for curation and documentation, **always use `internal:` + `dimension:` instead of `rename:`**.

```malloy
// WRONG — rename breaks when include {} is added later
source: x is conn.table('t') extend {
  rename: order_type is `Type`
}

// RIGHT — works with include {}, curation, and documentation
source: x is conn.table('t')
include {
  internal: `Type`
  public: order_id, user_id
}
extend {
  dimension: order_type is `Type`
}
```

This also applies to giving cleaner names to columns with spaces or reserved words.

## Cannot Redefine Query-Based Source Columns

Columns from `table -> { group_by, aggregate }` or `conn.sql()` already exist. You cannot re-declare them.

```malloy
// WRONG — "Cannot redefine 'user_id'"
source: facts is conn.table('t') -> { group_by: user_id, aggregate: total is sum(amt) }
  extend { dimension: user_id is user_id }
// RIGHT — add only NEW derived dimensions
source: facts is conn.table('t') -> { group_by: user_id, aggregate: total is sum(amt) }
  extend { dimension: is_high_value is total > 1000 }
```

To add `#(doc)` tags to existing query columns, use `include {}` between the query and extend.

## Never Use `conn.sql()` When Malloy Has a Native Pattern

```malloy
// WRONG — raw SQL for pre-aggregation
source: facts is conn.sql("""SELECT user_id, SUM(amount) AS total FROM orders GROUP BY user_id""")
// RIGHT — Malloy query-based source
source: facts is conn.table('orders') -> { group_by: user_id, aggregate: total is sum(amount) }
```

`conn.sql()` is a last resort for UNNEST, PIVOT, or dialect-specific functions only. Call `searchMalloyDocs` first.

## Duplicate Rows — Check Before Building Measures

```malloy
run: source -> { group_by: pk_field, aggregate: n is count(), having: n > 1, limit: 10 }
```

Symptoms: `sum()` returns astronomical values. Causes: event tables, batch retries, merged sources.

## String Concatenation — Use `concat()`, Not `||`

```malloy
// WRONG — || is not supported in Malloy
dimension: full_name is first_name || ' ' || last_name
// RIGHT
dimension: full_name is concat(first_name, ' ', last_name)
```

## `percentile` — Not a Built-In Function

Malloy has no built-in `percentile()`. Call `searchMalloyDocs("percentile")` for the current approach. As a workaround, use `min()`, `max()`, and bucketed distributions to understand value ranges.

## `except:` Removes Fields From Namespace Entirely

`except:` in `include {}` completely removes fields — dimensions and measures cannot reference excluded fields. Use `internal:` instead when derived dimensions need the raw column.

```malloy
// WRONG — dimension references excluded field
source: x is conn.table('t')
include { except: raw_date }
extend { dimension: order_date is raw_date::date }  // ERROR! raw_date is gone

// RIGHT — internal fields are still available in extend
source: x is conn.table('t')
include { internal: raw_date }
extend { dimension: order_date is raw_date::date }  // Works
```

## Source Order — Define Joined Tables First

Malloy compiles top-to-bottom. Define lookup/dimension tables before the source that joins them, or use `import` statements in multi-file projects.

## MUST Search Docs Before Using Unfamiliar Patterns

Call `searchMalloyDocs` BEFORE first use of any of these — don't guess the syntax:
- `pick` expressions
- Window functions (`calculate`)
- `percentile` or statistical functions
- Time interval functions (`days()`, `seconds()`)
- Query-based sources (source -> { } extend { })
- `!` operator / `sql_number()`

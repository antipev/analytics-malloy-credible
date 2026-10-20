---
name: credible-reports
description: Save a dashboard or notebook the user asks for with save_report, as one .malloy document over a published model. Use for "make me a dashboard", "build a notebook" or "save this as a report" in any chat that has no file tools for a draft.
---

# Dashboards and notebooks with `save_report`

A dashboard or notebook the user asks for is **one `.malloy` document** you hand to `save_report`; it needs a published model to read. `save_report` saves it to the user's workspace as a standalone report, never into a package. When the user wants it in the package, call `edit_package` (when you have it) and write it in the draft instead. In a draft chat with file tools, do not use this: write the report file with the `malloy-reports` skill, which only chats that can edit files carry. A dashboard that already ships in a package is a file you edit, never a new report.

`save_report` takes `layout` (`dashboard` or `notebook`), `title`, `environment`, `package`, `model_path` (the one model the document reads) and `document` (the whole text). The tool description owns the parameter shape. It compiles the document without running it and returns problems as an error: fix them and resend the whole document. When the result says `saved`, tell the user where it landed rather than pointing at a Save button; when it does not, do not claim it was saved.

## What the document must look like

- **A dashboard is the composite form**: `## artifact { layout=grid title="…" tiles=["source -> view", …] }` with the grid width as the sibling tag `dashboard { columns=N }`, both on one line. A `# artifact` on a single query is not read, and `dashboard_columns` is deprecated.
- **A notebook** is the same form with `layout=linear` and no `dashboard { … }`, `colspan` or `break`; prose is a `##|(markdown) name` block listed as `name { kind=text }`.
- **Define the views the tiles name in the document**, on an extension of a source from the model: `source: x_tiles is x extend { view: … }`.
- **Tags go on the line above `view:`**, never inside the view's braces (a tag after `limit:` fails to compile).
- **On a dashboard, layout goes on the view a query tile names**: `# colspan=N` and `# break`, with each row's colspans summing to `columns`. A text tile's `colspan` and `break` go on its `tiles` entry instead. Give each tile a `# label`.
- **A chart plots the first measure in `aggregate:`** unless told otherwise. Tag the one you mean (`# bar_chart { y=revenue }`) instead of reordering the measures.
- **One base model per document.** Every tile reads sources from `model_path`.
- **The text stands alone and compiles in restricted mode**: never write an `import`; it is compiled on top of `model_path`. Publisher's restricted mode also refuses `given:` and `##!`, but `save_report` resolves the document's own filters before it compiles (below): it writes each one's value in as a literal and drops the declarations and the `##!` line, so a `given:` the report declares for itself, with `##! experimental.givens`, is the one exception. Never a `##!` for anything else.
- **A filter the reader picks is the report's own.** Declare it in the document as a package report does: `##! experimental.givens` first, then `given: BRAND :: filter<string> is f''` with its control tags (`# label="Brand" control=select suggest { source=orders dimension=brand_name }`) on the lines above, and bind it on each tile's view that it filters, `view: by_brand is by_brand + { where: brand_name ~ $BRAND }`. Never give it a name the model declares (check `get_context`) or `GROUPS`, `ORG_ROLES`, `CELL_ID`: `save_report` refuses the document. The model's own givens apply as well; reference one by name only to read it, never redeclare it. Never ask a modeler to add a reader's filter to the model. Chat and MCP queries do not see a saved report's own filters, so say the value you mean when you query the same data.

```malloy
## artifact { layout=grid title="Storefront overview" tiles=["overview_tiles -> kpis", "overview_tiles -> revenue_by_month"] } dashboard { columns=12 }
source: overview_tiles is order_items extend {
  # colspan=12
  # label="Key figures"
  view: kpis is { aggregate: revenue is total_sales, orders is order_count }

  # colspan=12
  # break
  # label="Revenue by month"
  # line_chart { y=revenue }
  view: revenue_by_month is { group_by: month is created_at.month, aggregate: revenue is total_sales }
}
```

Renderer tags are in `.claude/skills/malloy-charts/SKILL.md`; query syntax is `.claude/skills/malloy-queries/SKILL.md`.

## Editing

To change a saved document, call `save_report` again with the same `title` and the full revised `document`.

## Legacy

`cells_json` (a whole notebook of markdown and query cells, with `imports_json`) and `cell_json` + `cell_index` (one cell) still work for reports saved that way. Do not start a new report with them.

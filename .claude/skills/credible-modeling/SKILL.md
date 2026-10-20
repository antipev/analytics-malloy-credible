---
name: credible-modeling
description: Build local semantic models with Malloy and Credible. Read this skill whenever the user asks about modeling data or specifically mentions Malloy or Credible.
---

# STOP - READ BEFORE WRITING ANY MALLOY CODE

> **AI AGENTS: You MUST review this file before writing Malloy code.** This workflow's steps each have a dedicated skill, loaded on demand via its `skill:<name>` reference — they are not all in context up front. Load the relevant skill as needed before acting. Before writing code, also load the gotcha skills: `.claude/skills/malloy-gotchas-modeling/SKILL.md`, `.claude/skills/malloy-queries/SKILL.md`, and `.claude/skills/malloy-charts/SKILL.md`.

## Pre-Flight Checklist

1. **Discover first** — ground yourself before writing ANY code, with the tool that matches what you are modeling. Data lives in one of three places, and each has its own tool:
   - **Already in a published package**: `get_context` returns its sources, views and fields. A package's models can also read its own data files, and nothing else can: no other package reads them, and no warehouse table holds that data. So when the user's data is in a published package's files, model it in that package, and never offer a new package or a model built from warehouse tables for that data. In the Credible app, `list_environments` with `environment` marks such a package `data: "embedded files"`; open it with `edit_package` when you have that tool, and otherwise link the package page, whose Edit button opens a draft. Offer `show_upload_form` only when no package holds the file. Locally, the files are in that package's folder: edit the package there.
   - **A database with no package yet**: `search_database_schema` walks the connection's tables and gives each one's columns. It covers only the tables chosen as searchable on the connection page, so a table it doesn't return may still exist: never say it doesn't.
   - **A data file in the package** (CSV, Parquet, JSON or Excel `.xlsx`): neither tool above can see it. Models read it by its path from the package root, through DuckDB: `source: orders is duckdb.table('orders.csv')`. It can sit at any path, so `orders.csv` is as likely as `data/orders.csv`; never assume `data/`. When the user names a file, or a name ending in `.csv`/`.parquet`/`.json`/`.xlsx`, query that path **first**, before any schema search, at the root and then under `data/`. If it fails, read the error: "No files found" means the path is wrong or the file isn't there, and anything else is a problem in your query. Query a file before defining fields, e.g. `run: duckdb.table('orders.csv') -> { select: * limit: 5 }`. `.xlsx` reads only the first sheet, and a blank row silently drops every row after it, so check the row count. For another sheet or a title row, read it in the model file, not in a query: `source: q2 is duckdb.sql("select * from read_xlsx('budget.xlsx', sheet='Q2')")`. A `run:` query may not contain `duckdb.sql(...)`. Where the file is and how you query it depends on where you are working:
     - **In the Credible app** (a modeling chat, onboarding or not): files live in the draft, which the user fills through an upload. Look in the draft context's data-file list and in `list_files` (which also names small CSV and JSON files the draft holds as text); a file in neither may be one a published package carries (see **Already in a published package** above). Query with `execute_query` and `live: true`. A live query runs against a `.malloy` file, so when the draft has no model yet, first `create_file` the model file it will become (e.g. `orders.malloy`) holding only that `source:` line, then query it. That file is the model, not a throwaway probe. When the user has a file to bring in, or the one they named is in neither the draft nor a published package, call `show_upload_form` when you have it (without it, ask them to add the file from the package page in the app): it opens an upload card in the chat that puts the files in this chat's draft, or a new one. End your turn after it; the next message names the uploaded paths. Never ask them to paste file contents, and never send them to another product to load it.
     - **Locally** (an IDE, Claude Code, `cred dev`): the files are on disk and nothing is uploaded. Put the file inside the package folder (see **Local DuckDB Data** below for the layout and `workingDirectory`), reference it package-root-relative, and query it with `execute_query_draft`. Files under the package root are embedded when it is published.

   Never guess connection names or field names.
2. **Search docs proactively** — call `search_malloy_docs` BEFORE writing unfamiliar patterns (window functions, query-based sources, pipelines, YoY, cohorts, rendering). Don't guess — Malloy syntax is specific and SQL intuition is often wrong.
3. **Check diagnostics** after writing — fix the FIRST error first, errors cascade. The app has no editor and no diagnostics tool, so run any query against the source with the draft query (`execute_query` with `live: true` and `model_path` = the draft file; `execute_query_draft` in an IDE) and read the compile error it returns. After changing a model that other files import (dashboards, notebooks, another model), run a live query on each of those files too: a change can break an importer without breaking the model, e.g. a new `given` the importer doesn't import, or a renamed field. The package builds only when every file compiles.
4. **Read the gotcha skills** — `.claude/skills/malloy-gotchas-modeling/SKILL.md`, `.claude/skills/malloy-queries/SKILL.md`, and `.claude/skills/malloy-charts/SKILL.md` prevent the most common mistakes.

**Quick syntax reminders:**
1. **Backtick reserved words:** `` `Date` ``, `` `Hour` ``, `` `Timestamp` ``, `` `Type` ``, `` `number` ``, `` `source` ``
2. **Use `having:` for aggregate filters** — not `where:` on measures
3. **Alias joined fields in `group_by`** if using them in `order_by`
4. **`count()` counts rows; `count(x)` counts distinct values of `x`** — write `count(x)`, never `count(distinct x)`
5. **One tag per line** — `# label="Revenue"` and `# currency` on separate lines
6. **No fixed scale on measures** — use `# currency` not `# currency=usd0m`
7. **Cast strings for aggregates:** `avg(score::number)` not `avg(score)`
8. **Boolean columns:** use `= true` not `= 'true'` (no quotes!)

---

## Planning and `modeling-notes.md`

If the IDE has a native plan mode, use it for the high-level approach — do data exploration during planning, then present a concrete plan for user approval before writing any files. Once approved, write `modeling-notes.md` during execution to record decisions (scope, sources, key choices, prior art, gaps). This file persists alongside the model.

Every decision the user has **not** settled goes in it under an **Open decisions** heading, written as an assumption rather than a fact, and comes back at Review. An unlabelled assumption is indistinguishable from a confirmed definition to everyone downstream.

## 9-Step Modeling Workflow

The agent orchestrates all steps. Steps marked **(user)** pause for input. Each step has a dedicated skill with full instructions (referenced as `skill:<name>` in the table below); load the relevant one when needed.

**A field is not complete until it has its definition, `#(doc)` tag, and rendering tags — and any threshold or business convention inside it is either user-confirmed, derived from a distribution you actually queried, or flagged as an assumption in its own `#(doc)`.** Documentation is part of defining a field, not a separate activity. `#(index)` is deliberately *not* on that list — indexing is a separate decision, made per dimension for a reason, and above 5,000 distinct values it needs recorded evidence that queries benefit. Never add it just to make a field feel finished. Read `.claude/skills/credible-document/SKILL.md` for full documentation standards (doc string writing, the `#(index)` decision ladder, tag ordering).

```
DISCOVER → SCOPE → SOURCES → DEFINITIONS → BUILD BASE → BUILD JOINED → REVIEW → CURATE → PUBLISH
 (silent)  (user)   (user)      (user)       (agent)      (agent)      (user)   (user)   (user)
```

| Step | Skill | What Happens |
|------|-------|-------------|
| 1. Discover | `.claude/skills/malloy-discover/SKILL.md` | Scan tables, columns, distributions; detect prior art |
| 2. Propose Scope | `.claude/skills/malloy-define/SKILL.md` | Present findings, user selects focus |
| 3. Propose Sources | `.claude/skills/malloy-define/SKILL.md` | Propose source plan, user confirms architecture |
| 4. Propose Definitions | `.claude/skills/malloy-define/SKILL.md` | Propose fields per base source, user confirms logic |
| 5. Build Base Sources | `.claude/skills/malloy-model/SKILL.md` | Write fully documented base source files (one per table), check diagnostics. Read `.claude/skills/credible-document/SKILL.md` for doc standards. |
| 6. Build Joined Sources | `.claude/skills/malloy-model/SKILL.md` | Write fully documented joined source files, validate. Read `.claude/skills/credible-document/SKILL.md` for doc standards. |
| 7. Review | — | Work through the review checklist below; the user confirms or corrects each item |
| 8. Curate | `.claude/skills/malloy-model/SKILL.md` | Propose access controls — field access modifiers and the discovery surface (a root `index.malloy` that exports the entry points, see `.claude/skills/credible-explores-and-visibility/SKILL.md`) via `.claude/skills/malloy-model/SKILL.md`; row-level secure givens per **Access Control: Secure Givens** below, user approves — optional, ask user |
| 9. Publish | `.claude/skills/credible-publish/SKILL.md` | User explicitly requests publish |

### The (user) steps are stops, not progress reports

These are governed semantic models: the business decisions inside them have to be confirmed by someone who knows the business, and the **(user)** steps exist to collect that confirmation. A model can compile, run, and be fully documented while being wrong everywhere it guessed — and you are perfectly capable of building the whole thing without pausing once, which is the failure this workflow exists to prevent.

So make each stop worth stopping at:

- **Show the data the proposal rests on.** A grain claim is proven by a count, not by a column name. A threshold comes from a distribution you queried. Put those numbers in front of the user alongside the proposal, not after it.
- **Ask them to check the thing you cannot check.** You can tell whether a query compiles and whether the joins fan out. Only they can tell whether the number is right. Say which of the two you are asking for, and never report a returned result set as proof the definitions are correct.
- **Never make "continue" the easiest thing to click.** An affirmative option that only says "let's build it" invites a reflex click and tells the user nothing about what they just agreed to. Name what is being confirmed ("Use `invoice_date` as the revenue date"), and put the adjust path beside it ("Change the grain", "Show me the other candidates"). Where several definitions are defensible, show their counts side by side so choosing means comparing.
- **When a decision goes unanswered** — the user declines to decide, or nobody is there to ask — do not proceed as though it were settled. Take your best-supported position, label it an assumption in the field's own `#(doc)`, record it under **Open decisions** in `modeling-notes.md`, and raise it again at Review.

### Step 7 Review is a checklist, not a summary

Present these to the user, with your answers:

- **Which definitions did the user actually confirm?** List them. Everything else is an assumption.
- **Which thresholds and bucket boundaries did you choose?** For each: the evidence behind it, and the `#(doc)` hedge that marks it as yours rather than theirs.
- **Which questions are still unanswered?** Each one needs a labelled assumption and an **Open decisions** entry.
- **Does the headline metric have more than one defensible definition?** If so, that is a blocking question: put the candidates to the user with their counts side by side, not in a footnote.

The user working through that list is what makes the model governed. A summary of what you built is not a checkpoint.

**Two paths to a model — both produce the same fully documented result:**
- **Schema-first:** "Model my data" → 9-step workflow above, loading the relevant `skill:<name>` skills
- **Analysis-first:** "Explore this data" → `.claude/skills/malloy-analysis/SKILL.md` → formalize via `.claude/skills/malloy-model/SKILL.md` (it surfaces the analysis-to-model workflow)

After analysis completes, **always recommend formalizing into a model.**

---

## Agent Behavior

**Research before asking.** Present proposals with evidence. Never ask open-ended questions — propose with data and let the user confirm.

**Use business language.** Say "I simplified the column name" not "reserved word replaced." Don't expose Malloy internals unless the user asks.

**Talk about the model, not the steps.** The user doesn't have the skill files open, so never name a step ("Steps 3 and 4", "Moving to Step 5"). Don't announce the next phase either ("Now I'll write the source files"): start it, and let the tool call's `header` say what it does. Say something when the direction changes, such as a table you're leaving out or a join that didn't hold, in a sentence on what you found and what you're doing instead.

**Say each option once.** When asking the user to choose, put the options in an `ask_user` card if your host has one, and let your prose carry only the evidence and your recommendation. Fall back to lettered options with one-line descriptions when there is no card. Never both — the same three choices as prose and again as card options reads as two decisions and buries the recommendation.

**Complete all workflow steps.** Once modeling begins, complete through review. A field without documentation is not finished. If you lose track, read `modeling-notes.md`. At the end, suggest publishing and a notebook. Leave that out only when your host's instructions say it shows follow-up suggestions of its own, which then carry them.

## Route by Intent

| User says... | Route to |
|-------------|----------|
| "Model my data", "create a model" | 9-step workflow (`.claude/skills/malloy-discover/SKILL.md`) |
| "Model from LookML" | 9-step with prior art via `.claude/skills/malloy-lookml-review/SKILL.md` |
| "Explore this data", "what's interesting?", "show me the top X" | `.claude/skills/malloy-analysis/SKILL.md` (its no-specific-question branch) |
| "Make me a dashboard" or a notebook, with file tools on a draft | A report file, via `.claude/skills/malloy-reports/SKILL.md` (see **Dashboards, notebooks and their filters** below) |
| "Make me a dashboard" or a notebook in a chat with no draft to edit | `save_report` with `layout`, via `.claude/skills/credible-reports/SKILL.md` |
| "Build a data app", or a custom page with its own design or interactions (a `public/*.html` file in the package) | the `credible-data-apps` skill, where your host ships it |
| A `# dashboard` view inside a report, or "create views" on an existing model | `.claude/skills/malloy-analysis/SKILL.md`, plus `.claude/skills/malloy-charts/SKILL.md` or `.claude/skills/malloy-reports/SKILL.md` as needed |
| "Build a model but not sure what metrics" | `.claude/skills/malloy-analysis/SKILL.md` first, then formalize via `.claude/skills/malloy-model/SKILL.md` |

**If the user's first message is a data question** (not "build me a model"), answer it with `.claude/skills/malloy-analysis/SKILL.md`, and record what the answer assumed with `.claude/skills/malloy-model/SKILL.md-as-you-go`, which also covers questions asked against tables that have no model yet. For open-ended exploration, `.claude/skills/malloy-analysis/SKILL.md` has a no-specific-question branch. After the analysis, **always recommend formalizing via the analysis-to-model workflow** (invoke `.claude/skills/malloy-model/SKILL.md`).

## Additional Support Skills

These supplemental skills can also be loaded as needed:

- **`.claude/skills/malloy-gotchas-modeling/SKILL.md`** ("When a Model Will Not Compile") — Fix compile errors and interpret diagnostics. Its first option, an editor-diagnostics tool, does not exist in the app; take its second, the draft query (`execute_query` with `live: true`)
- **`.claude/skills/malloy-materialization/SKILL.md`** — Persist an expensive source into a pre-built table (`#@ persist`), and debug a materialization that isn't building
- **`.claude/skills/credible-materialization/SKILL.md`** — How materialization behaves on Credible: Credible-managed storage (`storage=credible`), builds on publish, build status, freshness, and why draft queries never read a materialized table

## Credible MCP Tools

| Tool | Purpose |
|------|---------|
| `search_database_schema` | Raw schema: tables, columns, connection names |
| `execute_query` with `live: true` + `model_path` (app) / `execute_query_draft` (IDE) | Run ad-hoc queries against the DRAFT's files for validation |
| `search_malloy_docs` | Search Malloy docs (call BEFORE unfamiliar patterns) |
| `show_upload_form` (app) | Open the in-chat upload card, which puts the user's data files in this chat's draft |
| `publish_package` | Publish to Credible (only when user asks; in the app, a preview call first, then `confirmed: true` after the user confirms) |
| `execute_query` without `live` (+ `get_context` for discovery) | Query a PUBLISHED package — the only way to see published data; the draft query only sees draft files |

Tool names above are bare; the exact prefixed name depends on the surface hosting the chat — match against the tools you actually have.

Never guess connection names — always get them from `search_database_schema`.

**Files mint real drafts.** In the Credible app, the first `.malloy` file you create mints a draft package in the chat's workspace — never create throwaway files to probe a schema question; use `search_database_schema` (and `get_context`/`execute_query` for published data) instead. A data file is the exception the Discover step describes: its first model file is where you look at it. A chat edits one draft: a second package needs a fresh chat. To change an already-published package, call `edit_package` with its environment and name wherever you have it (a chat on that package, or a chat with no draft yet); never start a new package to change an existing one. Without `edit_package`, link the package page: its Edit button opens a draft.

**Name the package before the first file.** Don't stop to ask the user for a name. Pick a short, descriptive, directory-safe name from the data or the user's own words (`ecommerce`, `f1_racing`, `flights`); a basename like `model.malloy` is a misleading default. Tell the user the name once, alongside your first result. On a git-backed environment the name is permanent once the first `.malloy` file is created, so say the name you picked in your reply before that first `create_file`. Where the name lives depends on the environment:
- **Git-backed environment:** the name is fixed when the first `.malloy` file mints the draft, and can't be changed afterwards. Pass a short, directory-safe name as `package_name` on that first `create_file`.
- **Any other environment:** `package_name` is ignored. The name is `publisher.json`'s `name`, and changing it there changes the name the package publishes under.

The package name is separate from the chat title.

---

## Access Control: Secure Givens

Scope rows and gate sources with **givens** — Malloy `given:` parameters that, for access control, Credible resolves from the caller's **verified identity** (never the caller). This is the going-forward mechanism; it supersedes `#(filter)`-based row-level security.

### Row-level access with secure givens

A given used for access control must be **secure**: marked `#(secure)` (or the built-in `$GROUPS`), set-valued, and resolved server-side from verified identity. Declare `given:` at the **top of the model** (not inside the source) and reference it with `$`:

```malloy
##! experimental.givens

given:
  #(secure)
  ALLOWED_TENANTS :: string[]

source: orders is conn.table('orders') extend {
  // each caller sees only the tenants Credible grants them
  where: tenant_id in $ALLOWED_TENANTS
  measure: order_count is count()
}
```

`$GROUPS` is a built-in secure given (the caller's group memberships) — declare it (no `#(secure)` needed; Credible reserves and populates it) and scope rows by membership:

```malloy
##! experimental.givens

given:
  GROUPS :: string[]   // reserved secure name — platform-populated; a renamed/custom given needs #(secure)

source: sales_by_group is conn.table('sales') extend {
  where: region in $GROUPS
}
```

Secure givens are **set-valued** by design (a scalar secure given isn't enforced). The modeler only declares and uses them — an admin assigns their per-user / group / default values in the Credible admin portal (the model never holds the values).

### Source gating with `#(authorize)` and `#(access_filter)`

Two annotations over a secure given. `#(authorize)` decides whether the caller may reach the source at all — denial is a **403**. `#(access_filter)` decides which rows they see: a caller matched nowhere gets **200** with zero rows. Both sit on their own line directly above the `source:` line, and a source with neither, own or inherited, is unrestricted:

```malloy
##! experimental.givens

given:
  GROUPS :: string[]

#(authorize) 'admin' in $GROUPS
source: admin_orders is orders extend {}
```

```malloy
##! experimental.givens

given:
  ALLOWED_TENANTS :: string[]

#(access_filter) tenant_id in $ALLOWED_TENANTS
source: tenant_orders is orders extend {}
```

Each annotation's body is narrow — one or more `and`-joined terms, `<op>` fixed by the given's arity (`in` for a set-valued given, `=` for a scalar one) — and a term written on the wrong annotation is refused at load naming the other. `#(authorize) true`/`#(authorize) false` are the only sentinels, legal on the lock alone: `false` locks a source outright, and `true` is how an extension of a locked base re-opens it (leaving the annotation off instead **inherits** the `false`). Full grammar, inheritance, and the persistence caveat are in `.claude/skills/malloy-model/SKILL.md`.

### The trust rule

An access gate is only a real boundary over a **secure** given. `#(authorize)` or `#(access_filter)` — or a `where:` you rely on for access — over a plain, caller-supplied given is **bypassable**: the caller sends the value and passes the gate. A `where:` over an ordinary given is fine for *parameterization*; it just isn't an access boundary. `$GROUPS` is secure because Credible **reserves the name**, not because of how it's declared — so copying a `$GROUPS` gate and renaming it to your own attribute yields a **bypassable** gate unless you mark that given `#(secure)`.

In a package curated with a root `index.malloy`, the gated source's givens have to reach `index.malloy` too: import the file that declares `GROUPS` whole, or name `GROUPS` in the selective import. Otherwise the package refuses to load (see `.claude/skills/credible-explores-and-visibility/SKILL.md`).

The generic Malloy `given:` + `#(authorize)`/`#(access_filter)` grammar (what each body may contain, the no-defaulted-given rule, inheritance through `extend` versus a query derivation, and the fail-closed behavior when a derivation drops a column a term reads) is in `.claude/skills/malloy-model/SKILL.md`; this section covers Credible's secure-given enforcement on top of it.

## Dashboards, notebooks and their filters

A dashboard or notebook is a Publisher-owned `.malloy` report; `.claude/skills/malloy-reports/SKILL.md` owns its format. These rules are Credible's, on top of it:

- **There is no report builder in Credible.** The app has no Design view, Edit-to-builder or New-report dialog, so `.claude/skills/malloy-reports/SKILL.md`' "Editing in the Console" does not apply: a person changes a dashboard or notebook by asking in chat, and you edit its file.

- **A report is one file holding its whole visual specification**: tiles, text, layout and its filters. It imports semantic models and nothing else visual. A filter a reader picks (manufacturer, date range, region, a "major only" toggle) is the report's: declare it in the report file as a `given:` with its control tags, under `##! experimental.givens`, and bind it on each tile that should answer to it, `view: t is v + { where: <dimension> ~ $<GIVEN> }`. A `filter<…>` given binds with `~`; a plain `date` or `number` given is a value and binds with `>=`, `<=` or `=`.
- **Never put a reader's filter in the model** as a given a source's own `where:` reads, even where `.claude/skills/malloy-model/SKILL.md` suggests a given on the source. Binding is per declaration, not per name, so the report cannot take that given over with its own; the control lives outside the file that is supposed to specify the report, so editing the report cannot show or change it; and the model's `where:` filters every consumer of the source: chat, MCP and other reports.
- **A given stays in the model only when the data's meaning or access depends on it**: a row-level gate (`#(authorize)`, `#(access_filter)`, `#(secure)`, `$GROUPS`), a value every query must supply (a given with no default), a parameter a measure, dimension or view is defined with, or scoping the author deliberately applies to every consumer. A report imports such a given by name and never re-declares it.
- **Trusted givens stay in the model.** `GROUPS`, `ORG_ROLES`, `CELL_ID`, any org-registered name, and anything marked `#(secure)` are resolved from the caller's verified identity. Never declare one in a dashboard or notebook, never give one a control, and never bind one on a tile; the model's gates already apply to every query the document runs.
- **A filter given must not reuse a trusted name, or a name the model declares.** A `save_report` document may declare its own filters (`save_report` resolves them before Publisher's restricted compile sees the text), but one named like a given the model declares, or `GROUPS`, `ORG_ROLES`, `CELL_ID`, is refused before anything compiles. In a package report, reusing an org-registered secure name as a plain given is refused at publish, except `GROUPS`, which the router always supplies. Pick a different name for the control.
- **On a gated source, say up front that each viewer sees only their own rows.** Tell the user before they build, so a number that differs between two people is expected rather than reported as a bug.
- **Never rewrite a query to get around a denial.** A 403, or a gated source returning no rows, is the model working. Do not reshape the query, switch sources, or re-join to reach the data; tell the user what was refused and who can change the gate.

### Converting legacy reports and `#(filter)`

- **Each optional `#(filter)` on a source** becomes a `given:` declared in every report that reads the source, bound per tile as above, and is deleted from the model. Only a `#(filter) … required` stays in the model, as a given with no default that the source's `where:` reads.
- **A `.malloynb`** becomes a `.malloy` notebook beside it: `layout=linear` with `tiles=[…]`, each query a view tile and each prose cell a `##|(markdown) name` text tile. Never write `run:` cells; that report form is retired.

Before, in the model `recalls.malloy`:

```malloy
#(filter) dimension=Manufacturer type=in
#(filter) dimension="Report Received Date" type=greater_than
source: recalls is duckdb.table('recalls.csv') extend { … }
```

After, both annotations are deleted from `recalls.malloy` and the dashboard owns its filters:

```malloy
##! experimental.givens
## artifact { layout=grid title="Recalls" tiles=["recall_tiles -> by_month", "recall_tiles -> by_subject"] } dashboard { columns=12 }
import { recalls } from "recalls.malloy"

# label="Manufacturer" control=multiselect suggest { source=recalls dimension=Manufacturer }
given: MANUFACTURER :: filter<string> is f''
# label="Received"
given: RECEIVED :: filter<date> is f''

source: recall_tiles is recalls extend {
  # colspan=8
  # label="Recalls by month"
  view: by_month is recalls_by_month + { where: Manufacturer ~ $MANUFACTURER, where: `Report Received Date` ~ $RECEIVED }

  # colspan=4
  # label="Top subjects"
  view: by_subject is top_subjects + { where: Manufacturer ~ $MANUFACTURER, where: `Report Received Date` ~ $RECEIVED }
}
```

## SQL-to-Malloy Quick Reference

| SQL | Malloy |
|-----|--------|
| `COUNT(*)` | `count()` |
| `COUNT(DISTINCT x)` | `count(x)` |
| `NOW()` | `now` |
| `CASE WHEN...END` | `pick...when...else` |
| `col IN ('a','b')` | `col ? 'a' \| 'b'` |
| `COALESCE(a,b)` | `a ?? b` |
| `CAST(x AS type)` | `x::type` |
| `DATEDIFF(day, a, b)` | `days(a to b)` |
| `CONCAT(a, b)` or `a \|\| b` | `concat(a, b)` |
| `TIMESTAMP_DIFF(a, b, SECOND)` | `seconds(b to a)` |

## Critical Rules

1. **All keywords require colons**: `source:`, `dimension:`, `measure:`, `view:`
2. **Use `is` not `as`**: `dimension: name is expression`
3. **Arrow operator required**: `run: source -> { operations }`
4. **Specify join type**: `join_one:`, `join_many:`, `join_cross:`
5. **Safe division**: `revenue / nullif(count, 0)`
6. **Group definitions under one keyword**: `measure:` then indent fields beneath

## Common Anti-Patterns

```
WRONG: source flights is ...           RIGHT: source: flights is ...
WRONG: dimension: x as y               RIGHT: dimension: y is x
WRONG: count(*)                        RIGHT: count()
WRONG: count(distinct x)               RIGHT: count(x)
WRONG: revenue / order_count           RIGHT: revenue / nullif(order_count, 0)
WRONG: run: src { ... }                RIGHT: run: src -> { ... }
```

## Reserved Words — Scan Schema First

**Malloy has many reserved words. When in doubt, backtick it.** Most likely to appear as column names:

```
date, time, day, month, year, quarter, week, hour, minute, second,
number, string, boolean, type, table, source, index, count, sum, avg, min, max,
true, false, null, is, on, with, all, from, by, in, to, for, select, order_by,
top, bottom, desc, asc, row, range, current, window, rank
```

- `number` — only the bare word needs backticking; `account_number` is fine
- `source` — reserved; use a different alias like `traffic_source`
- `string`, `boolean`, `true`, `false` — backtick any column with these exact names

---

## Gotcha Skills — Read Before Writing Code

The following skills contain detailed WRONG/RIGHT patterns that prevent the most common Malloy errors. **Load them before writing code:**

- **`.claude/skills/malloy-gotchas-modeling/SKILL.md`** — Reserved words, NULL checks, date functions, type casts, rename pitfalls, query-based source gotchas, `conn.sql()` anti-pattern
- **`.claude/skills/malloy-queries/SKILL.md`** ("More Compile Mistakes") — Aggregate filters, joined field aliasing, time truncation vs extraction, chart constraints
- **`.claude/skills/malloy-charts/SKILL.md`** — Tag syntax, scale rules, sparkline setup, big_value patterns

## Local DuckDB Data (Credible)

When a model reads local files with `duckdb.table('…')`, the relative path is resolved against **different base directories** depending on what's running it:

| Context | Resolves `duckdb.table()` paths relative to |
|---|---|
| `execute_query_draft` (IDE extension MCP) | `package_root_path` (the dir containing `publisher.json`) |
| VS Code Malloy extension | `workingDirectory` in `malloy-config.json` (default: project root) |
| Published package | the package root (dir containing `publisher.json`) |

They only agree when **package root == `workingDirectory`**. Otherwise you get a split-brain symptom that looks like a broken model but isn't: queries run fine via `execute_query_draft`, while the IDE underlines the source with `IO Error: No files found that match the pattern "..."` (or the reverse). Chasing it as a Malloy error is a dead end — it's a path-base mismatch.

**Fix — adopt the embed-data layout and pin `workingDirectory` to the package root.** Keep data in a `data/` folder at the package root and reference it with package-root-relative paths (no `../`):

```
my-package/                     ← package root (publisher.json lives here)
├── publisher.json
├── model/
│   └── people.malloy           //  source: p is duckdb.table('data/raw/people.csv')
└── data/
    └── raw/people.csv
```

```malloy
// WRONG — climbs outside the package: breaks IDE resolution AND won't embed on publish
source: people is duckdb.table('../data/raw/people.csv')

// RIGHT — package-root-relative: identical locally, in execute_query_draft, and published
source: people is duckdb.table('data/raw/people.csv')
```

```json
// malloy-config.json — point workingDirectory at the package root so the IDE matches the engine
{ "connections": { "duckdb": { "is": "duckdb", "workingDirectory": "/abs/path/to/my-package" } } }
```

Because `data/` sits at the package root, these exact paths embed automatically on publish ([embed-data docs](https://docs.credibledata.com/how-to/modeling/embed-data)) — no path edits between local dev and production. Note `workingDirectory` is an absolute, machine-specific path, so keep `malloy-config.json` out of git.

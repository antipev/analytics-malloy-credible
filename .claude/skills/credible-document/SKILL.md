---
name: credible-document
description: 'Add documentation and search tags (#(doc), #(index)) to Malloy models for Credible AI. Use when user asks to "add documentation", "make discoverable", "add doc tags", "index for credible", or "document the model". `#(doc)` documents what a field means; `#(index)` indexes a dimension''s values for semantic search. To curate which sources agents see or can query, use a root index.malloy (see `.claude/skills/credible-explores-and-visibility/SKILL.md`); the `#(agent-hidden)` / `##(agent-hidden)` retrieval-hiding tags are replaced by a root index.malloy; only how to migrate them is documented here. For parameterizable filters and runtime parameters, see `.claude/skills/malloy-model/SKILL.md` — those are runtime/modeling constructs, not documentation tags.'
---

# Credible AI Documentation

Add Credible's discoverability tags to make fields searchable, filterable, and selectively visible to AI:

| Tag | Purpose | Goes on |
|-----|---------|---------|
| `#(doc)` | Plain-language description for natural-language search | source, dimension, measure, view, join |
| `#(index)` | Index a dimension's values for semantic search, so AI can match by value (not just description). Above 5,000 distinct values, confirm queries will benefit before adding it — see below | categorical dimensions |
| `#(filter)` | Deprecated, prefer `given:`. Parameterizable filter (runtime/modeling concern — see `malloy-model`) | source |
| `#(agent-hidden)` | *Replaced by a root `index.malloy`.* Never author; migrate existing uses (see below) | source |
| `##(agent-hidden)` | *Replaced by a root `index.malloy`.* Never author; migrate existing uses (see below) | top of file |

> **Prior art documentation seeds:** If `_plan/prior_art.md` exists, **you MUST** invoke the prior-art skill for doc tag seeds before writing documentation — for LookML prior art, `.claude/skills/malloy-lookml-review/SKILL.md` (follow its DOCUMENT reference).

## #(doc) Tag

Add before any source, dimension, measure, view, or join. When multiple fields share a keyword, use it once as a block header. Tags and field names are indented under the keyword; tags go on the line(s) directly above the field they annotate.

**Tag ordering** (when a field has multiple tags): `#(index)` → `#(doc)` → render tags (`# currency`, `# label`, etc.) → field name. Separate each field group with a blank line:

```malloy
#(doc) Customer who placed the order
join_one: users with user_id

dimension:
  #(doc) Date the order was placed (UTC)
  order_date is created_at::date

measure:
  #(doc) Total revenue from all orders in USD
  # currency
  revenue is sum(total)
```

### Writing Doc Strings for Retrieval

Doc strings power Credible AI search — users type natural language questions and the system matches against your `#(doc)` strings. Write descriptions that match how analysts would search:

- **Include business meaning**, not code mechanics — what it represents, not how it's implemented
- **Include units** (USD, count, percentage) — a unit is part of what a number *means*
- **Lead with meaning; list values only when the list is short** — a handful of values makes a description concrete and is worth including. Roughly ten is the point where it stops helping; past that you're writing a data dump instead of a description, and it goes stale as soon as someone adds a value. Treat that as a flexible rule of thumb, not a hard cap. `#(index)` is the mechanism for making a long value space discoverable — a doc string is not
- **Avoid Malloy jargon** — never use "filterable", "groupable", "dimension", "measure", "aggregation"

**Good examples:**
- `#(doc) Total revenue from completed orders in USD` — matches "what was our revenue?"
- `#(doc) Customer signup date (UTC)` — matches "when did the customer join?"
- `#(doc) Order status: pending, processing, shipped, delivered, cancelled` — matches "what are the order statuses?". Five values, so listing them is concrete and cheap
- `#(index)` + `#(doc) Where the order sits in the fulfillment lifecycle, from placement through delivery or cancellation` — the other valid shape, and the better one when the values change often or aren't the words a user would guess (`CANCEL`, `SHIP_PEND`). The doc carries the meaning and `#(index)` keeps the values discoverable, so neither has to hard-code the list

**Bad examples:**
- `#(doc) Filterable dimension for order status` — no analyst searches for "filterable"
- `#(doc) Groupable by region` — "groupable" is a system concept
- `#(doc) Aggregation of total sales` — "aggregation" doesn't match natural queries
- `#(doc) Item category: Beer, Wine, Spirits, Seltzer, Cider, Soda, Water, Juice, Coffee, Tea, Snacks, Candy, …` — a long value list crowds out the meaning and is stale on arrival. Describe what the field captures and let `#(index)` carry the values

### Multi-line descriptions — block annotations

If a doc string needs more than one line — a source-level `#(doc)` spelling out what analyses the source answers, or a dimension whose meaning takes a couple of sentences to pin down — use Malloy's **block annotation** syntax so the text can span several lines instead of running off the edge in one giant line: open with `#|(doc)`, write the description, close with `|#`, and put the whole block on the lines **directly above the thing it describes** — a source, dimension, measure, view, or join. That's the same placement as a single-line `#(doc)`; the block just spans more lines.

The route (`(doc)`, `(index)`, …) goes immediately after the opener, exactly as in the single-line form; everything between the opener and closer is the annotation body. A block `#|(doc)` is equivalent to a single-line `#(doc)` with the same text — it just reads better when the text is long.

```malloy
#|(doc)
  Order analysis. Use for revenue, fulfillment, and customer-order joins.
  Covers order status (pending, processing, shipped, delivered, cancelled),
  line-item totals in USD, and the customer who placed each order.
  For customer-health / churn questions, use customer_health instead.
|#
source: order_analysis is orders extend { ... }
```

Pick whichever form fits the text: single-line when the description is short, the block form when it isn't.

**Block annotation rules:**
- **Always close it** (compile error otherwise). `#|` closes with `|#`; an unclosed block fails with `Multi-line annotation is not closed`.
- **The closer sits on its own line, indented to match the opener's column** (also compile-affecting). A `|#` in the middle of a line, or indented deeper than the opener, is treated as body text rather than a closer — so a block whose `|#` never lines up reads as unclosed and won't compile.
- **Indent the body uniformly** (all spaces). Malloy strips the whitespace prefix shared by every body line (Python-style dedent), so matching your code's indentation doesn't leak into the text. Mixed indentation (e.g. a tab on one line, spaces on another) shares no prefix, so nothing is stripped and the raw whitespace lands in the doc text — silently, with no warning or error.
- Everything that applies to single-line doc strings still applies: write for retrieval (business meaning, units, a short value list only when it's genuinely short) and avoid Malloy jargon.

## #(index) Tag — Index Values for Search

`#(index)` indexes a dimension's values for semantic retrieval, so an agent can match a question against the values that are actually in the data. It's what lets a question mentioning "FORD" resolve to the `manufacturer` value `Ford Motor Company`, or "cancelled orders" resolve to the `order_status` value `CANCEL`.

**Index a dimension when agent queries will benefit from it — not by default.** The benefit is concrete and testable: an agent that can see the real values writes a correct filter on the first try, instead of guessing `shipped` when the data says `SHIP_PEND`. If you can't name that benefit for a dimension, leave the tag off. Cardinality is worth measuring because it shapes the answer, and because a very large value set is often a sign you've caught an identifier rather than a category.

**`#(index)` is not required for a field to be usable.** An unindexed dimension is still fully filterable — an agent just fetches its distinct values with a query first. Indexing removes that round trip and makes the values reachable by semantic search rather than exact match. Never add the tag just to make a field "complete".

```malloy
dimension:
  #(index)
  #(doc) Order status: pending, processing, shipped, delivered, cancelled
  order_status is status

  #(index)
  #(doc) Customer company name
  company_name is name

public:
  #(index)
  #(doc) US state where the order was shipped
  ship_state
```

### The decision ladder

Walk this per candidate dimension and stop at the first step that matches.

**Step 1 — Never index these.** No measurement needed; don't propose them at all.

- **Numeric, boolean, date and timestamp columns** (`price`, `quantity`, `is_active`, `created_at`) — **indexing doesn't work on these types.** The tag is ignored, so it buys nothing and leaves the impression the values are searchable. The questions these fields answer are ranges and comparisons ("orders over $100", "orders in March 2024") — a filtering concern, not value search; see `.claude/skills/malloy-model/SKILL.md` for parameterizing a source.
- **Near-unique strings** — UUIDs, surrogate and primary keys, emails, hostnames, usernames, URLs, account and order numbers. Anything where each value identifies one entity. If a user has the exact value they have direct lookup, not a search problem.
- **Free text** (descriptions, notes, comments, review bodies) — **the index drops strings longer than 1032 characters**, so the tag is silently inert on long text. To make free text searchable, derive structured fields from it (extract entities, classify into categories) and consider indexing those instead.
- **Anything not public** — `internal:` or `private:` columns in an `include {}` block are **not indexed**, so `#(index)` on one compiles and does nothing. Index only what's part of the source's public API. If a column's values genuinely need to be searchable, that's an argument for making it public, not for tagging it where it sits.

**Step 2 — Would indexing actually add anything?** The test: *could an agent write the correct filter literal from the column name alone?* The cases worth indexing are the ones where it can't —

- `manufacturer` holds `Ford Motor Company`, not `Ford`
- `order_status` holds `CANCEL` and `SHIP_PEND`, not `cancelled` and `shipped`
- the same country appears as `USA`, `U.S.`, and `United States`; a state as `Texas` or `TX`
- product or show titles are messy and multi-token, so users describe them instead of quoting them ("football shows" needs to reach `Monday Night Football`)

If the values genuinely are inferable from the column name, skip the tag. On a close call for a small categorical dimension, index it — the index stays small, and it's the right mechanism.

**Step 3 — Categorical, and 5,000 or fewer distinct values → index it and report.** No approval needed. Low cardinality is *not* a reason to skip: a dimension with a dozen values is worth indexing when those values aren't guessable. Note the reason in your summary so the user can see what you did and why.

**Step 4 — More than 5,000 distinct values → confirm queries will benefit, then tag.** Above the gate, "these values look searchable" isn't enough. Establish the benefit with at least one of:

- **Example natural-language questions** that need the values, where an agent would land on the wrong literal (or nothing) without the index. Write them down — if you can't produce one, that's the answer.
- **Observed evidence** — a query log or eval run where an agent searched for the dimension's values and didn't surface them. The strongest form, because it's measured rather than predicted.
- **A human confirming** the dimension is worth indexing.

Plus **a measured distinct count**, not an estimate.

Record which of those you relied on, so the basis is visible later. Present the candidates as one batch table rather than field-by-field prompts — distinct counts and evidence, **never size or storage estimates**:

| Dimension | Distinct values | Evidence queries will benefit |
|---|---|---|
| `item_name` | 68,715 | observed: eval Q14 "did we sell any Coors tallboys" — the agent searched item values and surfaced nothing |
| `item_brand` | 5,974 | NL examples: "sales by brand", "how did Coors do last quarter" — brand names users say out loud, not inferable from the column |

The general principle: **`#(index)` is a search aid for values users *describe* (`Ford Motor Company`, `CANCEL`, `Texas`), not values they *already have* (UUIDs, timestamps, order numbers) and not values the column name already gives away.** Cardinality decides how much scrutiny that takes — small dimensions index freely, large ones need evidence.

### Process

1. **Identify candidates** — scan every public column and dimension against Steps 1 and 2 above. When in doubt, ask: "could an agent guess a valid value of this field without looking?"
2. **MUST validate against the data** — before tagging anything, confirm the column really is a categorical string and get its distinct count. Never tag on the strength of a column name: agents that skip this flag int columns named `category_code`, or string columns that turn out to hold JSON blobs.
   ```malloy
   run: source -> { index: * }
   ```
   `index: *` is a good profiling pass for types and sample values, but it does **not** give reliable exact distinct counts. For any candidate near or above the 5,000 gate, run an explicit aggregate query for the distinct count and the avg/max value length. Check the exact Malloy aggregate syntax in `.claude/skills/malloy-queries/SKILL.md` or via `search_malloy_docs` — don't guess it.
3. **Tag what clears Step 3**, and list each one with its reason in your summary.
4. **Batch everything above 5,000 into the Step 4 table** with its evidence. Nothing above the gate gets tagged on the strength of judgment alone; if you can't produce evidence for a candidate, leave it untagged and say so.

## Review rubric: #(index) justification (Credible)

The shared `malloy-review` skill's documentation rubric is deliberately neutral and does not know about `#(index)`. When reviewing a Malloy model for Credible, apply this rubric alongside `malloy-review`'s D-series rules (slot it in as D-05, between "One tag per line" and "Source-level documentation describes when to use the source").

### D-05 — every `#(index)` earns its place, and nothing over the gate lacks evidence

This rule looks for **indexes that shouldn't be there**. It does *not* flag a categorical dimension for lacking `#(index)` — an unindexed dimension still works (the agent fetches distinct values with a query), so a missing tag is not a defect.

- **Severity:** **`major` (blocking)** for most cases — see per-case below; `minor` only when the column name already gives the values away · **Category:** docs · machine-checkable (data-driven, falls back to LLM-judgment)
- **Detection — preferred (data-driven).** Run `run: <source> -> { index: * }` via a draft query (`execute_query` with `live: true` in the app, `execute_query_draft` in an IDE) for types and sample values, then an explicit aggregate query for distinct counts on anything that looks large. Per **indexed** dimension:
  - **Numeric / boolean / date / timestamp type** → **`major`**. Indexing doesn't work on these types, so the tag is ignored and reads as working.
  - **`internal:` or `private:` column** → **`major`**. Not indexed. Remove the tag, or make the column public if its values really need to be searchable.
  - **Free text** — max length > 1032 chars, or avg length > ~200 → **`major`**. The index drops strings past 1032 chars, so the tag is inert on exactly the values someone hoped to search. Propose deriving structured fields from the text instead.
  - **Distinct-count ≈ row count** → **`major`**. A per-entity identifier (UUID, email, hostname, account number): a large index with nothing to search for, since anyone holding the exact value needs direct lookup. This is the shape that produces the biggest indexes for the least benefit.
  - **Distinct-count > 5,000 with no recorded evidence that queries benefit** → **`major`**. Over the gate, an index needs example questions, observed retrieval evidence, or explicit human confirmation. Flag it and ask for the basis.
  - **Values inferable from the column name alone** → `minor`. The index works and stays small; it just isn't doing anything.
- **Detection — fallback (no draft query).** Use name + type heuristics, and note in the finding that you fell back so the reviewer can confirm: an index on a column typed numeric / boolean / date / timestamp, or named `*_id` / `*_at` / `*_count` / `*_total` / `*_amount` / `*_price`, or suggesting free text (`description`, `notes`, `comment`, `body`, `summary`), is suspect. Cardinality can't be checked this way — say so rather than guessing.
- **Fix:** remove `#(index)` where it has no effect or the field is an identifier. For an unjustified index over the gate, surface the measured distinct count and let the user decide. For long-text columns, propose deriving structured fields and indexing those.
- **See:** the `#(index) Tag — Index Values for Search` section above (the decision ladder, the 5,000 gate, and the mandatory data validation step).

### Severity and category wiring

Fold these into `malloy-review`'s `severity-taxonomy.md` when reviewing for Credible:

- Most D-05 findings are `major` / blocking under the `docs` category: an `#(index)` that has no effect (numeric / boolean / date / timestamp type, `internal:` / `private:` column, text over 1032 chars), one on a per-entity identifier, and one above the 5,000-value gate with no recorded evidence that queries benefit. Each either reads as working while doing nothing, or indexes a large value space nobody showed a use for.
- The one `minor` case is an `#(index)` whose values the column name already gives away — it works and stays small, so it sits alongside tag-ordering and one-tag-per-line issues.
- Add rows to the severity/blocking table: `#(index)` with no effect, on an identifier, or over the gate without evidence (D-05) → `major` / blocking; `#(index)` on a name-inferable field (D-05) → `minor` / `non-blocking`.

### What to report instead of index coverage

Do **not** track an index-coverage percentage. A ratio of indexed-to-indexable dimensions rewards indexing everything, which is exactly the failure mode this rule exists to prevent.

Report instead:

- **How many dimensions carry `#(index)`**, and the **largest few by distinct count**.

That's a breadth signal a reviewer can actually produce without a distinct count for every indexed dimension. If one source is indexing far more of its dimensions than its neighbours, raise it as a Cross-Cutting Theme.

## #(filter) — deprecated, see `malloy-model`

`#(filter)` is deprecated in favour of native Malloy `given:` parameters. Never add a new `#(filter)` annotation: every use, `required` and `implicit` included, has a `given:` form. See `.claude/skills/malloy-model/SKILL.md` for the replacements and for reading an existing `#(filter)` model.

`#(filter)` is also a `#(...)`-shaped annotation, but unlike the others above it's a **runtime/modeling construct** — it shapes governance, query latency, and correctness, not discoverability. The legacy syntax is documented in `.claude/skills/malloy-model/SKILL.md` alongside the `given:` guidance that replaces it.

One rule worth knowing here: a source's own filters and access gates apply to every consumer that imports it, so a filter a dashboard or notebook reader picks never goes in the model. The report declares it as its own `given:` and binds it per tile; converting a `#(filter)` means moving it into each report that reads the source and deleting it from the model, keeping only a `required` one there as a given with no default (`.claude/skills/credible-modeling/SKILL.md`, **Dashboards, notebooks and their filters**). A report's givens narrow its own queries and never loosen what the source enforces.

## #(agent-hidden) and ##(agent-hidden): replaced by index.malloy

Do not author these tags. A root `index.malloy` that exports the entry points does
everything they did, and also stops a hidden source being queried by name (see
`.claude/skills/credible-explores-and-visibility/SKILL.md`). A source that `index.malloy` does not export is not
listed, and its fields stay findable through any exported source that joins it.

If an existing model carries them, migrate when the user asks to touch curation:

- **`#(agent-hidden)` on a source:** leave that source out of `index.malloy`'s
  `export { … }`, then delete the tag.
- **`##(agent-hidden)` at the top of a file:** import the file into `index.malloy` and
  export only the sources that should be entry points, then delete the tag.

One difference to raise before migrating: the tag hid a source but left it **queryable by
name**, and `index.malloy` makes it answer 404. If something relies on querying it by name,
confirm with the user first.

### `internal:` / `private:` vs `index.malloy`: different layers

- `internal:` and `private:` live **inside** a source's `include {}`. They decide which
  columns make up the source's public API.
- `index.malloy`'s `export { … }` decides whether the **source itself** is an entry point.

| Mechanism | Layer | Why you reach for it |
|---|---|---|
| `internal:` | Inside a source (one column in `include {}`) | The column **isn't part of your model's public API**. Common reasons: data is messy (empty/garbage, raw JSON, duplicates), or a documented derived dimension already supersedes it, or the raw column exists only to be joined on / referenced internally and shouldn't appear as a dimension callers can pick. The data may be perfectly fine; it's just not what you want exposed. |
| `private:` | Inside a source (one column in `include {}`) | The **data is sensitive**: SSN, raw credit card, password. Governance / security concern; harder block than `internal:`. |
| `index.malloy` `export { … }` | The whole source | The source should be reached through a curated entry point that extends or joins it, not landed on directly. |

In one sentence: **`internal:` shapes what's inside a source; `index.malloy` shapes whether
the source itself is an entry point.**

## Annotating Columns in Include (Experimental)

With `##! experimental.access_modifiers`, you can add `#(doc)` and `#(index)` tags to raw table columns inside `include` blocks. This documents columns without redefining them as dimensions.

```malloy
##! experimental.access_modifiers

source: orders is conn.table('orders') include {
  public:
    #(doc) Order line item identifier
    id

    #(doc) Customer email address
    email

    #(doc) Order status: pending, shipped, delivered
    status

  // internal: only for verified noise (empty cols, raw JSON blobs, duplicates)
}
extend {
  // ... dimensions and measures
}
```

**When to use:**
- Documenting raw columns without creating explicit dimensions
- Adding `#(index)` to columns for AI search — the decision ladder and the 5,000-value evidence gate apply here exactly as they do to declared dimensions. `#(index)` only does anything under `public:`; on an `internal:` or `private:` column it compiles and is ignored
- Curating which columns are public vs internal

## Source-Level Documentation

Document **when to use** a source, not what it contains. Credible AI can already search dimensions and measures directly — the source-level `#(doc)` should describe what questions/analyses this source answers.

**Base source files:** Document what the table represents.
```malloy
#(doc) Customer records with demographics and segmentation. One row per customer.
source: customers is conn.table('sales.customers') extend { ... }
```

**Source files:** Document what analytical questions the source answers.
```malloy
#(doc) Customer health analysis. Use for retention, segmentation, churn risk, and lifetime value. For order-level analysis, use order_analysis instead.
source: customer_health is customers extend { ... }
```

**Best practices:**
- Add `#(doc)` to all base source and joined source definitions
- Base source docs: describe what the table is (one row per what)
- Source docs: describe what questions/analyses the source answers
- Documentation happens per-source-file, not in one monolithic file

## Flag Ambiguous Descriptions

After writing `#(doc)` tags, present any that required judgment to the user for confirmation:

| Field | Proposed doc | Confidence | Uncertainty |
|-------|-------------|------------|-------------|
| `total` | "Total order amount in USD" | Medium | Could be gross or net — verified with sample query |
| `status` | "Order status: pending, shipped, delivered" | High | Values confirmed via `index: *` |

Only flag fields where the description required assumptions about business meaning or units.

Keep this table separate from the Step 4 `#(index)` table — this one confirms *wording*, that one records why a large index is worth having.

## Verify

After documenting, re-read each `#(doc)` tag and confirm it uses the terms a user would actually search for — the plain-language names and synonyms for the field, not just its column name.

## Done

Step complete. Output: `#(doc)` tags on all public fields and sources, plus `#(index)` on the dimensions that need it — each small one reported with its reason, and each one over the 5,000-value gate carrying recorded evidence that queries benefit.

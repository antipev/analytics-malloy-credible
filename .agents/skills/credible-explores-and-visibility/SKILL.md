---
name: credible-explores-and-visibility
description: How and when to curate which sources agents discover and can query, with a root `index.malloy` and Malloy `import` / `export`. Use when the user wants to hide base, staging, bridge, or scaffolding sources behind a curated entry point, control the discovery surface or query boundary, or asks about index.malloy, import, export, the deprecated `publisher.json` `explores` / `queryableSources` fields, or the legacy `#(agent-hidden)` tag. Applying any curation is OPTIONAL and must be proposed to the user and explicitly confirmed before changing files.
---

# Explores & Visibility

> Curating what agents see and can query is **opt-in and reversible**. A package with no
> root `index.malloy` lists every model and lets every source be queried, which is the
> correct, safe behavior for most packages. **Never add an `index.malloy`, an `export {}`,
> or a `publisher.json` curation key on your own.** Propose it, show what becomes hidden,
> and change files only after the user confirms.

## The mechanisms

| Mechanism | Layer | Declared in | Default | What it does |
|-----------|-------|-------------|---------|--------------|
| `import` | file to file | a `.malloy` file | none | Brings another file's sources into this file's namespace so they can be joined or extended. Always fine; it's how multi-file models compose. |
| `export { … }` | within a file | a `.malloy` file | exports all own top-level sources | Filters which sources the file exposes to importers and discovery. |
| root `index.malloy` | package | a file at the package root | absent, so everything is listed and queryable | **The package's surface.** Only this file is listed, and only what it exports can be queried. Everything else still compiles and stays importable and joinable. |
| `explores`, `queryableSources` | package | `publisher.json` | omitted | **Deprecated.** They still work. `explores` and `queryableSources: "declared"` warn at load; never add them to a new package. `queryableSources: "all"` does not warn and keeps one use (see below). |

This skill is the **decision framework** for whether and how to curate. If a package still
carries `explores` or `queryableSources`, its load warnings name the edit that replaces
each one.

## Golden rule: optional and user-confirmed

1. **Default to no curation.** A package with neither a root `index.malloy` nor an
   `explores` in `publisher.json` is uncurated, and that is correct unless the user wants
   otherwise. A root `index.malloy` or a non-empty `explores` already curates it, and hidden
   sources already answer 404: work with what is there rather than adding keys.
   A set `explores` wins. If it leaves `index.malloy` out, `index.malloy` is ignored. If it
   lists `index.malloy`, that file is on the surface next to the other listed files. To
   replace `explores`, carry its surface into `index.malloy`'s imports and `export { … }`
   (its deprecation warning names the files to import). Show the user what is published
   before and after, and delete `explores` in the same change once they confirm.
2. **Never curate as a side effect.** Do not add or edit `index.malloy` while modeling,
   documenting, or publishing. Curation is its own decision with its own confirmation.
3. **Propose, then confirm.** When curation is warranted, present:
   - the proposed `index.malloy` (its imports and its `export { … }`),
   - the exact list of files and sources that become **hidden from discovery**,
   - the sources that become **not directly queryable (404)**.
   Write files only after the user explicitly approves.
4. **State the trade-off.** Everything reads only what `index.malloy` exports: agents,
   dashboard tiles and filters, notebook cells, and a direct query by name. Before hiding a
   source, confirm nothing the user relies on reads it. A dashboard tile over a hidden
   source answers 404, and the package load warns about it.

## When to curate, and when not to

**Curate** when the package has a clear internal/external split:
- Raw, staging, bridge, or scaffolding sources exist only to build a curated entry point.
- Several files each define a source, and you want agents to land on the joined analysis
  sources, not the bases.

**Don't curate** when:
- The package has one source, or every source is a legitimate entry point.
- You're still exploring or iterating. Curation is a finishing step, not a starting one.
- The goal is *access control by caller*. That's `#(authorize)`, not curation (see below).

## How to apply (after confirmation)

Add an `index.malloy` at the package root that imports the files and exports the entry
points:

```malloy
// index.malloy
import "order_analysis.malloy"
import "customer_health.malloy"

export { orders, customer_health }
```

- The surface is what this file exports: `orders` and `customer_health` are listed and
  queryable, and every other source is hidden and answers 404 by name.
- Queries address `index.malloy`. A query to any other model file returns 404. Dashboard
  and notebook files are the exceptions: each is also a query path, and it reads only what
  `index.malloy` exports.
- A join the model already declares on an exported source still reads the hidden source
  and returns its fields. A join you write in the query to a hidden source is refused with
  a 404, like a direct query. If a hidden source's contents must not be readable even
  through a declared join, gate it with `#(authorize)`.
- One `index.malloy` can import any number of files, so a surface spanning several files
  needs no `publisher.json` key.
- A hidden source still works underneath: an exported source can join or extend it, and
  its fields stay findable and readable through that join.
- Dashboards and notebooks are always listed. What they may read is the surface, plus any
  source they declare on top of an exported one. A dashboard file's own `export { … }` adds
  nothing to the surface. Never tag `index.malloy` (it is never a report) and never import a
  report into it: a tagged file on the surface takes its sources off it (`report-on-surface`).

### Givens come through the imports

A caller can set a `given:` only if `index.malloy` has it in scope. You never list givens in
`export { … }`, and you never export a raw source to get its givens across. The import
decides it:

- **Import whole files.** `import "order_analysis.malloy"` brings every given that file
  declares. Then `export` only the curated sources. This is the default to write.
- A selective `import { orders } from "order_analysis.malloy"` brings only `orders`, and its
  givens stay behind. If you use one, name the givens too:
  `import { orders, REGION } from "order_analysis.malloy"`.
- Imports don't chain. If a source `index.malloy` exports reads a given that its model file
  takes from a `givens.malloy`, import `givens.malloy` into `index.malloy` as well. A given no
  exported source reads, such as one only a dashboard or notebook declares or imports for
  itself, needs nothing here.

What a missed given does depends on how it is declared:

- **With a default**, the source runs on the default and callers cannot change it. Nothing
  errors, so this one is easy to ship by mistake.
- **With no default**, every query on the source is refused with
  `404 Query target is not queryable`, although the source is exported.
- **Read by an `#(authorize)` or `#(access_filter)` gate** (including `$GROUPS`), the
  package refuses to load. Credible cannot inject `$GROUPS` into a model that does not
  declare it, so this fails closed.

### Undoing it

To make a package uncurated again, rename `index.malloy` and point every file that imports
it at the new name first: a broken import fails the whole package's compile. Don't use
`"explores": []` to opt out; it is deprecated.

### import syntax (reference)

```malloy
import "orders.malloy"                          // whole-file: brings in all its sources
import { orders, buyers is customers } from "..."  // selective: only these, optional rename
```

See [Malloy: Imports & Exports](https://docs.malloydata.dev/documentation/language/imports).

## Relationship to other visibility tools

- **`#(agent-hidden)`** is replaced by `index.malloy`. Never author it. If an existing model
  carries it, see `.claude/skills/credible-document/SKILL.md` for how to migrate.
- **`#(authorize)`** is access control by **caller identity or role**. `index.malloy` decides
  *what* is discoverable and queryable by name, not *who* may query. If contents must be
  protected, use `#(authorize)`, not curation.
- **`queryableSources: "all"`** keeps the listings `index.malloy` curates, but makes every
  source in the package queryable by name again, hidden ones included. Dashboards and
  notebooks are then no longer limited to what `index.malloy` exports either. Its one
  remaining use is hiding an `#(authorize)`-gated source from listings while authorized
  callers still query it by name. Because it reopens every hidden source, set it only after
  the user confirms that trade. Leave it out otherwise.

## Verify (after publishing with curation)

- The package's model listing shows only `index.malloy`. Dashboards and notebooks are
  still listed.
- A direct query against a hidden source returns 404.
- The package's load warnings name no dashboard tile that "won't load" and no filter whose
  "list will be empty".
- Consumers that should still work (curated entry points, notebooks, dashboards) do.
- Every given a published source reads shows on that source in `get_context`. A given
  missing there means `index.malloy` does not import it.

Note: a draft query (`execute_query` with `live: true` in the app, `execute_query_draft` in an
IDE) compiles the draft's files directly and **bypasses** the serve-time
listing and query boundary, so verify curation against the **published** package, not
locally.

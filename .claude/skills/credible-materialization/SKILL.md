---
name: credible-materialization
description: How materialization behaves on Credible - where a persisted table is built (the source's own warehouse, or Credible-managed storage with storage=credible), building one stored rollup over another, what happens when a package is published, how to tell whether a build succeeded, freshness and refresh, first-build and first-query cost, and why a draft query never reads a materialized table. Use whenever the user wants to materialize or persist a source on Credible, move a package to Credible-managed storage, chain rollups onto stored tables, asks why a materialized source is still slow or a build failed, or asks how fresh a materialized table is.
---

# Materialization on Credible

How to write a persist source - the `#@ persist` annotation, the `##! experimental.persistence` flag, which sources can be persisted, `storage=`, `partition=`, chaining, freshness and the `materialization` block in `publisher.json` - is in `.claude/skills/malloy-materialization/SKILL.md` and its reference files. This skill covers what is different when Credible runs the build: where tables go, when they are built, what the user can see, what the agent cannot, and the two ways a build fails here that the shared skill does not describe.

## Where the table is built

**In the source's own warehouse (the default, no `storage=`).** The connection must be able to write where `name=` points; `name="analytics.orders_daily"` puts the table in the `analytics` dataset or schema, which must already exist. Credible chooses the physical table name itself, shaped like `orders_daily__g000__1a2b3c4d` (with `__v0_1_6` before the `__g` part when the package uses `"scope": "version"`). Never hardcode that name anywhere; the model keeps referring to the source.

**In Credible-managed storage (`storage=credible`).** The table lives in storage Credible runs, so the connection needs no write access to the warehouse, and queries served from the table do not run in the warehouse at all.

```malloy
#@ persist name="orders_daily" storage=credible
source: orders_daily is orders -> { group_by: order_date; aggregate: order_count is count() }
```

- **`name=` must be a single undotted name**, still quoted (`name="orders_daily"`). Placement in managed storage is Credible's, so a dotted name fails the build: `combines storage="credible" with a dotted name=...`. Moving an existing package onto managed storage therefore renames every target, and every table is built from scratch (see **First builds cost the most**).
- **BigQuery connections need one more role.** The build reads results out of BigQuery through the Storage Read API, so the connection's service account needs **BigQuery Read Session User** (`roles/bigquery.readSessionUser`) on the project the connection runs queries in. In-warehouse builds don't need it, and nothing checks for it ahead of time: without it the build fails with a permission error naming `bigquery.readsessions.create`. Granting it is a job for whoever administers that Google Cloud project; once it is granted, the user can press **Rebuild** without republishing.
- **If managed storage is not set up in this environment**, the build fails with `the managed store is not available in this environment ... contact Credible`. That is not a model problem. Meanwhile you can offer to drop `storage=` and build in the warehouse instead; that is the user's call, because the connection then needs write access to the target and every table is rebuilt there.

**DuckDB sources are never materialized on Credible**, with or without `storage=`. They are served live, and the version shows **Unsupported** for its build.

## Chaining stored tables on Credible

A `storage=credible` source may read other `storage=credible` sources, directly or through the unpersisted sources between them, and is then built **from their stored tables** rather than from the warehouse; a public source defined over a stored table, with or without `include { public: ... }`, is served from it. The rules that decide whether a chain holds (declaration order, a distinct intermediate, joins onto stored siblings) are in `.claude/skills/malloy-materialization/SKILL.md`'s chaining reference. Two consequences are Credible's own:

- **A bare `extend` of a stored source is refused as a name collision.** `scope_live is site_seasons_base extend { ... }` inherits the parent's persist tag, so two sources now name one table, and Credible rejects the whole build before it starts: `Persist name collision: sources 'site_seasons_base' and 'scope_live' both materialize to table 'pfw_site_seasons'`. Make the intermediate distinct with a pass-through query: `scope_live is site_seasons_base -> { select: * } extend { ... }`.
- **A chained build runs in Credible's own engine under a fixed memory bound** (well under a gigabyte today), not in the warehouse. A grouped rollup over stored tables fits. A self-join or a wide hash aggregate over a fact of tens of millions of rows does not, and fails on every attempt with `Out of Memory Error: could not allocate block ...`; the automatic retries fail the same way. Keep that one rollup's input **unpersisted** so it builds in the warehouse and only its result is copied into the store. Everything else in the package can still chain.

**Nobody can see whether a chained source read the parent's table or recomputed it.** Credible does not keep the `upstreamReuse` field the shared skill describes; the only evidence is the build's duration (a chain over stored tables builds in minutes where a warehouse rebuild took much longer) and the warehouse's own job history. Say so rather than claiming reuse.

## What happens when a package is published

Publishing starts the build automatically, and indexing alongside it. The build is best-effort: **the publish succeeds even when the build fails**. Credible checks some persist rules only when the build starts, so these show up as a failed build, not a failed publish: a dotted name with managed storage, an unsupported `storage=` value, two sources writing to one table name (including the inherited-tag case above), and a schedule that breaks the scheduling rules.

- **Until the build finishes, queries compute live.** A version published from chat becomes the latest straight away, so the first queries after a publish are slow but correct. Judge speed, and anything that depends on the tables, only after the build is done.
- **The agent cannot see build status.** No tool reports it. Ask the user to open the package page's **Builds** section (visible to people who can update the package): a build reads **Building**, then **Done**, **Done, with failures** or **Failed**, and its page shows the error text verbatim. Per source, the model explorer shows **Materialized**, **Materialization failed** or **Refused**, and a source's detail shows the physical **Table** it reads.
- **A version whose build was rejected still gets indexed**, and that indexing appears in the Builds section as its own row labelled **Whole package** with no build number. It is not a second build and builds no tables; the materialization is the row with a build number.
- **A version id is published once.** To fix a failed build, change the model and publish the next version (`publish_package` suggests it), or have the user press **Rebuild** on the Builds section if nothing in the model needs to change. Rebuild recomputes every table of that version even when nothing changed, and the current tables keep serving until the new ones are ready, so it is also the way to refresh on demand.
- **A failed source is retried automatically: immediately, then after 20, 40, 80 and 160 minutes**, five attempts in all over about five hours, each one a run of its own in the Builds section. That rescues a timeout or a busy worker. It does not rescue a model error or a resource failure (the out-of-memory case above): those fail identically every time, and the retries hold a worker for the duration of each attempt. Fix the model in the next version, and have the user **archive** the failing version so the retries stop; a superseded version is retried just the same until it is archived.
- **Republishing unchanged persist logic reuses its table** (with the default package scope), so a new version is not by itself a refresh. Unchanged sources are reused in seconds; only the changed ones build.

## Freshness and refresh

**With no `materialization` block, a table is built at publish (or on Rebuild) and never refreshed.** To keep it current, declare a `freshness` window (the syntax is in `.claude/skills/malloy-materialization/SKILL.md`); Credible then rebuilds the table to stay within it. What is specific to Credible:

- **Write the window as a whole number and one unit** (`"24h"`, `"90m"`, `"7d"`). Anything else (`"1d12h"`) is accepted at publish and then ignored without an error, which means no refresh at all.
- **`"fail"` currently behaves like `"live"`**: a stale table is skipped and the query computes from the source, although the app describes it as failing the query. Nothing makes a stale query error today.
- **A `schedule` turns off per-source rebuilds** for that version, and an invalid cron is ignored without an error.
- Pick the window from how often the data changes, not how fast you want queries: every refresh rebuilds the table, and a chained rollup is rebuilt from the stored roots, not from the warehouse.

## Draft queries never read a materialized table

Drafts are never built. A query on the draft (`execute_query` with `live: true` in the app, `execute_query_draft` in an IDE) always computes from the sources, even when a published version of the same package has built tables. To see materialized behavior - speed, and the rows a stored table holds - query the published package (`execute_query` without `live`) after its build is done.

**Neither the app nor the agent can see whether one query was served from a table**: Credible does not pass on the `servedFrom` field or the generated SQL that `.claude/skills/malloy-materialization/SKILL.md` uses for this. The evidence available is the build status above, the source's **Materialized** state with its table name, and the published query's speed. Tell the user what you checked rather than claiming more.

## First builds cost the most, and first queries cost too

A source's first build, and every build after its SQL changes, reads all of its input.

- **In-warehouse BigQuery builds run under the connection's limits**: its **Maximum Bytes Billed** and **Query Timeout**, which default to 25 GiB and 10 minutes when left empty. A bigger build fails; raise them on the connection before publishing a large package.
- **Every build has a two-hour ceiling.**
- **Move large sources onto managed storage one at a time**, across successive versions. Each moved source is a full cold build, and many at once compete for the same build capacity.
- **Re-pointing a project connection renames and rebuilds** the project's materialized tables.
- **The first queries on a freshly built version are cold on every worker.** A page that fires a dozen queries at once reads several times slower than the same page a minute later, even on sources whose tables did not change. Judge a materialization's speed on the second pass, not the first.
- **A rollup that every query filters to one key is read whole per query** unless it is partitioned on that key (`partition="species_code"` and the like; the shared skill's storage reference says when). That is the usual reason a stored rollup still answers in seconds.

## Cleanup

Unlike a standalone Publisher, where cleanup is yours, Credible removes in-warehouse tables that no remaining version uses, when versions are archived or the package is deleted, and, for packages created since that default was introduced, archives a version 30 days after it stops being the latest (older packages keep the policy they had). Don't drop materialized tables by hand: a table that looks orphaned may still serve a version. Tables in Credible-managed storage are Credible's to manage; there is nothing to drop. Archiving a version (from the package page, or `cred archive <package> <version>`; reversible with unarchive) is also how to stop a version that keeps retrying a build that cannot succeed.

---
name: credible-publish
description: Publish Malloy models to Credible platform, grant people access via the in-chat sharing panel, and curate what agents see and can query with a root index.malloy. Use when user asks to "publish", "deploy", "share models with others", "share this package with <person>", "give <teammate> access", or wants to control which sources agents discover or can query directly — e.g. "only expose my curated source", "hide the raw/staging source from the listing", "stop agents querying the base source", or asks about the deprecated `explores` / `queryableSources` fields in publisher.json.
---

# Publishing to Credible

> **🛑 CRITICAL: Only publish when the user explicitly asks.** Making model changes, adding documentation, or optimizing for AI is NOT a publish request. Never auto-publish after completing other tasks.

## Prerequisites

- Malloy model and/or report files (all `.malloy`) ready
- `publish_package` MCP tool available

## Step 1: Verify publisher.json

Check if `publisher.json` exists in the package root. If it does, proceed to Step 2.

If it doesn't exist, create one — pick a package name based on the model content, write a brief description, and default to version `0.0.1`.

```json
{
  "name": "package-name",
  "version": "0.0.1",
  "description": "Brief description of the package"
}
```

**Naming conventions:**
- `name`: lowercase, hyphens allowed (e.g., `ecommerce`, `sales-analytics`)
- `version`: semver format (e.g., `0.0.1`, `1.2.3`)
- `description`: one line naming what the package covers (e.g., `"Orders, customers, and revenue analysis"`). It's a catalog label, not a changelog or spec — keep it to a sentence. Do **not** pack in row counts, model inventories, algorithm parameters, or per-version release notes; that history belongs in the model files, `_plan/`, and the commit/PR, not here.

### Curating discovery & the query boundary (optional)

A package exposes everything by default, and that is the right default. Curating it
with a root `index.malloy` is the user's decision and its own confirmation, never a side
effect of publishing: follow `.claude/skills/credible-explores-and-visibility/SKILL.md`.

## Step 2: Publish via MCP Tool

Call the `publish_package` MCP tool. Follow the active tool schema for the exact input shape.

In the Credible app it is **two calls with the user's confirmation in between**: call it without `confirmed` and nothing is published — it returns a preview (the name, version and description it would publish, the files, the versions already published, and a suggested next version when yours is taken). Show the user that preview; only after they confirm, call it again with `confirmed: true` (and the agreed `version` if it differs from `publisher.json`).

```
Call: publish_package
```

**What it does:**
- Publishes the package contents to Credible
- Sets the version as latest when supported by the current tool

## Step 3: Offer to share with people (optional)

Publishing makes the package **consumable**; sharing decides **who consumes it**. **Only if the `show_package_sharing` tool is available** in this chat — after a successful publish, or whenever the user asks to "share this with `<person>`" / "give `<teammate>` access" — open the private sharing panel:

```
Call: show_package_sharing  { "package_name": "<name>", "environment": "<environment>" }
```

- `package_name` (required) — the name you just published.
- `environment` (optional) — defaults to the chat's environment. The org/env come from chat context; the package is referenced by name.

> **🛑 You do not grant access yourself.** The panel is a private surface between the user and Credible: the user picks who to share with, and for someone not yet in the organization the panel adds them — or, if the user isn't an org admin, requests that an admin approve them. Adding a person to an organization is a substantial, human-only action, so the agent never performs it and never sees who was added.

After calling, **END YOUR TURN.** When the user finishes you'll receive a short, privacy-safe summary of what was shared; they may also ask questions first. Then confirm and continue.

Use this to grant **people** access. To just hand someone a URL to an already-shared package, give them the package's `view_url` link instead — no panel needed. (If the `show_package_sharing` tool isn't available in this chat, the `view_url` link is the way to share.)

## Package Structure

`.malloy` files can live at the package root or in subdirectories — the publisher supports cross-directory imports. `import` paths resolve relative to the importing file (e.g. a sibling is `import "orders.malloy"`; across directories, `import "../sources/orders.malloy"`). Create new files at the package root unless the user asks for a folder.

```
<package-name>/
  publisher.json
  _plan/                        # Decision artifacts (kept as documentation)
  customers.malloy              # Base source file
  orders.malloy                 # Base source file
  user_order_facts.malloy       # Computed source
  order_analysis.malloy         # Source file (joins base sources)
  customer_health.malloy        # Source file
  monthly_report.malloy         # Report: a notebook or dashboard (optional)
```

Publishable contents:
- `.malloy` files - Semantic model definitions (base sources + joined sources)
- Report `.malloy` files - notebooks and dashboards, with a model-level `## artifact` tag (`layout=linear` or `layout=grid`), anywhere except `public/` (format: `.claude/skills/malloy-reports/SKILL.md`). A legacy `.malloynb` still publishes and renders
- Data files (CSV/Parquet) - Embedded data published with package

## Reports compile as their own model

A report compiles with every `import` in scope, so it can import and join sources from several `.malloy` files with no re-export model (`.claude/skills/malloy-reports/SKILL.md`). When the package root holds an `index.malloy`, its tiles read only what that file exports (`.claude/skills/credible-explores-and-visibility/SKILL.md`). Do not write a `.malloynb`: in a Credible draft one is read-only, and the agent converts it to a `.malloy` notebook beside it on request (`layout=linear` with `tiles=[…]`, prose as text tiles, never `run:` cells).

## Git-backed environments

When an environment models in git, a package that a merge has already published can't be
republished by upload — not from `publish_package`, and not from `cred publish`. The refusal is a
409 with code `GIT_BACKED_PACKAGE_REQUIRES_DRAFT` (a publish) or
`GIT_BACKED_PACKAGE_REQUIRES_GIT_VERSION` (promoting a version with no git commit behind it).
**Nothing was published.**

- **Don't retry, and don't bump the version in `publisher.json`.** Neither gets through; the
  refusal is about where the version came from, not its number.
- **The way forward is a pull request into the environment's base branch** — merging it is what
  publishes. The error names the repository and base branch, and its `details` carries them as
  `gitRepositoryOwner`, `gitRepositoryName` and `gitBaseBranch`. In the app, use
  `create_pull_request`; outside it, clone that repository and open the pull request there.
- **Or edit the package as a draft in Credible.**
- If the message says the environment **no longer has a git repository recorded**, an
  organization admin has to restore it before anything can publish. Tell the user; no tool you
  can call fixes it.

## Version Management

- Each publish creates an immutable version
- "Latest" pin determines default version for consumers
- A new version serves at once, but it becomes the default (what a query with no `version` reads, and what `get_context` finds) only after indexing, which takes minutes. Pass `version` to read it sooner
- Publishing needs a change. If nothing changed since the last publish, say it is already published at version X and ask what should change
- All versions remain available indefinitely
- Bump the version in `publisher.json` before every republish — versions are immutable, so republishing an existing version fails; bumping up front avoids a guaranteed failed attempt
- Don't append release notes to `description` when bumping a version — the description stays a brief catalog label across versions. Rewrite it only if what the package *covers* actually changed; version-to-version history lives in the changelog/PR, not the description.

## Workflow

1. Verify `publisher.json` exists; if not, create it (pick a name from the model content, default `0.0.1`)
2. If this package has been published before, bump the patch version in `publisher.json` first
3. Call `publish_package` (in the app: preview first, confirm with the user, then `confirmed: true`; if the preview or publish says the version already exists, bump to the next version and retry). In a git-backed environment a package a merge already published can't be republished from here, and no version bump gets around it: the change goes through a pull request (`create_pull_request`), and the merge publishes it
4. Confirm deployment succeeded, share link to package on Credible
5. If the `show_package_sharing` tool is available and the user wants teammates to use the package, call it to open the private sharing panel (Step 3) — you don't grant access yourself

## Common Issues

- **MCP tool not available**: Ensure the Credible extension is installed and the MCP server is enabled in IDE settings
- **Version already exists**: Bump the patch version in `publisher.json` and retry
- **"Nothing was published" with a `GIT_BACKED_PACKAGE_…` code**: the package publishes from git — see Git-backed environments. Bumping and retrying will not help here
- **A dashboard or notebook is missing after publish, or a `report-kind-removed` finding**: its `## artifact` tag still sets `kind=`, which Publisher no longer reads. Write `layout=grid` for `kind=dashboard` and `layout=linear` for `kind=notebook` (`.claude/skills/malloy-reports/SKILL.md`).

## Reports

A dashboard or notebook saved as a report is `save_report`, covered by `.claude/skills/credible-reports/SKILL.md`, not by publishing.

## Done

Step complete. Output: package published to Credible platform.

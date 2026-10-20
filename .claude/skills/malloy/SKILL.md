---
name: malloy
description: Index of all Malloy skills. Use when user asks "malloy help", "what malloy skills are available", "how do I use malloy", or needs guidance on which Malloy skill to use.
---

# Malloy Skills Index

## First-Time Setup

**MCP tools not available?** Search for and read `**/credible-setup/SKILL.md` to configure your IDE.

**No .malloy files in workspace?**
Say "model my data" and the agent will orchestrate the full 9-step workflow automatically.

## Skill Reference

| Skill | Use when... |
|-------|-------------|
| `/credible-setup` | IDE setup, MCP configuration, login, database connection — run when tools aren't available |
| `/malloy-discover` | Step 1: Silent data discovery — tables, schemas, distributions, prior art detection |
| `/lookml-review` | Prior art adapter for LookML — coordinates reference files for field extraction, derived tables, visibility, docs |
| `/malloy-scope` | Step 2: Present findings, propose analytical focus |
| `/malloy-define` | Steps 3-4: Propose source plan and field definitions |
| `/malloy-model` | Steps 5-8: Writing fully documented base and joined source .malloy files, review, curate |
| `/malloy-analyze` | EDA exploration, building views, dashboards, notebooks (separate from modeling) |
| `/malloy-charts` | Chart selection guide and renderer reference for all Malloy visualization types |
| `/malloy-debug` | Fixing compile errors, understanding error messages |
| `/malloy-patterns` | Finding syntax/pattern docs: YoY, cohorts, percent-of-total, rendering, window functions |
| `/credible-document` | Adding `#(doc)` and `#(index)` tags to an existing model that lacks documentation |
| `/credible-publish` | Publishing models to Credible platform |

> **Adapter pattern:** Each prior art adapter (LookML, future dbt) follows the same structure: a coordinator SKILL.md + reference files under `reference/` dispatched by phase skills.

## Two Paths to a Model

Both paths produce the same result: a fully documented, indexed, publishable semantic model.

**Path A — Schema-first:** "Model my data" → 9-step workflow (see `.cursor/rules/malloy-always.mdc` for full details)

**Path B — Analysis-first:** "Explore this data" → `/malloy-analyze` → formalize via `/malloy-model` (`reference/analysis-to-model.md`) → publish

After EDA, always recommend formalizing into a model.

## Views, Dashboards, Notebooks

- Use `/malloy-analyze` for EDA, views, dashboards, notebooks
- Use `/malloy-charts` for chart type selection and renderer reference

## Syntax Help

Call `searchMalloyDocs` with your question. Use `/malloy-patterns` to discover available topics.

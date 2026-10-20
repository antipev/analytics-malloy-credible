---
name: credible-data-apps
description: Use when an in-package HTML data app will run embedded inside Credible — same-origin auth, the sandboxed viewer and resize contract, and handing off to the Credible Agent with the credible:chat-message and credible:chat-context events. Read when building or modifying a data app that runs in Credible.
---

# Data Apps Inside Credible

> Credible runs a package's `public/` HTML app in a sandboxed, same-origin iframe and gives it a one-way channel to the Credible Agent. Running here means *less* code, not more: auth, listing, and resizing are handled for you. This skill is the 1P companion to `.claude/skills/malloy-html-data-apps/SKILL.md` (generic Malloy Publisher, including generic embedding).

## What Credible handles for you

- **Auth just works.** Credible serves the embed same-origin behind a first-party session cookie, so plain `Publisher.query(...)` authenticates as the signed-in user. Do NOT add Auth0, `setToken`, or token plumbing — that is only for a standalone Publisher or a cross-origin embed.
- **It is listed automatically.** A package shipping a `public/` directory is discovered and shown in the workspace "Data Apps" section. You register it nowhere.
- **Resizing is automatic.** The Credible viewer mounts your app in a sandboxed iframe and resizes it from the `publisher:resize` height your page posts — but only after validating the message's origin and source, then clamping the height (120–20000px). Your only job is to load `/sdk/publisher.js`; it posts the height for you.

This is the *runtime* — the embed runs in deployed Credible environments (and locally and in CI). It is not authoring: there is no watch or live-reload in Credible. Authoring the files is a separate, local-only Publisher loop covered in `.claude/skills/malloy-html-data-apps/SKILL.md`.

## Design for the sandbox

The iframe is sandboxed (`allow-scripts allow-same-origin allow-forms`). Build for that:

- **No top-level navigation and no popups.** Drive everything in-page.
- **Don't size from the viewport.** The viewer measures your content's bottom edge, not the window, to avoid a grow-forever loop. Let content define height; never rely on `body { min-height: 100vh }`.
- **Everything under `public/` is strictly first-party code.** It runs with the viewing user's data authority, so do not load untrusted third-party scripts and do not send query results to other hosts. (This is *why* `.claude/skills/malloy-html-data-apps/SKILL.md` has you vendor chart libraries into `public/`.)

## Hand off to the Credible Agent

A data app can push the user into the Credible Agent's chat without leaving the page, over a one-way `postMessage` channel. There are two events; you may send either or both.

**Post to the host's SPECIFIC origin, never `"*"`.** The host accepts a message only when both its origin and its source window match the embedded frame, so a wildcard target is both unsafe and ignored. Derive the app-plane origin from your own data-plane URL (swap the `data` plane label for `app` within the host) and only post when you are actually inside the Credible viewer (in an iframe AND on a recognized data-plane host).

| Event `type` | Payload | Effect | Limits |
|---|---|---|---|
| `credible:chat-message` | `{ type, prompt }` | Opens the chat panel and **auto-sends** `prompt` as the user's turn | `prompt` trimmed, non-empty, ≤ 4000 chars |
| `credible:chat-context` | `{ type, context }` | **Silently** attaches context to the turn — the agent reads it in its system prompt, but it **never appears in the transcript** (only a small "Context: …" chip is visible). Rides every turn until you post a new one. | `context` is the hybrid object below (a bare `{ label?, value }` is still accepted). Hard cap: `JSON.stringify(context)` ≤ **8000** chars or the whole message is **dropped**. When rendered, each value is flattened to one line and truncated to **1000** chars (whole block to **4000**) — keep any single value well under 1000. |

Send `chat-message` to ask a question; send `chat-context` to tell the agent **what the user is looking at** — the view, the model sources, and (highest-leverage) the exact queries behind the numbers. Post the **context before the message** so it's attached to the turn the message starts. The context is advisory: it *informs* the agent, it does not restrict its tools or scope.

```js
// Derive the app-plane origin from this data app's own URL. No hardcoded host.
function appOrigin(loc = location) {
  const parts = loc.host.split(".");
  // Anchor the swap to the END of the plane label (/data$/), not a bare
  // .replace("data","app"). A substring replace mangles any env label that
  // contains "data" mid-string (e.g. "mydata-data" → "myapp-data" instead of
  // "mydata-app"), so the derived origin is wrong and every postMessage is
  // silently dropped by the host origin check. This fails closed and is
  // invisible without a fresh-eyes review — anchor it.
  if (parts.length >= 3) { parts[1] = parts[1].replace(/data$/, "app"); return `${loc.protocol}//${parts.join(".")}`; }
  return `${loc.protocol}//${loc.host}`;
}
// Only inside the Credible viewer: in an iframe AND on a Credible data-plane host.
// Match the plane label exactly ("data" or "<env>-data"), not a substring — or a
// lookalike host like foo.database.example.com would pass and you'd post rows to it.
function inCredibleViewer(loc = location) {
  const parts = loc.host.split(".");
  const plane = parts[1] || "";
  return window.parent !== window && parts.length >= 3 && (plane === "data" || plane.endsWith("-data"));
}

function askCredibleAgent(prompt, context /* optional; the hybrid shape below */) {
  const text = String(prompt ?? "").trim().slice(0, 4000);
  if (!text) return "noop"; // host rejects empty prompts — don't claim success
  if (!inCredibleViewer()) {
    // Standalone fallback: copy the question and open the package page in a new tab.
    // Use the full current URL — location.origin would drop the package path.
    try { navigator.clipboard?.writeText(text); } catch {}
    window.open(location.href, "_blank", "noopener");
    return "fallback";
  }
  const origin = appOrigin();
  if (context) {
    window.parent.postMessage({ type: "credible:chat-context", context }, origin);
  }
  window.parent.postMessage({ type: "credible:chat-message", prompt: text }, origin);
  return "embedded";
}
```

### The context shape

`context` is hybrid — send what you have. All fields are descriptive (they inform the agent; none restrict its tools or scope):

| Field | Meaning |
|---|---|
| `scope` | `{ project, package, version, model }` — where in the catalog the view lives |
| `view` | `{ type, id, title }` — what's on screen now (`title` shows in the chip) |
| `entities` | `[{ kind, name }]` — the model sources/views the user is looking at |
| `items` | `[{ label, value }]` — the **actionable specifics** (see below); the highest-leverage field |
| `givens` | the user's current filter selections, e.g. `{ region: "EMEA", segments: ["smb"] }` |

`givens` are surfaced to the agent as **awareness** ("Filters: region=EMEA"), **not** enforced as a query filter — so "compare EMEA to APAC" still works even when the dropdown says EMEA. There is no hard data-app filter channel today.

```js
askCredibleAgent(
  "In okrs_4.malloy, why is OKR 4.1.1 below target? Name the main driver.",
  {
    scope: { project, package: pkg, version, model: "okrs_4.malloy" },
    view: { type: "tile", id: "4.1.1", title: "New Digital Customers" },
    entities: [{ kind: "source", name: "okr_4_1_1_new_digital_customers" }],
    items: [
      { label: "actuals query", value: "run: okr_4_1_1_new_digital_customers -> tile" },
      { label: "target query",  value: "run: okr_4_1_1_targets -> { select: target_month, target_value; order_by: target_month }" },
      { label: "likely main driver", value: "split actuals by activation_type ('Starter' vs 'Clicks')" },
    ],
  },
);
```

### Put the exact queries in `items` — naming the source is not enough

Listing the backing source in `entities` and stopping there still makes the agent burn many turns (measured: ~15 → a handful on the `okr_4_fy27` 4.1.1 tile) rediscovering what the app already knows: that the named thing is a *source* not a view; that `tile` is a *runnable view* on it (in a prompt, ``a source's `tile` `` flattens to plain text and reads as an English word, so the agent doesn't know to run it); and the target query's shape. So hand over the literal queries the tile already ran, as `items`:

- **`actuals query` / `target query`** — the exact `run: <source> -> <view>` strings, pulled from your tile-definition data (the same strings a "show source" viewer would display) so they can't drift. The agent runs them verbatim instead of reconstructing them.
- **`likely main driver`** — name the non-time dimension a "why off target?" split should use (e.g. `activation_type`). Sends the agent straight to the driver. Omit it when there's no useful categorical breakdown rather than inventing one.

For a multi-KPI section view, prefix each `label` with the KPI code (`"4.1.1 actuals query"`) so the folded queries stay attributable. Keep values single-line — the host flattens newlines to spaces anyway.

### Prompt discipline (so the agent answers well)

The agent answers best when the prompt does the grounding the agent would otherwise spend turns discovering:

- **Name the exact model file and the exact source/view names** the numbers came from. Read them from your app's own tile definitions (the single source of truth from `.claude/skills/malloy-html-data-apps/SKILL.md`, `reference/runtime.md`) — never fabricate or mechanically derive a source name. A guessed `..._targets` source the agent then can't find wastes its turns and breaks trust.
- **State the question answer-first and bounded:** "lead with the answer in one or two sentences, then at most three short bullets; don't over-explore."
- **Attach the specifics, don't describe them.** When the answer depends on the exact queries or rows behind the numbers, send them as `chat-context` (the `items` / `entities` fields above) rather than pasting them into the prompt text — the agent reads the structured context, and the user's chat stays clean.
- **Keep the prompt self-contained anyway.** Context is *additive*: name the model + sources in the prompt too, so the handoff still works if the context is dropped (an older host, or the standalone fallback).

### Always provide the standalone fallback

The same app may be opened outside Credible (a standalone Publisher, a shared link). Every handoff trigger must degrade gracefully — copy the question to the clipboard and open the package page — so a missing parent or non-Credible host never makes the button silently do nothing.

## Before you ship: run a fresh-eyes review pass

Build-time headless checks (`.claude/skills/malloy-html-data-apps/SKILL.md` → `verification-harness.md`) confirm the app *runs*; they do not confirm it's *right*. A finished-app review with fresh eyes routinely catches what green checks miss — measured on real builds: a wrong `<title>`, stale/placeholder copy left in, dead code, a `NaN` render that only appears on certain data, and **fails-closed bugs the harness can't see**, like the `appOrigin` `/data$/` anchor above (a wrong origin drops every `postMessage` silently — there's no error to catch).

So after the app passes its headless checks, do a dedicated read-through (a fresh subagent, or a genuine second pass) against intent:

- **Title & copy** — `<title>`, headings, captions match this app (not a copied template).
- **The origin/handoff path** — `appOrigin` anchors `/data$/`; `inCredibleViewer` matches the plane label exactly; the standalone fallback actually fires.
- **No dead code, no `NaN`/`undefined`/fake-`0`** in any tile across the data it will actually see.
- **Assumptions surfaced in the UI**, not just in your head.

---
name: credible-setup
description: Set up IDE for Credible modeling. Use when MCP tools are unavailable, user asks to "set up", "configure MCP", "get started", "connect my database", "connect Snowflake/BigQuery/Postgres", or any prerequisite check fails.
---

# IDE Setup for Credible Modeling

Credible supports **Cursor**, **VS Code**, and **Claude Code** (running in a Cursor or VS Code terminal) as IDEs for building semantic models.

## Step 1: Detect IDE

Ask the user which IDE they're using: A) Cursor, B) VS Code, C) Claude Code (in a Cursor or VS Code terminal)

## Step 2: Install Credible Extension

> **Claude Code users:** The Credible extension must be installed and running in the IDE where Claude Code is open (Cursor or VS Code).

1. Open Extensions: **Cmd+Shift+X** (Mac) / **Ctrl+Shift+X** (Windows/Linux)
2. Search **"Credible"**
3. Install the Credible extension

The extension provides:
- **Modeling skills** — `/malloy-model`, `/malloy-discover`, `/credible-publish`, etc.
- **Credible Service Panel** — view and select environments and connections
- **Credible-Modeling MCP server** — warehouse metadata, schema search, query execution
- **Malloy extension** (installed automatically) — syntax highlighting, compilation checks, Schema/Explore/Preview buttons

The skills and MCP server work together to power modeling.

## Step 3: Login

### Cursor / VS Code
1. Open the Explorer panel (sidebar)
2. Find the **Credible** panel at the bottom of the sidebar
3. Click **Login**
4. Select your **organization** and **project**

> If the Credible panel is not visible, reload the window first (**Cmd+Shift+P** → "Reload Window").

### Claude Code
No login needed — authentication is handled by the Credible extension in the IDE. Skip to Step 5 (Enable MCP Server).

## Step 4: Enable MCP Server

### Cursor
1. Open Settings: **Cmd+Shift+J** (Mac) / **Ctrl+Shift+J** (Windows/Linux)
2. Navigate to **Tools & MCP**
3. Find **Credible-Modeling** and toggle it **on**

### VS Code
1. Open Command Palette: **Cmd+Shift+P** (Mac) / **Ctrl+Shift+P** (Windows/Linux)
2. Run **"MCP: List Servers"**
3. Find **Credible-Modeling** and click **Enable**

### Claude Code
1. Find the MCP server URL from the workspace config:
   - Read `.cursor/mcp.json` or `.vscode/mcp.json`
   - Extract the URL from `mcpServers.Credible-Modeling.url`
2. Run the command to register the MCP server:
   ```bash
   claude mcp add --transport http Credible-Modeling <url>
   ```
   Example (port varies per session):
   ```bash
   claude mcp add --transport http Credible-Modeling http://127.0.0.1:52214/mcp
   ```
3. Verify with `/mcp` command in Claude Code

> **If no config file exists:** The user needs to open the project in Cursor or VS Code with the Credible extension installed first. The extension creates the MCP config file automatically.

## Step 6: Verify Setup

Call `search_database_schema` to confirm everything is working.

**Tool available + returns data:** Setup is complete. Tell the user they're ready — suggest "say 'model my data' to begin."

**Tool available + no data:** MCP server is connected but no database connections are configured. See "Connecting Your Data" below.

**Tool still not available:**
1. Suggest reloading the IDE window (**Cmd+Shift+P** → "Reload Window" in Cursor/VS Code)
2. Re-check that the MCP server is enabled (repeat Step 4)
3. If still failing, check the Credible extension output panel for errors

## Troubleshooting

- **MCP tools disappear after switching projects** — Re-enable the MCP server (repeat Step 4)
- **No Credible panel in sidebar** — Reload the IDE window
- **Login fails** — Verify your org admin has set up your account at credibledata.com
- **`search_database_schema` returns empty** — See "Connecting Your Data" below

## Connecting Your Data

Only needed if `search_database_schema` returns no results or the user asks to connect their own data. Demo environments come with a pre-configured `bq_demo` connection — use `/malloy-discover` to explore those datasets.

To connect your own data:
- Visit **`<org>.app.credibledata.com`** (replace `<org>` with your organization name)
- Full setup docs: **https://docs.credibledata.com**
- Supported warehouses: Snowflake, BigQuery, Postgres, DuckDB, and more
- An org admin may need to configure this depending on permissions
- Credible automatically indexes the connection once added, making it available via `search_database_schema`

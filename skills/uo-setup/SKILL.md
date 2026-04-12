---
name: uo-setup
description: Retrieve the MCP connection config from the developer's running UOFastMCP instance and inject it into Claude Desktop, the Claude/VSCode extension, and the project .mcp.json — all in one step. Run this once after first login to UOFastMCP, or any time the server URL or auth token changes.
argument-hint: [--desktop] [--project] [--all]
allowed-tools: [mcp__UOFastMCP__get_mcp_config, Read, Write, Edit, Bash]
---

# uo-setup

Fetch the live MCP server config from UOFastMCP and inject it into every Claude config location on this machine.

## Arguments

The user invoked this with: $ARGUMENTS

Parse as: `[--desktop] [--project] [--all]`

- `--desktop` — inject into Claude Desktop's `claude_desktop_config.json` only
- `--project` — inject into the project's `.mcp.json` only (covers Claude Code CLI + VSCode extension)
- `--all` — inject into both (default when no flag given)

## Examples

```
/uo-setup
/uo-setup --all
/uo-setup --desktop
/uo-setup --project
```

## Prerequisites

The UOFastMCP server must be running and the developer must have completed login at `localhost:8000`. The tool `mcp__UOFastMCP__get_mcp_config` returns the server's current connection config for this session (URL, auth token if applicable).

---

## Steps

### Step 1 — Fetch live server config

Call `mcp__UOFastMCP__get_mcp_config` with no arguments.

The tool returns a JSON object in this shape:

```json
{
  "name": "UOFastMCP",
  "url": "http://localhost:8000/sse",
  "headers": {
    "Authorization": "Bearer <session-token>"
  }
}
```

The `headers` key is optional — only present when the server requires auth. Store this entire object; it becomes the value at `mcpServers.UOFastMCP` in every config file you write.

Build the `mcpServers` entry:

```json
{
  "mcpServers": {
    "UOFastMCP": {
      "url": "<url from tool>",
      "headers": { ... }    // omit entirely if not returned
    }
  }
}
```

### Step 2 — Detect platform and resolve config paths

Run this Bash command to get both platform and the Claude Desktop config path:

```bash
if [ -n "$APPDATA" ]; then
  echo "windows"
  echo "$APPDATA/Claude/claude_desktop_config.json"
elif [ "$(uname)" = "Darwin" ]; then
  echo "macos"
  echo "$HOME/Library/Application Support/Claude/claude_desktop_config.json"
else
  echo "linux"
  echo "$HOME/.config/Claude/claude_desktop_config.json"
fi
```

Also resolve the project `.mcp.json` path by running:

```bash
echo "$(pwd)/.mcp.json"
```

Store both paths.

### Step 3 — Inject into Claude Desktop config (skip if `--project` only)

Read the file at the Desktop config path. If it does not exist or is empty, start from `{}`.

Parse the JSON, then set or replace `mcpServers.UOFastMCP` with the entry from Step 1. Preserve all other keys in `mcpServers` and all other top-level keys untouched.

Write the merged JSON back to the same path with 2-space indentation.

**Important:** Claude Desktop must be restarted for config changes to take effect. Note this in the report.

### Step 4 — Inject into project `.mcp.json` (skip if `--desktop` only)

Read `.mcp.json` at the project root. If it does not exist, start from `{}`.

Set or replace `mcpServers.UOFastMCP` exactly as in Step 3. Preserve all other entries.

Write back with 2-space indentation.

This file is read by both the Claude Code CLI and the Claude VSCode extension — one write covers both.

### Step 5 — Report

Print a clear summary table:

```
UOFastMCP setup complete
─────────────────────────────────────────────────────
Server URL     : http://localhost:8000/sse
Auth token     : yes (Bearer) / no
─────────────────────────────────────────────────────
Claude Desktop : ✓  C:\Users\...\AppData\Roaming\Claude\claude_desktop_config.json
Project MCP    : ✓  .mcp.json
─────────────────────────────────────────────────────
```

If any write was skipped (because of a flag), show `–` instead of `✓`.

Remind the user:
- **Restart Claude Desktop** to pick up the Desktop config change.
- The project `.mcp.json` takes effect immediately for new Claude Code / VSCode sessions.
- Re-run `/uo-setup` any time the UOFastMCP server token changes (e.g. after server restart with new credentials).

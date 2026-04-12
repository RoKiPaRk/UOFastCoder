---
name: uo-setup
description: Log in to the developer's UOFastMCP server via /auth/login, retrieve the authenticated MCP connection config, and inject it into Claude Desktop (claude_desktop_config.json) and the project .mcp.json. Run once after starting UOFastMCP, or any time credentials change.
argument-hint: [--url <server-url>] [--desktop] [--project] [--all]
allowed-tools: [Read, Write, Edit, Bash]
---

# uo-setup

Log in to UOFastMCP, get the authenticated MCP config, and write it to every Claude config location on this machine.

## Arguments

The user invoked this with: $ARGUMENTS

Parse as: `[--url <server-url>] [--desktop] [--project] [--all]`

- `--url` — UOFastMCP server base URL (default: `http://localhost:8000`)
- `--desktop` — write to Claude Desktop `claude_desktop_config.json` only
- `--project` — write to project `.mcp.json` only
- `--all` — write to both (default when no target flag is given)

## Examples

```
/uo-setup
/uo-setup --url http://myserver:8000
/uo-setup --desktop
/uo-setup --project
```

---

## Steps

### Step 1 — Resolve the server URL

If `--url` was given, use that value. Otherwise default to `http://localhost:8000`.

Verify the server is reachable:

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/health
```

If the status code is not `200` (or the command errors), stop and tell the developer:
> UOFastMCP is not reachable at `<url>`. Make sure the server is running (`uofast-mcp`) and try again.

### Step 2 — Get credentials

Check environment variables first:

```bash
echo "${UOFASTMCP_USERNAME:-}"
echo "${UOFASTMCP_PASSWORD:-}"
```

If both are set, use them silently. If either is missing, ask the developer:

> Enter your UOFastMCP username:
> Enter your UOFastMCP password:

Do not log or display the password in any output.

### Step 3 — Authenticate via `/auth/login`

POST to the login endpoint:

```bash
curl -s -X POST "<server-url>/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"username": "<username>", "password": "<password>"}'
```

Parse the JSON response. The response will contain a token. Handle both common shapes:

| Field name | Use as |
|---|---|
| `token` | Bearer token |
| `access_token` | Bearer token (OAuth2 style) |
| `url` | Override SSE URL if present |
| `mcp_url` | Full SSE URL if present (use as-is) |

If the response contains an HTTP error status or an `error` / `detail` field, stop and show the error message to the developer.

### Step 4 — Build the MCP server entry

Construct the `mcpServers.UOFastMCP` block:

```json
{
  "url": "<server-url>/sse",
  "headers": {
    "Authorization": "Bearer <token>"
  }
}
```

- If the login response included a `url` or `mcp_url` field, use that as the `url` value instead of constructing it.
- If the response has no token (server runs without auth), omit the `headers` block entirely and just use `{ "url": "<server-url>/sse" }`.

### Step 5 — Detect platform and resolve config paths

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

Also resolve the project `.mcp.json`:

```bash
echo "$(pwd)/.mcp.json"
```

### Step 6 — Write Claude Desktop config (skip if `--project` only)

Read the file at the resolved Desktop config path. If it does not exist, start from `{}`.

Merge: set `mcpServers.UOFastMCP` to the entry from Step 4. Leave all other keys untouched.

Write back with 2-space indentation.

If the config directory itself does not exist, skip this target and note in the report that Claude Desktop does not appear to be installed.

### Step 7 — Write project `.mcp.json` (skip if `--desktop` only)

Read `.mcp.json` in the project root. If it does not exist, start from `{}`.

Merge: set `mcpServers.UOFastMCP` to the entry from Step 4. Preserve all other `mcpServers` entries.

Write back with 2-space indentation.

This single file is read by both the Claude Code CLI and the Claude VSCode extension.

### Step 8 — Report

```
UOFastMCP setup complete
─────────────────────────────────────────────────────────────────
Server          : <server-url>
Authenticated   : yes (Bearer token) / no (open server)
─────────────────────────────────────────────────────────────────
Claude Desktop  : ✓  <desktop config path>
Project .mcp.json : ✓  <project path>
─────────────────────────────────────────────────────────────────
```

Use `–` for any target that was skipped.

Reminders:
- **Restart Claude Desktop** to pick up the Desktop config change.
- The project `.mcp.json` is active immediately for new Claude Code / VSCode sessions.
- Re-run `/uo-setup` if the UOFastMCP server restarts and issues a new token.

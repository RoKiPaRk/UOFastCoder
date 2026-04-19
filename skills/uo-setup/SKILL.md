---
name: uo-setup
description: Verify the UOFastMCP server connection and walk through configuration if not connected. Run this first to confirm everything is working before using other UO skills.
argument-hint: []
allowed-tools: [mcp__UOFastMCP__list_files]
---

# uo-setup

Check that the UOFastMCP server is reachable and the credentials are correct. If connected, confirm and show the account. If not, give exact steps to fix it.

## Steps

### Step 1 — Test the connection

Call `mcp__UOFastMCP__list_files`.

**If the call succeeds:**

Tell the user:

```
✓ Connected to UOFastMCP
  Files visible: <count from list_files>

You're ready. Run /uo-document --all to build schema memory, then use any other UO skill.
```

Stop here.

---

**If the call fails** (tool not found, auth error, connection refused, or any error):

Tell the user exactly what went wrong, then give the full fix sequence:

---

## If not connected — fix steps

### Start the MCP server

If not already running:

```bash
pip install uofast-mcp      # first time only
uofast-mcp                  # starts on http://localhost:8000
```

### First-run setup wizard (if you haven't run it yet)

Open **http://localhost:8000/setup** in a browser.
Complete all 5 steps. **Step 5 (Client Setup) shows the exact `claude mcp add` command to copy — credentials included.**

### Register the server with Claude Code

Copy the command from the wizard's CLI tab and run it in your terminal:

```bash
claude mcp add --transport sse UOFastMCP http://localhost:8000/sse \
  --header "Authorization: Basic <your-token-from-wizard>"
```

This saves the connection to `~/.claude/mcp.json` (local, applies to all projects).

**For project-level config** (commit to source control so teammates can use it), create `.mcp.json` in your project root:

```json
{
  "mcpServers": {
    "UOFastMCP": {
      "url": "http://localhost:8000/sse",
      "headers": {
        "Authorization": "Basic <your-token-from-wizard>"
      }
    }
  }
}
```

> **Important:** The token must be a hardcoded Base64 string (`dXNlcjpwYXNz…`), not a shell variable like `${UOFASTMCP_USERNAME}`. Claude Code does not expand shell variables in `.mcp.json`.

### Restart the Claude Code session

After running `claude mcp add` or saving `.mcp.json`, restart Claude Code so it picks up the new server.
Then run `/uo-setup` again to confirm.

### Get a fresh token if yours stopped working

Your token is `base64(username:password)`. Generate one from the wizard's Client Setup page, or:

```bash
python -c "import base64; print(base64.b64encode(b'YOUR_USER:YOUR_PASSWORD').decode())"
```

Paste the output as the value after `Basic ` in the header.

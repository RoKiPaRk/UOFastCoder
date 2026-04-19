# UOFastCoder

AI-powered development tools for **U2 Unidata** and **Universe** — built into Claude Code.
Write BASIC programs, generate Python ORM code, build web forms, and create reports, all from your live DICT.

---

## Quick Start

### 1. Install and start the MCP server

```bash
pip install uofast-mcp
uofast-mcp
```

Server runs at `http://localhost:8000`.

### 2. Run the setup wizard (first run only)

Open **http://localhost:8000/setup** in a browser and complete the 5-step wizard:

| Step | What happens |
|---|---|
| Welcome | Prerequisites check |
| Security | Set admin password + JWT secret |
| Connection | Enter U2 host, user, password, account path — live test included |
| Complete | Writes `.env` and `unidata_config.ini` |
| **Client Setup** | **Shows the exact `claude mcp add` command to copy-paste** |

The final step generates your credentials and config automatically — no manual Base64 encoding needed.

### 3. Register the MCP server with Claude Code

Copy the command from the wizard's **Client Setup** page. It looks like:

```bash
claude mcp add --transport sse UOFastMCP http://localhost:8000/sse \
  --header "Authorization: Basic <your-token>"
```

This stores the connection in your **local** Claude Code config (`~/.claude/mcp.json`).

**For project-level config** (checked into source control for shared use), the wizard also shows a `.mcp.json` snippet — add it to your project root:

```json
{
  "mcpServers": {
    "UOFastMCP": {
      "url": "http://localhost:8000/sse",
      "headers": { "Authorization": "Basic <your-token>" }
    }
  }
}
```

### 4. Install the UOFastCoder plugin

Inside a Claude Code session (start with `claude` in your project):

```
/plugin marketplace add RoKiPaRk/UOFastCoder
/plugin install UOFastCoder
```

### 5. Verify the connection

```
/uo-setup
```

Confirms the MCP server is reachable and credentials are correct. If anything is wrong it gives exact steps to fix it.

### 6. Document your database (one-time setup)

```
/uo-document --all
```

This reads every file and BASIC program in your account and writes `docs/u2-schema.md` and `docs/u2-business-logic.md`. Every other skill uses these files as memory — run it once, re-run after schema changes.

That's it. You're ready.

---

## Commands

| Command | What it does |
|---|---|
| `/uo-setup` | Verify MCP server connection; show fix steps if not connected |
| `/uo-document [FILE...] [--logic [PROG...]] [--all]` | Build schema + business logic memory docs |
| `/uo-explore <FILE>` | Explain a file's structure, fields, and relationships |
| `/uo-basic <PROG> <FILE> [description]` | Write or modify a UniBasic program |
| `/uo-python <FILE>` | Generate UopyModel + service layer + Flask routes |
| `/uo-python --from-basic <PROG> <FILE>` | Translate a BASIC program to Python/uopy |
| `/uo-ui <FILE>` | Generate web form + Blueprint + HTML templates |
| `/uo-report <FILE> [description]` | Generate a UniQuery report + Python script |

---

## Typical Workflows

**New file → full Python stack:**
```
/uo-document CLIENTS          # refresh schema for this file
/uo-python CLIENTS            # model + service + routes
/uo-ui CLIENTS                # form + templates
```

**Write or modify a BASIC program:**
```
/uo-basic GET.CLIENT CLIENTS  # new program
/uo-basic GET.CLIENT CLIENTS add email validation  # modify existing
```

**Understand an existing BASIC program then port it:**
```
/uo-document --logic PROCESS.ORDERS
/uo-python --from-basic PROCESS.ORDERS ORDERS
```

**Ad-hoc report:**
```
/uo-report ORDERS orders by country with totals
```

---

## For Generated Python Code

Install the supporting packages if you plan to use the generated Flask/ORM output:

```bash
pip install uofast-orm uofastforms
```

---

## License

MIT — [RoKiPaRk](https://github.com/RoKiPaRk)

# UOFastCoder

AI-powered development tools for **U2 Unidata** and **Universe** — built into Claude Code.
Write BASIC programs, generate Python ORM code, build web forms, and create reports, all from your live DICT.

---

## Quick Start

### 1. Start the UOFastMCP server

```bash
pip install uofast-mcp
uvicorn uofast_mcp.app:app --host 0.0.0.0 --port 8000
```

### 2. Create your user account (first run only)

Open **http://localhost:8000/admin/setup** in a browser.
Log in with `admin` / `changeme123!` and create your API user.

### 3. Connect Claude Code to it

Generate the Base64 credential string:

```bash
python -c "import base64; print(base64.b64encode(b'YOUR_USER:YOUR_PASSWORD').decode())"
```

Then add to `.mcp.json` in your project root:

```json
{
  "mcpServers": {
    "UOFastMCP": {
      "url": "http://localhost:8000/sse",
      "headers": {
        "Authorization": "Basic <paste-base64-string-here>"
      }
    }
  }
}
```

Or use the CLI (which handles encoding for you):

```bash
claude mcp add --transport sse UOFastMCP http://localhost:8000/sse \
  --header "Authorization: Basic <paste-base64-string-here>"
```

### 4. Install the plugin

```bash
# Step 1 — register the GitHub repo as a plugin marketplace (one-time)
claude /plugin marketplace add RoKiPaRk/UOFastCoder

# Step 2 — install the plugin from that marketplace
claude /plugin install UOFastCoder@RoKiPaRk-UOFastCoder
```

### 5. Document your database (one-time setup)

```
/uo-document --all
```

This reads every file and BASIC program in your account and writes `docs/u2-schema.md` and `docs/u2-business-logic.md`. Every other skill uses these files as memory — run it once, re-run after schema changes.

That's it. You're ready.

---

## Commands

| Command | What it does |
|---|---|
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

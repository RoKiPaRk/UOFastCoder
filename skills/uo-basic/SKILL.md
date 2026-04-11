---
name: uo-basic
description: Generate or modify a UniBasic program for a U2 Unidata or Universe file. Reads live DICT structure, reads any existing source first to match coding style, writes the source to the BP file, and reports the result. Does NOT compile — the developer compiles manually.
argument-hint: <PROGRAM_NAME> <FILE> [description] [--bp <BP_FILE>]
allowed-tools: [mcp__UOFastMCP__get_dict_items, mcp__UOFastMCP__read_record, mcp__UOFastMCP__query_file, mcp__UOFastMCP__write_bp_program, mcp__UOFastMCP__read_bp_program, mcp__UOFastMCP__execute_command]
---

# uo-basic

Generate or modify a UniBasic program from a live U2 DICT. Matches the existing coding style. Does **not** compile.

## Arguments

The user invoked this with: $ARGUMENTS

Parse as: `<PROGRAM_NAME> <FILE> [description] [--bp <BP_FILE>]`

- `PROGRAM_NAME` — program to create or modify (e.g. `GET.CLIENT`, `PROCESS.ORDERS`)
- `FILE` — primary U2 file the program works with
- `description` — optional plain-English description of what it should do or what to change
- `--bp` — BP file to write to (default: `BP`)

## Examples

```
/uo-basic GET.CLIENT CLIENTS Subroutine to read a CLIENTS record by ID
/uo-basic PROCESS.ORDERS ORDERS Batch program to process all open orders
/uo-basic CALC.TOTALS ORDERS --bp BP.BATCH
```

## Prerequisites

UOFastMCP server must be running. The target BP file must exist in Unidata.

---

## Steps

### Step 1 — Inspect the file

Call `mcp__UOFastMCP__get_dict_items` for `<FILE>`. Identify:
- All D-type fields with exact attribute numbers
- MV fields (attr 8 = `M`)
- I-type computed fields (can use in SELECT but not WRITE)

Call `mcp__UOFastMCP__query_file` to see real records.

### Step 2 — Read existing source (CRITICAL)

Call `mcp__UOFastMCP__read_bp_program` for `<PROGRAM_NAME>` in `<BP_FILE>`.

**If the program exists:**

Analyse the existing code to identify:
- **Flow control style** — does it use `GOTO`, `GOSUB`/`RETURN`, structured `IF/THEN/ELSE/END`, or a mix? Match whatever is already there.
- **Variable naming conventions** — prefix style (`F.`, `R.`, `L.`, or something else?), case style, separator style (`.` vs `_`)
- **EQU/constant style** — how are attribute numbers declared? Inline or grouped?
- **Indentation** — spaces or tabs, how many?
- **Comment style** — `*`, `!`, or REM? What format do existing comments use?
- **Error handling pattern** — `STOP`, `ABORT`, logging pattern, etc.

Preserve the existing style exactly. Do not introduce patterns not already present.

**Change markers (required when modifying existing code):**

Wrap every block of changed or added lines with:

```
*Begin DD MMM YYYY HH:MM <DEVELOPER> <change reason>
   <new or modified lines>
*End DD MMM YYYY HH:MM <DEVELOPER> <change reason>
```

- Use today's date and current time (UTC)
- `<DEVELOPER>` — use the git user name if known, otherwise `UO-Basic`
- `<change reason>` — brief description matching the user's request
- Lines being **deleted** should be commented out (prefix with `*`) inside the Begin/End block, not removed
- Do not add Begin/End markers around unchanged code

Example:
```
*Begin 10 APR 2026 14:30 UO-Basic Add email validation
   IF EMAIL = "" THEN
      PRINT "Email is required"
      GOTO ERR.EXIT
   END
*End 10 APR 2026 14:30 UO-Basic Add email validation
```

**If the program does NOT exist:** write it fresh using the style rules below.

### Step 3 — Write the program

Use `mcp__UOFastMCP__write_bp_program` to write the complete source.

**New program requirements:**
- Comment block header (program name, purpose, author, date)
- `EQU` statements for every attribute number — get exact values from DICT, never guess
- `OPEN '' TO F.<FILE> ELSE STOP` for every file opened
- `ELSE` clause on every READ for not-found handling
- Complete code — no `* TODO` stubs

**Style defaults (for new programs only — if existing code differs, match existing):**

```
* ============================================================
* Program : <PROGRAM_NAME>
* Purpose  : <one-line description>
* Author   : UO-Basic
* Date     : DD MMM YYYY
* ============================================================

EQU CLIENT.COMPANY  TO 1
EQU CLIENT.FNAME    TO 2
...

   OPEN '' TO F.CLIENTS ELSE
      PRINT "Cannot open CLIENTS"
      STOP
   END

   READ R.CLIENT FROM F.CLIENTS, CLIENT_ID ELSE
      PRINT "Not found: ":CLIENT_ID
      STOP
   END
```

### Step 4 — Report

Tell the user:
- Whether the program was new or modified
- If modified: which sections were changed and why (referencing Begin/End markers)
- The BP file and program name where source was written
- **Remind the user to compile manually**: `BASIC <BP_FILE> <PROGRAM_NAME>` in TCL
- Any fields excluded (I-type) and why
- If it's a subroutine: how to catalog it (`CATALOG <BP_FILE> <PROGRAM_NAME> LOCAL`)

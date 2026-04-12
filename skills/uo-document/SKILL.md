---
name: uo-document
description: Introspect U2 Unidata/Universe files and BASIC programs, then write persistent reference documentation — a schema map of all file structures and relationships, and a business logic summary of selected BASIC programs. Output is saved to docs/ and referenced in CLAUDE.md so every other skill can read it as memory.
argument-hint: [FILE...] [--logic [PROG...]] [--bp <BP_FILE>] [--all]
allowed-tools: [mcp__UOFastMCP__list_files, mcp__UOFastMCP__get_dict_items, mcp__UOFastMCP__read_bp_program, mcp__UOFastMCP__execute_command, mcp__UOFastMCP__query_file, Read, Glob, Write, Edit]
---

# uo-document

Introspect live U2 files and BASIC programs, produce persistent reference documentation, and register it as project memory so every other skill can reference it automatically.

## Arguments

The user invoked this with: $ARGUMENTS

Parse as: `[FILE...] [--logic [PROG...]] [--bp <BP_FILE>] [--all]`

- `FILE...` — one or more U2 file names to document (default: all files)
- `--logic [PROG...]` — also document BASIC programs; if no names given, documents all programs in BP file
- `--bp <BP_FILE>` — BP file to read programs from (default: `BP`)
- `--all` — document every file AND every program in the BP file

## Examples

```
/uo-document
/uo-document CLIENTS ORDERS PRODUCTS
/uo-document --logic GET.CLIENT PROCESS.ORDERS
/uo-document CLIENTS ORDERS --logic GET.CLIENT
/uo-document --all
```

## Prerequisites

- UOFastMCP server running
- `docs/` directory writable in the project

---

## Steps

### Step 1 — Discover scope

**Files to document:**
- If specific `FILE` names given: use those
- If `--all` or no files given: call `mcp__UOFastMCP__list_files` and document every file returned (skip system files: `VOC`, `&SAVEDLISTS&`, `&HOLD&`, `ERRMSG`, `LANGUAGE`, `&PH&`, files starting with `&` or `.`)

**BASIC programs to document (only if `--logic` or `--all`):**
- If specific `PROG` names given after `--logic`: use those
- If `--all` or `--logic` with no names: call `mcp__UOFastMCP__execute_command` with `LIST <BP_FILE> *ID* NOPAGE` to get all program names in the BP file; document each one

### Step 2 — Build the schema map

For each file in scope:

1. Call `mcp__UOFastMCP__get_dict_items` — collect every DICT item:
   - Name, type (D/I/V), attribute number, column heading, conversion code, MV flag (attr 8)
2. Call `mcp__UOFastMCP__execute_command` with `COUNT <FILE>` — get record count
3. Call `mcp__UOFastMCP__query_file` with `limit=2` — get sample record IDs and data

**Detect relationships:**

For each D-type field, check if it is a foreign key:
- Name ends with `_NO`, `_ID`, `_IDS`, `_CODE`, `_KEY`, `_NUM`, `_REF` → FK candidate
- Strip the suffix and check if a matching file exists in the discovered file list (e.g. `CLIENT_NO` → look for `CLIENTS`, `CLIENT`)
- MV flag = `M` on a FK field → one-to-many relationship

For I-type fields, check the formula (attr 2/3) for `XLATE(` or `TRANSLATE(` — these are confirmed cross-file reads; extract the referenced file name from the function call.

Record each confirmed relationship as:
```
<SOURCE_FILE>.<FIELD> --[MV?]--> <TARGET_FILE>
```

### Step 3 — Write `docs/u2-schema.md`

Check if `docs/u2-schema.md` already exists (use `Glob`). If it does, read it — preserve any hand-written notes in sections marked `<!-- MANUAL -->` ... `<!-- /MANUAL -->` and update everything else.

Output format:

```markdown
# U2 Schema Reference
> Generated: YYYY-MM-DD HH:MM UTC
> Server: <host from connection>
> Files documented: N

---

## Files

### <FILE_NAME>
**Purpose:** <one-sentence description inferred from field names and sample data>
**Record count:** ~<N>
**Key field (record ID):** <describe ID structure from sample records, e.g. "numeric auto-increment" or "CLIENT_NO format">

| Field | Attr# | Type | Label | Conv | MV | Notes |
|-------|-------|------|-------|------|----|-------|
| COMPANY | 1 | D | Company Name | — | — | |
| ORDER_DATE | 3 | D | Order Date | D4/ | — | Date |
| TOTAL | 99 | I | Grand Total | MD2 | — | Computed — read-only |
| LINE_IDS | 10 | D | Line Items | — | ✓ | → LINES |

**Relationships:**
- `LINE_IDS` (attr 10, MV) → `LINES` — one ORDERS record links to many LINES records
- `CLIENT_NO` (attr 2) → `CLIENTS` — lookup to customer master

---

### <NEXT_FILE>
...

---

## Relationship Map

Crow's-foot notation: `>--` = many-to-one, `--<` = one-to-many

```
CLIENTS ──< ORDERS     (ORDERS.CLIENT_NO)
ORDERS  ──< LINES      (ORDERS.LINE_IDS, MV)
LINES   >── PRODUCTS   (LINES.PRODUCT_CODE)
```

## Cross-File I-Descriptors

| File | Field | References | Via |
|------|-------|------------|-----|
| ORDERS | CLIENT_NAME | CLIENTS | XLATE(CLIENTS, CLIENT_NO, COMPANY, X) |
```

### Step 4 — Document BASIC programs (only if `--logic` or `--all`)

For each program in scope:

1. Call `mcp__UOFastMCP__read_bp_program`
2. Parse the source:
   - **Header comment block** — extract stated purpose, author, date
   - **Files opened** — all `OPEN '' TO F.XXX` statements → which U2 files it touches
   - **Operations** — classify: READ, WRITE, DELETE, SELECT/loop, EXECUTE (UniQuery), CALL/GOSUB (external subroutines called)
   - **BASIC programs called** — any `CALL <name>` or `GOSUB` to labelled external routines
   - **Parameters** — if subroutine: `SUBROUTINE NAME(A, B, C)` → list and describe each arg
   - **Key business rules** — conditional logic that represents business decisions (e.g. `IF STATUS = "OPEN" AND AMOUNT > 1000 THEN ...`); describe in plain English

Do **not** reproduce the source code in the output — describe the logic in plain English.

### Step 5 — Write `docs/u2-business-logic.md`

Check if `docs/u2-business-logic.md` already exists. If it does, read it — update only the programs in scope, preserve entries for programs not in scope, and preserve `<!-- MANUAL -->` blocks.

Output format:

```markdown
# U2 Business Logic Reference
> Generated: YYYY-MM-DD HH:MM UTC

---

## <PROG_NAME> (`<BP_FILE>/<PROG_NAME>`)
**Type:** Subroutine | Main program | Batch program
**Purpose:** <from header comment, or inferred>
**Author:** <from header comment>
**Last modified:** <from Begin/End change markers if present>

**Files accessed:**
- `CLIENTS` — read (by CLIENT_ID)
- `ORDERS` — read/write
- `AUDIT.LOG` — write (append)

**Parameters:** *(subroutines only)*
| Param | In/Out | Description |
|-------|--------|-------------|
| CLIENT_ID | In | Record ID to look up |
| R.CLIENT.OUT | Out | Full client record on success |
| ERR | Out | Empty on success; error message on failure |

**Business rules:**
1. Opens CLIENTS and validates the record exists; returns error if not found
2. Applies credit limit check: if outstanding balance exceeds credit limit, sets ERR and returns
3. Writes an audit entry to AUDIT.LOG with the timestamp and operator ID

**Calls:**
- `GET.NEXT.ID` — sequence number generator
- `WRITE.AUDIT` — audit log writer

**Called by:** *(if determinable from cross-references)*
- `PROCESS.ORDERS`

---

## <NEXT_PROG>
...
```

### Step 6 — Register as project memory in CLAUDE.md

Read `CLAUDE.md` (use `Glob` first to confirm it exists). Look for a section headed `## U2 Reference Documentation`.

If the section does not exist, append it. If it exists, replace it. Do not modify any other part of CLAUDE.md.

```markdown
## U2 Reference Documentation

Auto-generated from live Unidata database by `/uo-document`. Re-run to refresh.

- [`docs/u2-schema.md`](docs/u2-schema.md) — All U2 file structures, DICT fields, and cross-file relationships. **Read this before working with any U2 file.**
- [`docs/u2-business-logic.md`](docs/u2-business-logic.md) — Business logic extracted from BASIC programs. **Read the relevant entry before modifying or translating a BASIC program.**

> Last generated: YYYY-MM-DD HH:MM UTC
```

### Step 7 — Report

Tell the user:
- Files documented (count + names)
- BASIC programs documented (count + names), or note if `--logic` was not requested
- Any files skipped and why (system files, empty DICTs)
- Any BASIC programs that could not be read (not found, empty)
- Relationships discovered (summary count)
- Reminder: re-run `/uo-document` after schema changes to keep memory fresh

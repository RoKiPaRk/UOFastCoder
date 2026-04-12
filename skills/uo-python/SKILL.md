---
name: uo-python
description: Generate a complete UopyModel, service layer, and Flask routes for a U2 Unidata file from a live DICT — OR translate an existing UniBasic program into equivalent Python code using the uopy library.
argument-hint: <FILE> [--model-only] [--service-only] | --from-basic <PROG> <FILE> [--bp <BP_FILE>]
allowed-tools: [mcp__UOFastMCP__get_dict_items, mcp__UOFastMCP__read_record, mcp__UOFastMCP__query_file, mcp__UOFastMCP__read_bp_program, Read, Glob, Write, Edit]
---

# uo-python

Two modes:
1. **DICT mode** — generate UopyModel + service layer + Flask routes from a live U2 file DICT
2. **Basic mode** — read an existing UniBasic program and translate it to equivalent Python using `uopy`

## Arguments

The user invoked this with: $ARGUMENTS

Parse as one of:
- `<FILE> [--model-only] [--service-only]` — DICT mode
- `--from-basic <PROG> <FILE> [--bp <BP_FILE>]` — Basic mode

**DICT mode flags:**
- `FILE` — U2 file name (e.g. `CLIENTS`, `ORDERS`)
- `--model-only` — generate only the UopyModel, skip service and routes
- `--service-only` — generate only the service layer

**Basic mode flags:**
- `PROG` — BASIC program name (e.g. `GET.CLIENT`, `PROCESS.ORDERS`)
- `FILE` — U2 file the program works with (used to cross-reference DICT for field names)
- `--bp` — BP file to read from (default: `BP`)

## Examples

```
/uo-python CLIENTS
/uo-python ORDERS --model-only
/uo-python --from-basic GET.CLIENT CLIENTS
/uo-python --from-basic PROCESS.ORDERS ORDERS --bp BP.BATCH
```

## Prerequisites

- UOFastMCP server running
- `uofast-orm` PyPI package installed (`pip install uofast-orm`)
- Project uses the standard UOFast layout (`models/`, `services/`, `routes/`)

---

## DICT Mode Steps

### Step 1 — Read the live DICT

Call `mcp__UOFastMCP__get_dict_items`. For every item record: type (D/I/V), attribute number, column heading, MV flag (attr 8 = `M`).

Call `mcp__UOFastMCP__read_record` on a sample record to confirm MV field structure.

### Step 2 — Check for existing model

Use `Glob` to check if `models/<file_lower>.py` exists. Read it if so.

### Step 3 — Generate `models/<file_lower>.py`

```python
from uofast_orm import UopyModel

class <Name>Model(UopyModel):
    _file_name = "<FILE>"           # exact U2 name
    _field_map = {
        "snake_prop": "DICT_NAME",  # attr N — label
        ...
    }
    _mv_fields = {"DICT_NAME", ...} # DB names (ALL_CAPS) only
    _write_field_names = [...]      # exclude ALL I-type fields
```

Rules:
- `_mv_fields` uses DB field names (ALL_CAPS), not Python property names
- `_write_field_names` must exclude every I-type (computed) field — writing them raises errors
- Add attr # and heading as inline comments

### Step 4 — Generate `services/<file_lower>_service.py`

Five functions: `list_<entities>`, `get_<entity>`, `create_<entity>`, `update_<entity>`, `delete_<entity>`.

All use `pool.acquire()` / `pool.release()` in `try/finally`.

### Step 5 — Generate `routes/<file_lower>.py` (unless --model-only or --service-only)

Flask Blueprint with GET list, GET one, POST, PUT, DELETE endpoints.
Auth guard via `session.get("authenticated")` on every route.

### Step 6 — Report

- Files written with paths
- Fields excluded and why (I-type, `@`-system)
- MV fields and what that means for API consumers
- How to register the blueprint in `app.py`
- Next: `/uo-ui <FILE>` to generate the web form

---

## Basic Mode Steps

### Step 1 — Read the BASIC source

Call `mcp__UOFastMCP__read_bp_program` for `<PROG>` in `<BP_FILE>`. If the program does not exist, stop and tell the user.

### Step 2 — Cross-reference the DICT

Call `mcp__UOFastMCP__get_dict_items` for `<FILE>`. Build a map of:
- `EQU` constant name → attribute number → DICT field name and label
- Which fields are MV (attr 8 = `M`)
- Which are I-type (read-only computed)

Use this to replace bare attribute numbers in the translated code with real field names.

### Step 3 — Translate to Python

Produce a Python file that replicates the program's logic using `uopy`. Apply the translation rules below.

**Output target:** `services/<prog_lower>.py` for subroutines; `scripts/<prog_lower>.py` for main programs.

#### Translation rules

| UniBasic | Python (`uopy`) |
|---|---|
| `OPEN '' TO F.FILE ELSE STOP` | `f_file = uopy.File('FILE', session)` — wrap in try/except UOError |
| `READ R FROM F.FILE, ID ELSE ...` | `rec = uopy.Record(f_file, id); rec.read()` — check `rec.is_new` for not-found |
| `READU R FROM F.FILE, ID LOCKED ...` | `rec = uopy.Record(f_file, id); rec.read(lock=True)` |
| `WRITE R ON F.FILE, ID` | `rec.write()` |
| `DELETE R FROM F.FILE, ID` | `rec.delete()` |
| `RELEASE R` | `rec.release_lock()` |
| `R<N>` (attr access by number) | `rec[N]` or use DICT name as comment: `rec[N]  # FIELD_NAME` |
| `R<EQU_CONST>` | Resolve EQU from Step 2; use `rec[N]  # FIELD_NAME` |
| `R<N, M>` (MV access) | `rec[N].split(chr(253))[M-1]` or `dynarray[N][M]` |
| `DCOUNT(R<N>, @VM)` | `len(rec[N].split(chr(253)))` |
| `EXECUTE stmt CAPTURING output` | `cmd = uopy.Command(stmt, session=session); cmd.run(); output = cmd.response` |
| `SELECT F.FILE TO L` / `READNEXT ID FROM L ELSE EXIT` | `ids = list(uopy.List(0, session).read_list() or [])` then `for id in ids:` |
| `GOSUB label` | Extract label block as a Python function; call it |
| `GOTO label` | Restructure as `if/else`, `while`, or `continue/break` where possible; comment `# was GOTO label` where not |
| `EQU X TO N` | Python constant: `X = N  # attr N` at module level, or just inline comment |
| `SUBROUTINE NAME(A, B, C)` | `def name(a, b, c):` — map positional args by name |
| `RETURN` (in subroutine) | `return` |
| `STOP` / `ABORT` | `raise SystemExit` or `raise RuntimeError("message")` |
| `PRINT "msg":VAR` | `print(f"msg{var}")` |
| String concat `:` | `+` or f-string |
| `IF X = "" THEN ... END` | `if x == "":` |
| `@VM`, `@SM`, `@TM` | `chr(253)`, `chr(252)`, `chr(251)` |

**Subroutine template:**

```python
"""
Translated from UniBasic: <BP_FILE>/<PROG>
Original purpose: <from program header comment if present>
"""
import uopy


def <prog_lower>(session: uopy.Session, <args>) -> <return_type>:
    """<docstring from BASIC header comment>"""
    # Attribute constants (from EQU statements)
    <FILE>_<FIELD> = <N>   # attr N — <heading>

    f_<file> = uopy.File('<FILE>', session)

    rec = uopy.Record(f_<file>, record_id)
    rec.read()
    if rec.is_new:
        raise ValueError(f"Record not found: {record_id}")

    field_value = rec[<FILE>_<FIELD>]
    ...
    return result
```

**Standalone script template:**

```python
"""
Translated from UniBasic: <BP_FILE>/<PROG>
Original purpose: <from program header comment if present>

Run: python scripts/<prog_lower>.py
"""
import os
import uopy
from dotenv import load_dotenv

load_dotenv()

# Attribute constants (from EQU statements)
<FILE>_<FIELD> = <N>   # attr N — <heading>


def main():
    session = uopy.connect(
        host=os.getenv("UNIDATA_HOST"),
        port=int(os.getenv("UNIDATA_PORT", "31438")),
        user=os.getenv("UNIDATA_USERNAME"),
        password=os.getenv("UNIDATA_PASSWORD"),
        account=os.getenv("UNIDATA_ACCOUNT"),
        service=os.getenv("UNIDATA_SERVICE", "udcs"),
    )
    try:
        <translated logic>
    finally:
        session.close()


if __name__ == "__main__":
    main()
```

### Step 4 — Flag untranslatable constructs

Some BASIC constructs have no clean Python equivalent. For each one, insert a comment:

```python
# UNTRANSLATABLE: <original BASIC line>
# Reason: <why it cannot be auto-translated>
# Suggested approach: <what the developer should do manually>
```

Common cases:
- `CHAIN` — spawns another program; suggest subprocess or direct function call
- `PERFORM` — inline subroutine execution; inline the code manually
- `LOCK` / `UNLOCK` on entire files — no uopy equivalent; use record-level locking
- `PRINT @(col, row)` — terminal positioning; replace with plain print or remove
- `INPUT var` — interactive input; replace with function parameter
- `ICONV` / `OCONV` — conversion functions; note the conversion code and implement manually

### Step 5 — Report

- Output file written
- EQU constants resolved (show the mapping table)
- GOSUB blocks extracted as functions (list them)
- GOTO statements and how they were restructured
- Any `# UNTRANSLATABLE` blocks — list them so the developer knows what to review
- Reminder: the translated code preserves logic but may not be performance-optimal; review before production use

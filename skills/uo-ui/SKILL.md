---
name: uo-ui
description: Generate a UOFastForms FormModel, Flask Blueprint, and Jinja2 HTML templates for a U2 Unidata file. Reads live DICT structure, infers field types and validators, and writes all three layers into the project.
argument-hint: <FILE> [--form-only]
allowed-tools: [mcp__UOFastMCP__get_dict_items, mcp__UOFastMCP__read_record, mcp__UOFastMCP__execute_command, Read, Glob, Write, Edit]
---

# uo-ui

Generate FormModel + Flask Blueprint + Jinja2 templates from a live U2 DICT.

## Arguments

The user invoked this with: $ARGUMENTS

Parse as: `<FILE> [--form-only]`

- `FILE` — U2 file name (e.g. `CLIENTS`, `ORDERS`)
- `--form-only` — generate only the FormModel, skip blueprint and templates

## Examples

```
/uo-ui CLIENTS
/uo-ui ORDERS
/uo-ui INVENTORY --form-only
```

## Prerequisites

- UOFastMCP server running
- `uofastforms` PyPI package installed (`pip install uofastforms`)
- UopyModel for this file already generated (run `/uo-python <FILE>` first)

---

## Steps

### Step 1 — Read live DICT

Call `mcp__UOFastMCP__get_dict_items`. Identify: type, attr #, conversion, heading, MV flag.

Skip I-type fields — computed, cannot be submitted in a form.

### Step 2 — Check for existing model

Read `models/<file_lower>.py` if it exists — use its property names as form field keys.

### Step 3 — Generate `forms/<file_lower>_form.py`

Field type inference (priority order):
| Condition | `field_type` |
|-----------|-------------|
| Conversion `D*` | `"date"` |
| Conversion `MD*`/`MR*` with decimals | `"float"` |
| Conversion `MD0`/`MR0` | `"integer"` |
| Name contains `EMAIL`/`MAIL` | `"email"` |
| Name contains `NOTES`/`DESC`/`COMMENT` | `"textarea"` |
| Name contains `QTY`/`COUNT`/`NUM` | `"integer"` |
| Name contains `AMOUNT`/`PRICE`/`TOTAL` | `"float"` |
| MV flag = `M`, free text | `mv_style="textarea"` |
| MV flag = `M`, coded values | `mv_style="fieldlist"` |
| FK field (`*_NO`, `*_ID`, `*_IDS`) | `exclude=True` |
| I-type field | `exclude=True` |

For `choices` fields: run `mcp__UOFastMCP__execute_command` to sample distinct values:
`SORT <FILE> <FIELD> BY <FIELD> COUNT.SUP NOPAGE`

### Step 4 — Generate `routes/<file_lower>.py` (unless --form-only)

Blueprint with: GET list, GET/POST edit (handles create + update), DELETE.

### Step 5 — Generate templates (unless --form-only)

- `templates/<file_lower>/list.html` — Bootstrap 5 table with search, Edit/Delete buttons
- `templates/<file_lower>/edit.html` — Bootstrap 5 form, correct input per field type

### Step 6 — Report

- Files written
- Fields excluded and why
- `choices=[]` entries needing real values
- How to register blueprint in `app.py`
- LookupModel opportunities (fields linking to other U2 files)

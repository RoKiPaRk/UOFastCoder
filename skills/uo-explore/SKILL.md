---
name: uo-explore
description: Explore a U2 Unidata or Universe file — list DICT fields, sample records, identify foreign-key relationships, and explain the data model in plain English. Use before generating any code for a file.
argument-hint: <FILE> [RECORD_ID]
allowed-tools: [mcp__UOFastMCP__list_files, mcp__UOFastMCP__get_dict_items, mcp__UOFastMCP__read_record, mcp__UOFastMCP__query_file, mcp__UOFastMCP__execute_command]
---

# uo-explore

Explore a U2 Unidata file: list its DICT fields, show a sample record, identify relationships to other files, and explain the data model.

## Arguments

The user invoked this with: $ARGUMENTS

Parse as: `<FILE> [RECORD_ID]`

- `FILE` — U2 file name (e.g. `CLIENTS`, `ORDERS`, `INVENTORY`)
- `RECORD_ID` — optional: a specific record ID to display

## Examples

```
/uo-explore CLIENTS
/uo-explore ORDERS 10001
/uo-explore INVENTORY WIDGET-001
```

## Prerequisites

This skill requires the **UOFastMCP** server to be running and configured in `.mcp.json`. See the plugin README for setup instructions.

---

## Steps

### Step 1 — Verify the file exists

Call `mcp__UOFastMCP__list_files` and confirm the file appears. If it doesn't exist, say so clearly and stop.

### Step 2 — Read DICT definitions

Call `mcp__UOFastMCP__get_dict_items` for the named file. For each item collect:
- Name, type (D = stored, I = computed, V = virtual), attribute number
- Column heading (label), conversion code (e.g. `D4/` = date, `MD2` = 2-decimal)
- MV flag (attr 8: `M` = multi-value)

### Step 3 — Sample records

If a `RECORD_ID` was given: call `mcp__UOFastMCP__read_record` for that ID.
Otherwise: call `mcp__UOFastMCP__query_file` with no criteria and `limit=3`.

### Step 4 — Identify relationships

Look for fields ending in `_NO`, `_ID`, `_IDS`, `_CODE`, `_KEY` — likely foreign keys to other files.

### Step 5 — Present the results

#### File: `<FILE>`

**Overview** — one paragraph describing the apparent business purpose.

**DICT Fields**

| Field Name | Attr # | Type | Label | Conv | MV | Notes |
|------------|--------|------|-------|------|----|-------|
| COMPANY    | 1      | D    | Company Name | — | — | |
| ORDER_DATE | 3      | D    | Order Date | D4/ | — | Date |
| TOTAL      | 99     | I    | Grand Total | MD2 | — | Computed — read-only |
| LINE_IDS   | 10     | D    | Line Items | — | ✓ | MV |

**Sample Record** — actual data formatted with field names, not raw attribute numbers.

**Relationships** — foreign key fields and the files they reference.

**Next Steps**
- `/uo-basic <FILE>` — write a UniBasic program
- `/uo-python <FILE>` — generate UopyModel + service layer
- `/uo-ui <FILE>` — generate a web form
- `/uo-report <FILE>` — build a report

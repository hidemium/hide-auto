---
name: "hideauto-shared-data-files"
description: "Mandatory rules when building a HideAuto workflow that reads/writes files or spreadsheets (readFile, writeFile, readSpreadsheet, writeSpreadsheet): keep paths in variables, and mark data rows (claim → verify → used) so parallel runs never reuse the same data. Use whenever a workflow takes items from a list (emails, accounts, addresses…) or writes results to a file — even if the user never mentions parallel runs."
---

# hideauto-shared-data-files

Read it whenever a workflow contains **any** of: `readFile`, `writeFile`, `readSpreadsheet`, `writeSpreadsheet`.

## 0. Baseline assumption: the workflow ALWAYS runs in parallel

A HideAuto workflow is not run once — the same workflow will run as **many threads at the same time** (one profile per thread, each thread its own process). Threads **do not share variables**, but they **do share files/sheets**. Consequences:

- Two threads reading the same list will grab **the same row** unless rows are marked.
- Two threads writing the same local file at the same time will **overwrite each other's data**.

→ **Always design as if running in parallel, even if the user did not ask for it.** `run_test` uses a single thread, so race bugs do NOT show up — "the test passed" is not proof of safety.

When done, briefly tell the user which marking columns/mechanism you added and why (so they prepare their data file correctly).

---

## 1. File paths / sheet URLs must live in variables

- Declare file paths, output folders, Google Sheet URL/ID and sheet names as workflow variables (`variable_manage`; or use an existing global variable).
- Nodes only reference `{{key}}` — **never hard-code** a path inside a node.

```json
"variables": [
  { "id": "v-data-sheet", "key": "data_sheet", "value": "https://docs.google.com/spreadsheets/d/..." },
  { "id": "v-data-tab",   "key": "data_tab",   "value": "Accounts" },
  { "id": "v-result-dir", "key": "result_dir", "value": "D:/hideauto-output" },
  { "id": "v-gsheet-sa",  "key": "gsheet_service_account", "value": "<Service Account JSON>" }
]
```

```json
{ "code": "readSpreadsheet", "readSpreadsheetSource": "googleSheet",
  "readSpreadsheetGoogleIdOrUrl": "{{data_sheet}}", "readSpreadsheetSheet": "{{data_tab}}" }
```

- Data read from files must also go into **variables** (`readFileSaveTo`, `readSpreadsheetColumns[].variableKey`); later nodes use `{{var}}` — do not re-read the file to get the same value again.

---

## 2. Shared data lists (emails, accounts, addresses…) → marking is MANDATORY

### 2a. Required sheet columns

| Column | Purpose | Values |
|---|---|---|
| `status` | Marker column | `new` = unused · `claimed:<token>` = held by a thread · `used:<token>` = done · `failed:<token>` = used but failed |
| `row` | The row's own sheet row number | Google Sheet: formula `=ROW()`; xlsx: fill 2, 3, 4… |
| (data) | `email`, `password`, `address`… | |

Why these two columns are needed (current node limits):
- `readSpreadsheet` in `findRow` mode only matches **exact equality** and **rejects an empty compare value** → it cannot find "empty cell". Unused rows must have `status` = `new` (pre-filled by the user).
- `readSpreadsheet` **does not return the matched row number** → read the `row` column into a variable so `writeSpreadsheet` can target the right cell (`status!{{row_no}}`).

Check the sheet's real columns with `spreadsheet_inspect` before configuring nodes. If the user's sheet lacks these columns: **tell the user to add them** (or add them yourself if it is a local file you are allowed to edit). Never skip marking.

### 2b. Claim → verify → use → mark flow (exact order; no other node between steps 2–3)

```
1. random        → claim_token (characters, length 12)        ← this thread's own token
2. readSpreadsheet findRow: status == "new"
                  columns: row → row_no, email → email, ...
                  error (no "new" rows left) → addLog "Out of data" → stop
3. writeSpreadsheet (cells): "status!{{row_no}}" = "claimed:{{claim_token}}"   ← IMMEDIATELY after step 2
4. pause random 1–3 s                                           ← let other threads' writes land
5. readSpreadsheet findRow: status == "claimed:{{claim_token}}"
                  columns: row → row_no, email → email, ...    ← re-read; use values from this read
   success → the row is ours → step 6
   error   → another thread took it → back to step 2 (bounded retries, see 2d)
6. ... work with {{email}}, {{password}} ...
7a. success → writeSpreadsheet "status!{{row_no}}" = "used:{{claim_token}}"
7b. failure → writeSpreadsheet "status!{{row_no}}" = "failed:{{claim_token}}"
```

Rules:
- **The search condition is always `status == "new"`.** Never use `firstRow` for a shared list — it always returns the first row, so every thread gets the same one.
- **Step 3 must directly follow step 2** — every node in between widens the window for duplicates.
- **Step 5 (verify) is mandatory.** Two threads can both find the same `new` row before either writes; the last writer wins, and verify lets the loser notice and pick another row.
- **Every `error` branch after step 5 must lead to step 7b** (profile open errors, click errors…). A row stuck at `claimed:` can never be used again.
- On failure write `failed:` rather than resetting to `new`, so a broken row is not retried forever. Reset to `new` only if the user explicitly wants automatic retries.
- To record extra info (timestamp, profile used), write extra cells in the **same** step-7 node, e.g. `used_at!{{row_no}}`, `profile!{{row_no}}`.

### 2c. Node examples (steps 2, 3, 7a)

```json
{ "id": "n-find", "type": "basic", "data": {
  "code": "readSpreadsheet", "label": "Find unused row",
  "readSpreadsheetSource": "googleSheet",
  "readSpreadsheetGoogleIdOrUrl": "{{data_sheet}}", "readSpreadsheetSheet": "{{data_tab}}",
  "readSpreadsheetHasHeader": true,
  "readSpreadsheetRowMode": "findRow",
  "readSpreadsheetCondition": { "columnKey": "status", "compareValue": "new" },
  "readSpreadsheetColumns": [
    { "columnKey": "row",   "variableKey": "row_no" },
    { "columnKey": "email", "variableKey": "email" }
  ] } }
```

```json
{ "id": "n-claim", "type": "basic", "data": {
  "code": "writeSpreadsheet", "label": "Claim row",
  "writeSpreadsheetSource": "googleSheet",
  "writeSpreadsheetGoogleUrl": "{{data_sheet}}", "writeSpreadsheetSheet": "{{data_tab}}",
  "writeSpreadsheetServiceAccountJson": "{{gsheet_service_account}}",
  "writeSpreadsheetAppendLastRow": false,
  "writeSpreadsheetColumns": [
    { "columnKey": "status!{{row_no}}", "value": "claimed:{{claim_token}}" }
  ] } }
```

Step 7a is the same as `n-claim` with `value` = `"used:{{claim_token}}"`. For the full field set always use the node template (`graph_get_templates`) — do not guess.

### 2d. Bound the retries

The "verify failed → back to step 2" loop must be bounded: `setVariables` increments a counter `claim_try`, an `if` node checks `claim_try` > 5 → `addLog` + stop. Never create an infinite loop.

---

## 3. Local files (.xlsx / .csv / .txt / .json) — careful with parallel threads

**Every node that writes a local file reads the whole file and overwrites the whole file** (including `writeFile` in `append` mode and `writeSpreadsheet` cell/append writes on local files). Two threads writing at once → the later write erases the earlier one's changes (lost result lines, lost `claimed`/`used` marks on other rows).

Therefore:

1. **Shared data lists used by many threads → prefer Google Sheet.** Google writes per cell / server-side append and does not overwrite other rows. If the user provides a local file and the workflow will run in parallel, **suggest moving to Google Sheet** and explain why.
2. If a local file must be used for a shared list: still apply §2 in full (claim → verify → mark) to reduce duplicates, and **tell the user clearly** that risk remains when several threads write at the same time.
3. **Never let multiple threads write results into the same local file.** Pick one:
   - One file per thread: name by token/profile, e.g. `writeFileOutputPath` = `{{result_dir}}/result-{{claim_token}}.csv` (or `{{profile_uuid}}` if an `openConnect` node ran earlier).
   - Or append to a Google Sheet (`writeSpreadsheet`, `writeSpreadsheetAppendLastRow: true`, source `googleSheet`).
4. **`readFile` (.txt list, one item per line):**
   - Do not use `readFileMode: "at"` with a fixed line number for a shared list — every thread gets the same line.
   - If a .txt file must be used: `readFileMode: "random"` + `readFileDeleteLineAfterRead: true` (take and remove) to reduce duplicates; tell the user this only reduces, not eliminates, duplicates/lost lines. There is no `used` marker, so to keep history write consumed items to a per-thread file (item 3).
   - **Never** `readFileMode: "all"` + `readFileDeleteLineAfterRead: true` — the first thread wipes the file.

---

## 4. Checklist before reporting done

- [ ] All file paths / sheet URLs / sheet names are variables; nodes only use `{{key}}`.
- [ ] Shared lists: search by `status == "new"`, claim immediately, verify by token, mark `used`/`failed` on the success branch and on every error branch.
- [ ] The claim retry loop is bounded.
- [ ] No two threads write the same local file (per-thread output files or Google Sheet).
- [ ] `run_test` passed (remember: a single-thread test does not prove parallel safety — check by re-reading the sheet that `status` moved `claimed:` → `used:`).
- [ ] Told the user: required `status`/`row` columns, pre-filled `new` values, and (if local files) the remaining risk.

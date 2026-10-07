---
name: "hideauto-data-resources"
description: "Input/output data for HideAuto workflows. Use when a workflow has nodes that need REAL DATA (reading mail, logging in, signing up, 2FA, filling forms, using proxies, posting prepared content…) or when each run produces new information / has real side effects (creating accounts, fetching codes/tokens, ordering, scraping…): BEFORE building, ask whether the user already has resource lists and where they are stored → take the file path/sheet link, analyze its columns and build suitable read-file/spreadsheet nodes; also suggest logging each run's results to a spreadsheet for tracking."
---

# hideauto-data-resources

Once files/sheets are agreed on, also apply `hideauto-shared-data-files` (paths in variables, row marking, parallel-safe writes).

## 0. When to apply

**A. The workflow needs real data to run** — nodes/steps use information the agent cannot invent:

- Logging in (email/username + password), reading mail (`readHotmail…`, needs email + password/refresh token/client id), 2FA codes (secret).
- Sign-ups needing prepared info (email, phone, address, name, photo).
- Filling forms, posting/commenting prepared content, processing a list of links, using proxies from a list.

**B. Each run produces new information or has real side effects** the user will need to look up: creating accounts, changing passwords, enabling 2FA, posting, ordering, fetching codes/tokens/cookies/links, scraping, bulk runs across many profiles.

A → do §1–§3. B → also do §4. Usually both.

**Never** hard-code real data (emails, passwords…) into nodes, even if the user pastes it into chat — put it into the list/variables.

## 1. Ask before building (one compact round, not piecemeal)

Before editing the graph, ask the user via `ask_user` (with ready-made options). Combine into **one round**:

1. **Do you already have the resources, and where are they stored?** Name exactly what the workflow needs, e.g. "a Hotmail list with email | password | refresh_token | client_id". Suggested options:
   - "Yes, Google Sheet" → ask for the link + tab name.
   - "Yes, a local file (.xlsx/.csv/.txt)" → ask for the full path.
   - "Not yet" → should the workflow generate them (`random` node; only fits tasks like sign-ups), or will the user prepare a file from a **column template** (§2d)?
2. (If part B applies) **Log each run's results to a spreadsheet?** State the benefit (review what was done, success/failure, avoid duplicates, easy filter/export) and propose the columns (§4).
3. **Google Sheet or local file?** Recommend Google Sheet for multiple threads (parallel-safe writes — `hideauto-shared-data-files` §3). Writing to a Google Sheet needs a Service Account JSON and the sheet shared with the Service Account's email; reading needs the sheet set to "anyone with the link can view".

If the request already answers these, **do not ask again**. If the user declines result logging, respect it and mention once that it can be added later.

## 2. Path in hand → analyze the file, then build the read nodes

### 2a. Inspect the real structure (don't make the user retype columns)

- **Local .xlsx / .xls / .csv:** `spreadsheet_inspect` `mode: "overview"` with `source.path` = the path → sheets, row counts, headers. Then `mode: "rows"` (small `limit`) for a few sample rows.
- **Google Sheet:** `spreadsheet_inspect` reads Google Sheets only through an existing node → first create a `readSpreadsheet` node (source `googleSheet`, URL/tab via variables), then inspect with `source: { workflowId, nodeId }`. If it cannot be read, the sheet is usually not shared as "anyone with the link can view": tell the user.
- **.txt file:** read the first lines with `file_read` (outside the workspace may need user approval). If it cannot be read, ask the user to paste 2–3 sample lines (passwords masked). Determine the line format, e.g. `email|password|refresh_token|client_id`.
- **Do not echo sensitive data** (passwords, tokens) in replies — only mention column names and formats.

### 2b. Map columns → data the nodes need

- Build a mapping: data the node needs ↔ file column, e.g. `password` ↔ column `pass`. Map obvious columns yourself; ask via `ask_user` for ambiguous ones (`col3`, no header, two similar columns).
- A required column is missing (e.g. no `refresh_token` for a mail node) → tell the user; never invent values.
- No header (`hasHeader: false`) → use column letters `A`, `B`, `C`… as `columnKey`.

### 2c. Build the nodes

- File path / sheet link / tab name → workflow **variables** (`variable_manage`); nodes use `{{key}}`.
- **Spreadsheet (.xlsx/.csv/Google):** a `readSpreadsheet` node placed **before** the nodes that use the data; `readSpreadsheetColumns` maps each needed column → a clearly named variable (`email`, `password`…); later nodes use `{{email}}`, `{{password}}`.
- **Lists consumed across runs** (almost always) → apply claim → verify → mark from `hideauto-shared-data-files` §2: needs `status` (`new`) and `row` columns. If missing → tell the user to add them (or add them yourself for a local file you may edit). Use `spreadsheet_inspect` `mode: "match"` with condition `status = new` to check how many usable rows remain.
- **.txt file, one item per line:** `readFile` (`readFileMode: "random"` + `readFileDeleteLineAfterRead: true` for a shared list — see limits in `hideauto-shared-data-files` §3), then split the line parts (`email|password|…`) into variables with a suitable node (`extractionInText` / `runJs`). If the workflow runs multiple threads, suggest moving the list to a Google Sheet so rows can be marked.
- The read node's `error` branch (out of data / unreadable file) → `addLog` with a clear reason → stop. Never continue with empty variables.
- Take the full field set from the node template (`graph_get_templates`) — do not guess. Run `run_test` to confirm the data is read correctly (`addLog` a non-sensitive column, e.g. `{{email}}`).

### 2d. No resources yet → column template

Give the user a template for the task, always with `row` + `status` (adapt columns):

| row | status | email | password | recovery_email | proxy |
|---|---|---|---|---|---|
| 2 | new | a@mail.com | ... | ... | ... |

Google Sheet: `row` column = `=ROW()`. Build the workflow against the template (paths in variables) so the user only fills in data and sets the variable values.

## 3. Check before reporting done (input side)

- [ ] No real data hard-coded in nodes.
- [ ] Paths/links are variables; read nodes come before the nodes using the data; every needed column is mapped to a variable.
- [ ] Shared lists are marked (`hideauto-shared-data-files` §2).
- [ ] The read node's error branch leads to `addLog` + stop.

## 4. Result-tracking sheet (output)

**Append one row** per run (or per processed item). Suggested columns — pick what fits, no extras:

| Column | Content |
|---|---|
| `time` | Run time |
| `profile` | `{{profile_uuid}}` (after `openConnect`) |
| `input` | Key of the resource used, e.g. `{{email}}` — to cross-reference the input list |
| (newly produced info) | e.g. `username`, `password`, `post_url`, `order_id`, `token`, `code` |
| `status` | `success` / `failed` |
| `note` | Error reason / notes |

How:
- `writeSpreadsheet` node with `writeSpreadsheetAppendLastRow: true`, `columnKey` = header name, `value` = `{{var}}`. The sheet should have its header row prepared.
- Timestamp: there is no built-in time variable → use `runJs` `return new Date().toLocaleString('sv-SE')` saved to `run_time` (`runJsSaveResultTo`).
- Log **failures too**: the `error` branches of the main steps lead to an append node with `status = failed` (and mark `failed:` in the input list). Failed rows matter as much as successful ones.
- Log results **as soon as they exist** (e.g. right after an account is created), not only at the end — if a later step fails, the created info is not lost.
- Multiple threads: use Google Sheet append, or one file per thread (`hideauto-shared-data-files` §3). Never let multiple threads write the same local file.
- Keep the result sheet path/URL in variables (e.g. `result_sheet`, `result_tab`).

## 5. Notes for the user

- Input and result sheets may contain **passwords, tokens, cookies** — remind the user to keep sharing private (only the Service Account + people who need it; if the input sheet must be "anyone with the link can view", don't spread the link).
- When reporting done: state which sheet/file is input and which is results, which columns map to which variables, and what the user must prepare (headers, `status`/`row` columns, `new` values, sharing with the Service Account).

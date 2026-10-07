---
name: "hideauto-result-tracking"
description: "When the user asks for a workflow/scenario/script where each run produces new information or has real side effects (sign-ups, creating emails, posting, commenting, ordering, fetching codes/tokens, scraping…): BEFORE building, ask whether they already have resource lists (emails, accounts, proxies, addresses, content…), and proactively suggest logging each run's results to a spreadsheet for tracking and review."
---

# hideauto-result-tracking

Once files/sheets are agreed on, also apply `hideauto-shared-data-files` (paths in variables, row marking, parallel-safe writes).

## 0. When to apply

Apply when each run of the workflow **produces new information** or **has real side effects** the user will later need to look up, e.g.:

- Registering / creating accounts, creating emails, changing passwords, enabling 2FA.
- Posting, commenting, messaging, following, ordering, deposits/withdrawals, submitting forms.
- Fetching codes, tokens, cookies, links, order IDs; scraping/collecting data.
- The workflow will run in bulk across many profiles.

Not needed for one-off read/view workflows that produce nothing worth keeping.

## 1. Ask before building (one compact question round, not piecemeal)

Before editing the graph, ask the user via `ask_user` (with ready-made options). Combine these into **one round**:

1. **Do you already have resource lists?** Name them concretely for the task, e.g. emails (+ passwords), accounts, phone numbers, proxies, addresses, names/photos, post content, links to process…
   - Yes → where (Google Sheet / .xlsx / .csv / .txt)? Read the real columns with `spreadsheet_inspect` instead of making the user type them.
   - No → should the workflow generate them (`random` node: email, name, password…), or will the user prepare a file? If the user prepares it, give them a **column template** (section 2).
2. **Log each run's results to a spreadsheet?** State the benefit: review created accounts, see which succeeded/failed, avoid duplicates, easy filtering and export. Propose the column list (section 3) so the user only has to accept or tweak.
3. **Google Sheet or local file?** Recommend Google Sheet if running multiple threads (parallel-safe writes — see `hideauto-shared-data-files` §3). Writing to Google Sheet needs a Service Account JSON, and the sheet must be shared with the Service Account's email.

If the request already answers these points, **do not ask again** — just use them. If the user declines result logging, respect it and mention once that it can be added later.

## 2. Input list (resources)

- One resource per row, header in the first row. Besides data columns, **always** include `status` (initial value `new`) and `row` (row number; Google Sheet `=ROW()`) for marking used rows — details in `hideauto-shared-data-files` §2.
- Suggested template (adapt columns to the task):

| row | status | email | password | recovery_email | proxy |
|---|---|---|---|---|---|
| 2 | new | a@mail.com | ... | ... | ... |

## 3. Result-tracking sheet (output)

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

## 4. Notes for the user

- The result sheet may contain **passwords, tokens, cookies** — remind the user to keep sharing private (only the Service Account + people who need it).
- When reporting done: state which sheet is input and which is results, the columns used, and what the user must prepare (headers, `new` values, sharing with the Service Account).

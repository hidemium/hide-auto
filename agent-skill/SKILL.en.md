---
name: "hideauto-build-workflow"
description: "Build and edit browser-automation workflows in the HideAuto app: edit .hideauto files directly and use the hideauto CLI to validate / test-run / read logs. Use when the user asks to create, edit, or debug a HideAuto workflow."
---

# hideauto-build-workflow

Guide for an AI agent to operate the **HideAuto** app to build and debug browser-automation workflows.

The **`hideauto` command** is already on the machine (installed app: on PATH automatically; dev: after `npm run cli:link`). If `hideauto` isn't found, try the full path or tell the user. **Standalone** commands (validate / list-node-types / get-node-template / workspace) run without the app; **connected** commands (run-test / run-stop / reload-session / list-profiles / ping) need the HideAuto app running — if it isn't, they return `app-not-running` (exit 4). Check with `hideauto ping` first.

---

## 1. Mental model

- **A workflow is a graph** of `nodes` (steps) + `edges` (flow), stored as **one plain-JSON `.hideauto` file** on disk (the `workflows/` folder). The file is an envelope — **the graph lives under `.data`** (see §2).
- **The file is the source of truth.** You **edit the file directly** with normal file read/write tools — there is no "node-editing API". The CLI only covers what you *can't* do by editing the file: validation, test runs, reading logs, listing the node catalog / profiles.
- **Find the working directory first:** run `hideauto workspace` → it returns `workflowsDir` (where all `.hideauto` files live; it's also a git repo). Operate on files with **absolute paths** under this dir. `workflowsDir` depends on the signed-in user, so don't guess or hard-code it.
- **A workflow's id = its relative path** under `workflowsDir` (POSIX, no extension). E.g. `<workflowsDir>/Marketing/Post Reels.hideauto` → id `Marketing/Post Reels`.
- Workflows drive a real browser (open a profile, visit URLs, click, type, read data…). Running one has real side effects → **always test in a controlled way**.

### Golden rules

1. **Read before you edit.** Open the current file, understand the graph, then change it. Prefer editing/cloning an existing node over inventing one from scratch.
2. **Never invent a node `code`.** Run `hideauto list-node-types` to see which nodes exist. A wrong `code` → validate reports `node-unknown-code`.
3. **Validate after every edit** (`hideauto validate`). Only move on when `ok: true`.
4. **The app auto-detects your file edits.** If the workflow is open in the app, its file-watcher reloads the canvas automatically (when the user has no unsaved UI edits) — usually you need to do nothing. Use `hideauto reload-session` only to **force** the app to match disk now (discarding that workflow's unsaved in-memory draft).
5. **Test before declaring done** (`hideauto run-test`). Read the log, fix until it passes. (run-test locks the canvas + opens the Logs tab in the app, exactly like the user clicking Test — the user can watch.)
6. **The profile is workflow content** — decide it with the user and write it into the `openConnect` node (see §5). Don't treat the profile as a run-time parameter.
7. **Never trust a bare `outcome:"success"`.** It only means the node ran without throwing — NOT that the intended effect happened. After every meaningful state-changing action, insert a node that reads the real state (`getUrl` / `getText` / `runJs` for `document.title`…) then an `addLog` to confirm (see §6c).
8. **Stay inside `workflows/`.** Don't touch files outside this dir. Don't run real campaigns/schedules.

---

## 2. `.hideauto` file structure

The file is an **ENVELOPE** — the graph lives under **`.data`**, not at the top level. Editing nodes/edges means editing `.data.nodes` / `.data.edges`:

```json
{
  "id": "<keep as-is>",
  "filePath": "<keep as-is>",
  "updatedAt": "<keep as-is; the app resets it on save>",
  "data": {
    "nodes": [
      { "id": "n-start", "type": "start", "position": { "x": 40, "y": 160 },
        "data": { "code": "start", "label": "Start" } },

      { "id": "n-open", "type": "basic", "position": { "x": 380, "y": 150 },
        "data": { "code": "openConnect", "label": "Open & Connect",
                  "openConnectMode": "defaultProfile" } },

      { "id": "n-goto", "type": "basic", "position": { "x": 380, "y": 280 },
        "data": { "code": "gotoUrl", "label": "Go to url", "gotoUrl": "{{url}}" } }
    ],
    "edges": [
      { "id": "e1", "source": "n-start", "sourceHandle": "out",
        "target": "n-open", "targetHandle": "in" },
      { "id": "e2", "source": "n-open", "sourceHandle": "success",
        "target": "n-goto", "targetHandle": "in" }
    ],
    "variables": [
      { "id": "v-url", "key": "url", "value": "https://example.com" }
    ],
    "viewport": { "x": 0, "y": 0, "zoom": 0.85 }
  }
}
```

- **Graph = `.data`.** When editing an existing file: keep `id`/`filePath`/`updatedAt` as-is, only touch inside `.data`. The real workflow identity is its relative path, not the `id` field.
- **node.id** is unique within the file. **node.type** is the canvas type (`start`, `stop`, `basic`, `if`, `ifElse`, `loop`, `elementExists`, `comment`). **node.data.code** is the real action (`openConnect`, `gotoUrl`, …). **node.data.label** shows on the canvas.
- Nodes may carry ephemeral fields (`measured`, `selected`…) — the app strips them; you **don't** need to add any.
- **Other fields inside `data`** depend on `code` (e.g. `gotoUrl` has a `gotoUrl` field). Get the correct fields by running **`hideauto get-node-template <code>`** — it returns `dataDefault` (the full default, including `code`+`label`) and `aiKeys` (editable fields). This is the source of truth — **don't guess fields**. (Or clone a node with the same `code` already in the file.) Edge handle names aren't in the template — just write them and let `validate` catch mistakes.
- **edges** connect the source node's `sourceHandle` to the target node's `targetHandle`. A `basic` node usually emits from `success` / `error`; the `start` node emits from `out`; targets are usually `in`. If unsure about a handle name, just write it — **`validate` reports `edge-invalid-handle`** so you can fix it; or copy an edge from a node of the same kind.
- **variables**: `key` is the variable name, referenced inside a node as `{{key}}`. There are special variables for the profile (§5).
- **viewport**: just the canvas view position; doesn't affect running. Keep as-is or ignore.
- There must be **exactly one `start` node**.

---

## 3. CLI commands (quick reference)

Call `hideauto <command> <path> [flags]`, read **stdout** (JSON; `run-test` is JSONL), check the **exit code**.

| Command | App running? | Purpose |
|---------|--------------|---------|
| `hideauto workspace` | No | **Run FIRST** — prints `workflowsDir` (where the `.hideauto` files are) + appDataPath |
| `hideauto open [--timeout n]` | (launches) | Launch the app if not running, wait until ready (use when `ping` returns `app-not-running`) |
| `hideauto validate <path>` | No | Check the file is valid (graph structure) |
| `hideauto list-node-types` | No | List usable nodes (catalog only) |
| `hideauto get-node-template <code>` | No | Get a node's `data` template (fields + defaults) |
| `hideauto ping` | Yes | Check the app is running (call before "Yes" commands) |
| `hideauto list-profiles [--search q] [--limit n]` | Yes | List the app's profiles to reference in openConnect (existedProfile) |
| `hideauto run-test <path> [--scope ...] [--start-node <id>] [--verbose\|--quiet] [--max-lines n]` | Yes | Test-run, stream compact logs (§6) |
| `hideauto run-stop <path>` | Yes | Stop a running test run for the workflow |
| `hideauto reload-session <path>` | Yes | Force the app to reload the file from disk (usually unneeded — see rule #4) |
| `hideauto critic-review <path>` | Yes | Review workflow quality → findings + verdict (exit 1 if blocked) |
| `hideauto list-variables` | Yes | List global variables (workflows reference `{{key}}`; secret values masked) |
| `hideauto list-workflows` | Yes | List existing workflows in the app (id/name/folder) |
| `hideauto browser-state <path>` | Yes | Show the browser connection state for a workflow |
| `hideauto browser <sub> <path> [arg]` | Yes | **Inspect the LIVE run-test browser** (query/eval/screenshot/url/goto) — find/verify selectors (§6b) |

Errors go to **stderr** as `{ "error": { "code", "message" } }` with a non-zero exit code. `app-not-running` (exit 4) = the GUI app isn't open (the "Yes" commands need it).

---

## 4. Standard loop

```
0. Locate      → hideauto workspace   (get workflowsDir). For connected commands: hideauto ping → if app-not-running, hideauto open (wait for the app to start)
1. Understand  → clarify anything vague with the user (goal, URLs, data, PROFILE §5)
2. Context     → read the current .hideauto file in workflowsDir; hideauto list-node-types (once per session)
3. Edit file   → add/edit inside .data.nodes / .data.edges / .data.variables. New node: get-node-template <code> first
4. Validate    → hideauto validate <path>   → on error, go back to (3)
5. (Optional)  → the app auto-reloads if open; force with hideauto reload-session <path> if you want certainty
6. Test-run    → hideauto run-test <path>   → read the JSONL log (§6)
7. Diagnose    → which node errored? fix its data/edge → go back to (3)
8. (Advised)   → hideauto critic-review <path> → fix high-severity findings / a "blocked" verdict before handing off
9. Done        → report the result to the user + summarize the workflow you built
```

Never jump ahead while the previous step isn't clean (validate still has errors, or the run still `failed`).

---

## 5. Profile & the `openConnect` node (IMPORTANT)

**Which profile to use is file content, NOT a `run-test` parameter.** The `openConnect` node decides it. `run-test` has no profile flag. To change the profile → edit `openConnect` → run again.

**Always confirm with the user before writing `openConnect`:** run with a new profile, an existing profile, or the default?

5 modes (the `openConnectMode` field inside the openConnect node's `data`):

| Mode | When to use | Fields to fill |
|------|-------------|----------------|
| `defaultProfile` | Quick run, the bundled core's default profile | — |
| `newProfile` | **Create a fresh profile each run** (new fingerprint). Delete after if needed | `openConnectFingerprint*` (optional), `openConnectDeleteProfileDirWhenDone: true` to clean up |
| `existedProfile` | **Use an existing app profile** (by path from `list-profiles`) | `openConnectProfilePath` **or** the `{{profile_path}}` variable |
| `otherApp` | Profile in a **separate external app** (Hidemium/AdsPower/GPM) — rarely used here | `openConnectPlatform` + `openConnectBrowserUuid` |
| `remoteCdp` | Connect to an already-open browser via CDP — edge case | `openConnectRemoteCdp` **or** the `{{remote_ws}}` variable |

- **Only this app's own profiles exist.** `hideauto list-profiles` lists them (each has a `path`). The main modes are app-internal: `defaultProfile` / `newProfile` / `existedProfile`. (`otherApp`/`remoteCdp` target a separate external app — rarely used.)
- "Run with a **new profile**" = `openConnectMode: "newProfile"` — pure file edit, no command needed.
- Use an existing profile: `existedProfile` + `openConnectProfilePath` = the `path` from `list-profiles` (or the `{{profile_path}}` variable).
- `openConnectCloseWhenDone` (close the browser when the run finishes) depends on your test needs.

Example openConnect using an existing app profile:

```json
{ "id": "n-open", "type": "basic", "position": { "x": 380, "y": 150 },
  "data": { "code": "openConnect", "label": "Open & Connect",
            "openConnectMode": "existedProfile",
            "openConnectProfilePath": "{{profile_path}}" } }
```
```json
// in "variables":
{ "id": "v-pf", "key": "profile_path", "value": "<path from list-profiles>" }
```

---

## 6. Reading `run-test` logs

`run-test` prints **compact JSONL (the default)** to stdout to **save tokens**, and blocks until done:

```jsonc
{ "type": "started", "runId": "...", "logFilePath": "...", "mode": "compact" }
{ "type": "step", "nodeId": "n-open", "nodeCode": "openConnect", "outcome": "success", "ms": 812 }
{ "type": "step", "nodeId": "n-goto", "nodeCode": "gotoUrl", "outcome": "error", "ms": 5, "message": "...", "variables": { } }
{ "type": "summary", "status": "failed", "steps": 2, "errors": 1,
  "elapsedMs": 1200, "firstError": { "nodeId": "n-goto", "message": "..." } }
```

- **Diagnose:** check `summary.firstError` (or the `step` line with `outcome:"error"`) → `nodeId` + `message` tell you which step broke and why → fix that node's `.data` (selector/URL/variable…) or the edge leading into it. Only error steps carry `variables` (state at failure).
- **Compact drops** `variables` on successful steps, and drops `step-started`/debug → lighter on tokens. To see everything, read the file at `logFilePath` (JSONL on disk) with a file-reading tool — you **don't** need `--verbose`.
- **Token flags:** `--quiet` (errors + summary only — cheapest) · `--verbose` (full `{type:"log",line:RunLogLine}` per step) · `--max-lines <n>` (cap step lines, default 1000; long loops → `summary.stepsOmitted`).
- **Exit code:** `0` completed · `1` failed · `130` stopped.
- **Stop:** kill the CLI process (Ctrl-C) — the app stops the run automatically.

`--scope`:
- `full` (default) — run from Start.
- `from-node` + `--start-node <id>` — run from a node.
- `single-node` + `--start-node <id>` — run exactly one node (quick single-step test).

---

## 6b. Inspect the LIVE browser to find selectors

When you're unsure of a selector for a node: **use the very browser that run-test just opened** (no separate browser). After `run-test`, the browser stays on the last page (or the failing node's page) → inspect it directly:

- `hideauto browser url <path>` → current URL + title (confirm you're on the right page).
- `hideauto browser query <path> "<css>"` → `{ count, first:{tag,id,cls,text,visible,html} }`. **count=1 + the right element** = a good selector. count=0 → wrong; count>1 → not specific enough.
- `hideauto browser eval <path> "<js expression>"` → run **READ-ONLY** JS in the page to probe (e.g. `document.querySelectorAll('.item').length`, `[...document.querySelectorAll('a')].map(a=>a.getAttribute('href'))`). **Do NOT** click/submit/mutate state via eval — that's the workflow node's job.
- `hideauto browser screenshot <path> [--full]` → saves a PNG to the OS temp dir, returns `{path}`. `Read` the image to see the page, **then DELETE the file** (don't litter the user's machine).
- `hideauto browser goto <path> <url>` → navigate (if you need another page to inspect).

Loop: `run-test` (up to the point you need) → `browser query/eval` to find the selector → put it in the node → `run-test` again to verify. Without a prior `run-test` (no live browser) → `browser-unavailable` error.

---

## 6c. Confirm state after an action (verify + addLog)

A node's `outcome:"success"` only means it **ran without throwing** — NOT that the intended effect happened (landed on the right page, logged in, the form actually submitted). So **after every meaningful state-changing node** (`gotoUrl`, a `clickElement` that navigates, `typeText`+submit, login…), insert a step that **reads the real state, then logs it**:

1. **Read the real state** with an appropriate "get" node and **save it to a variable** (`{{key}}`):
   - `getUrl` → save via `getUrlSaveTo` — confirm you reached the right URL.
   - `getText` (`getTextSelector` + `getTextSaveTo`) — read an anchor element's text (e.g. the username after login, a post title).
   - `getHtml` (`getHtmlSelector` + `getHtmlSaveTo`) / `getAttribute` — read html/attributes when needed.
   - **Page title:** there is NO dedicated node → use `runJs` with `runJsScript: "return document.title"` and save via `runJsSaveResultTo`.
   - To **assert** an element exists/is visible: `waitForSelector`, `checkElementVisible`, or `elementExists` (branching).
2. **Log the confirmation** with an `addLog` node referencing the saved variable (`addLogMessage` supports `{{var}}`):
   ```json
   { "id": "n-log-after-login", "type": "basic", "position": { "x": 700, "y": 150 },
     "data": { "code": "addLog", "label": "Verify login",
               "addLogMessage": "After login → url={{cur_url}} user={{login_name}}",
               "addLogIncludeTimestamp": true } }
   ```
   This line shows up in the Execution Log (visible to the user during Test) and in the JSONL run-log you read in §6 → use it to confirm the prior action really took effect, instead of trusting `outcome:success`.
3. **To halt on failure:** use `elementExists` / `if` to branch to an error path instead of blindly continuing.

Typical chains:
- `gotoUrl` → `getUrl` (save `{{cur_url}}`) → `addLog "url={{cur_url}}"`.
- `clickElement` (Login) → `waitForSelector ".avatar"` → `getText` (`.username` → `{{login_name}}`) → `addLog "logged in as {{login_name}}"`.

> **Why use `addLog` to print variables:** in compact logs (§6) a `success` step does NOT include `variables` (only failing nodes do). `addLog` is the cheap way to "print" a variable you read into the run-log without `--verbose`.

---

## 7. Reading `validate` errors

Output: `{ ok, errors[], warnings[] }`. Each issue: `{ severity, rule, message, nodeId?, edgeId?, index? }`. `rule` is a stable slug:

| rule (some) | Meaning & fix |
|-------------|---------------|
| `graph-no-start` | Missing a `start` node → add one |
| `node-unknown-code` | `data.code` doesn't exist → check `list-node-types` |
| `node-missing-code` / `node-missing-label` | node is missing `data.code` / `data.label` |
| `node-duplicate-id` / `edge-duplicate-id` | duplicate `id` → change it |
| `edge-dangling-source` / `edge-dangling-target` | edge points to a nonexistent node → fix `source`/`target` |
| `edge-invalid-handle` | wrong handle name → copy the handle from a node of the same kind |
| `node-invalid-position` | missing numeric `position: {x, y}` |

`warnings` don't block (e.g. `graph-multiple-start`, `variable-invalid`) but should be addressed.

> `validate` only checks **structure**, **not** configuration content (whether a selector exists, whether a URL is right). Those only surface at `run-test`.

---

## 8. Safety & boundaries

- **Only operate inside `workflows/`.** Don't touch the app's other files.
- **Don't run campaigns/schedules** (bulk runs over real profiles). Only `run-test`.
- Before a workflow does real side effects beyond the test browser (sending mail, posting, purchasing…): call it out and let the user decide.
- Don't create/delete many profiles. For `newProfile` during tests, consider `openConnectDeleteProfileDirWhenDone` to avoid disk clutter.
- Editing files (most work) → do it directly. Going through the CLI → only the commands in §3, don't invent others.

---

## 9. Notes

- **Per-node `data` templates** come from `get-node-template <code>` (list-node-types only returns the light catalog: code/title/canvasType). Backup: clone a node with the same `code`.
- **`{{key}}`** inside a node can reference a **global variable** (run `hideauto list-variables` to see them) or a variable in the workflow's own `.data.variables`.
- **`critic-review`** returns pre-built English `message`+`fixHint`; `verdict:"blocked"` means fix before handing off.
- The CLI is self-describing: `hideauto --help`, `hideauto list-node-types`, and `hideauto get-node-template <code>` are the live sources of truth.

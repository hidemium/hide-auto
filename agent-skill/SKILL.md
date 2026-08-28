---
name: "hideauto-build-workflow"
description: "Xây và sửa workflow tự động hoá trong app HideAuto: sửa file .hideauto trực tiếp, dùng CLI hideauto để validate/chạy test/đọc log. Dùng khi user nhờ tạo, sửa, hoặc debug một workflow HideAuto."
---

# hideauto-build-workflow

Hướng dẫn AI agent vận hành app **HideAuto** để xây và debug workflow tự động hoá trình duyệt.

**Lệnh `hideauto`** đã có sẵn trên máy (bản cài: tự trên PATH; bản dev: sau `npm run cli:link`). Nếu gõ `hideauto` không thấy, thử đường dẫn đầy đủ hoặc báo user. Lệnh **standalone** (validate/list-node-types/get-node-template) chạy không cần app; lệnh **connected** (run-test/run-stop/reload-session/list-profiles/ping) cần app HideAuto đang mở — không mở → lỗi `app-not-running` (exit 4). Kiểm tra bằng `hideauto ping` trước.

---

## 1. Mô hình cần nắm

- **Workflow = một graph** gồm `nodes` (bước) + `edges` (luồng), lưu trong **một file `.hideauto` là plain JSON** trên đĩa (thư mục `workflows/`). File là envelope — **graph nằm ở `.data`** (xem §2).
- **File là nguồn sự thật.** Bạn **sửa file trực tiếp** bằng công cụ đọc/ghi file — không có "API sửa node". CLI chỉ phục vụ những việc **không** làm được bằng sửa file: kiểm tra hợp lệ, chạy thử, đọc log, tra catalog/profile.
- **Tìm thư mục làm việc trước tiên:** chạy `hideauto workspace` → trả `workflowsDir` (nơi chứa mọi file `.hideauto`, cũng là git repo). Thao tác trên file **bằng đường dẫn tuyệt đối** dưới thư mục này. `workflowsDir` phụ thuộc user đang đăng nhập nên đừng đoán/hard-code.
- **Định danh workflow = đường dẫn tương đối** dưới `workflowsDir` (POSIX, bỏ đuôi). Ví dụ `<workflowsDir>/Marketing/Post Reels.hideauto` → id `Marketing/Post Reels`.
- Workflow điều khiển trình duyệt thật (mở profile, vào URL, click, gõ, đọc dữ liệu…). Chạy nó = có side effect thật → **luôn test có kiểm soát**.

### Nguyên tắc vàng

1. **Đọc trước khi sửa.** Mở file hiện tại, hiểu graph, rồi mới đổi. Ưu tiên sửa/nhân bản node đã có hơn là bịa node mới từ đầu.
2. **Không bịa `code` node.** Chạy `hideauto list-node-types` để biết node nào tồn tại. `code` sai → validate báo `node-unknown-code`.
3. **Validate sau mỗi lần sửa** (`hideauto validate`). Chỉ đi tiếp khi `ok: true`.
4. **App tự bắt file bạn sửa.** Nếu workflow đang mở trong app, file-watcher tự reload canvas (nếu user chưa sửa dở trên UI) — thường KHÔNG cần làm gì. Dùng `hideauto reload-session` khi muốn **ép** app khớp đĩa ngay (bỏ bản RAM chưa lưu của workflow đó).
5. **Test trước khi tuyên bố xong** (`hideauto run-test`). Đọc log, sửa tới khi đạt. (run-test khoá canvas + bật Logs tab trong app giống user bấm Test — user xem được.)
6. **Profile là nội dung workflow** — chốt với user, ghi vào node `openConnect` (xem §5). Đừng coi profile là tham số lúc chạy.
7. **Ở trong `workflows/`.** Không sửa file ngoài thư mục này. Không tự chạy campaign/schedule thật.

---

## 2. Cấu trúc file `.hideauto`

File là **ENVELOPE** — graph nằm ở **`.data`**, KHÔNG phải top-level. Sửa node/edge là sửa `.data.nodes` / `.data.edges`:

```json
{
  "id": "<giữ nguyên>",
  "filePath": "<giữ nguyên>",
  "updatedAt": "<giữ nguyên; app tự set lại khi save>",
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

- **Graph = `.data`.** Sửa file hiện có: giữ nguyên `id`/`filePath`/`updatedAt`, chỉ đụng bên trong `.data`. Định danh workflow thật = đường dẫn tương đối, không phải field `id`.
- **node.id** duy nhất trong file. **node.type** là kiểu canvas (`start`, `stop`, `basic`, `if`, `ifElse`, `loop`, `elementExists`, `comment`). **node.data.code** là hành động thật (`openConnect`, `gotoUrl`, …). **node.data.label** hiển thị trên canvas.
- Node có thể có field ephemeral (`measured`, `selected`…) — app tự strip; **không cần** thêm.
- **Các field khác trong `data`** tuỳ theo `code` (vd `gotoUrl` có field `gotoUrl`). Lấy đúng field bằng cách chạy **`hideauto get-node-template <code>`** — trả `dataDefault` (default đầy đủ, gồm cả `code`+`label`) và `aiKeys` (field được phép sửa). Đây là nguồn chuẩn, **đừng đoán field**. (Hoặc nhân bản node cùng `code` đã có trong file.) Tên handle của edge không nằm trong template — cứ viết rồi để `validate` bắt nếu sai.
- **edges** nối `sourceHandle` của node nguồn tới `targetHandle` của node đích. Node `basic` thường phát từ handle `success` / `error`; node `start` phát từ `out`; đích thường là `in`. Nếu không chắc tên handle → cứ viết, **`validate` sẽ báo `edge-invalid-handle`** để bạn sửa; hoặc soi edge của node cùng loại đã có.
- **variables**: `key` là tên biến, tham chiếu trong node bằng `{{key}}`. Có biến đặc biệt cho profile (§5).
- **viewport**: chỉ là vị trí nhìn canvas, không ảnh hưởng chạy. Giữ nguyên hoặc bỏ qua.
- Phải có **đúng một node `start`**.

---

## 3. Lệnh CLI (tham chiếu nhanh)

Gọi `hideauto <command> <path> [flags]`, đọc **stdout** (JSON; `run-test` là JSONL), kiểm tra **exit code**.

| Lệnh | Cần app đang mở? | Dùng để |
|------|------------------|---------|
| `hideauto workspace` | Không | **Chạy ĐẦU TIÊN** — in `workflowsDir` (thư mục chứa `.hideauto` để sửa) + appDataPath |
| `hideauto open [--timeout n]` | (tự bật) | Bật app nếu chưa chạy, chờ tới khi sẵn sàng (dùng khi `ping` báo `app-not-running`) |
| `hideauto validate <path>` | Không | Kiểm tra file hợp lệ (cấu trúc graph) |
| `hideauto list-node-types` | Không | Liệt kê node dùng được (chỉ catalog) |
| `hideauto get-node-template <code>` | Không | Lấy template `data` (field + default) của 1 node |
| `hideauto ping` | Có | Kiểm tra app đang chạy (gọi trước các lệnh "Có") |
| `hideauto list-profiles [--search q] [--limit n]` | Có | Liệt kê profile của app để điền vào openConnect (existedProfile) |
| `hideauto run-test <path> [--scope ...] [--start-node <id>] [--verbose\|--quiet] [--max-lines n]` | Có | Chạy thử, stream log gọn (§6) |
| `hideauto run-stop <path>` | Có | Dừng test run đang chạy của workflow |
| `hideauto reload-session <path>` | Có | Ép app đọc lại file từ đĩa (thường không cần — xem nguyên tắc #4) |
| `hideauto critic-review <path>` | Có | Chấm chất lượng workflow → findings + verdict (exit 1 nếu blocked) |
| `hideauto list-variables` | Có | Liệt kê global variables (workflow tham chiếu `{{key}}`; value secret bị mask) |
| `hideauto list-workflows` | Có | Liệt kê workflow đã có trong app (id/name/folder) |
| `hideauto browser-state <path>` | Có | Trạng thái browser đang kết nối cho workflow |
| `hideauto browser <sub> <path> [arg]` | Có | **Soi browser LIVE** của run-test (query/eval/screenshot/url/goto) — tìm/verify selector (§6b) |

Lỗi in ra **stderr** dạng `{ "error": { "code", "message" } }`, exit code ≠ 0. `app-not-running` (exit 4) = app GUI chưa mở (các lệnh "Có" cần app).

---

## 4. Vòng lặp chuẩn

```
0. Định vị         → hideauto workspace   (lấy workflowsDir). Cần lệnh connected: hideauto ping → nếu app-not-running thì hideauto open (chờ app bật)
1. Hiểu yêu cầu    → hỏi/chốt với user điều còn mơ hồ (mục tiêu, URL, dữ liệu, PROFILE §5)
2. Nắm bối cảnh    → đọc file .hideauto trong workflowsDir; hideauto list-node-types (đầu phiên)
3. Sửa file        → thêm/sửa trong .data.nodes / .data.edges / .data.variables. Node mới: get-node-template <code> lấy field trước
4. Validate        → hideauto validate <path>   → có error thì quay lại (3)
5. (Tuỳ chọn)      → app tự reload nếu đang mở; ép bằng hideauto reload-session <path> nếu cần chắc
6. Chạy thử        → hideauto run-test <path>   → đọc JSONL log (§6)
7. Chẩn đoán       → node nào error? sửa data/edge tương ứng → quay lại (3)
8. (Nên) Chấm      → hideauto critic-review <path> → sửa finding severity cao / verdict "blocked" trước khi giao
9. Xong            → báo user kết quả + tóm tắt workflow đã làm
```

Không tự nhảy sang bước sau khi bước trước chưa sạch (validate còn error, hay run còn `failed`).

---

## 5. Profile & node `openConnect` (QUAN TRỌNG)

**Dùng profile nào là nội dung file, KHÔNG phải tham số `run-test`.** Node `openConnect` quyết định. `run-test` không có cờ chọn profile. Muốn đổi profile → sửa `openConnect` → chạy lại.

**Luôn chốt với user trước khi ghi `openConnect`:** chạy bằng profile mới, profile có sẵn, hay mặc định?

5 mode (field `openConnectMode` trong `data` của node openConnect):

| Mode | Khi nào dùng | Field cần điền |
|------|--------------|----------------|
| `defaultProfile` | Chạy nhanh, profile mặc định của core bundled | — |
| `newProfile` | **Tạo profile mới mỗi lần chạy** (fingerprint mới). Xoá sau nếu cần | `openConnectFingerprint*` (tuỳ chọn), `openConnectDeleteProfileDirWhenDone: true` để dọn |
| `existedProfile` | **Dùng profile CÓ SẴN của app** (theo path từ `list-profiles`) | `openConnectProfilePath` **hoặc** biến `{{profile_path}}` |
| `otherApp` | Profile ở **app NGOÀI riêng biệt** (Hidemium/AdsPower/GPM) — hầu như KHÔNG dùng ở đây | `openConnectPlatform` + `openConnectBrowserUuid` |
| `remoteCdp` | Kết nối trình duyệt đã mở qua CDP — edge case | `openConnectRemoteCdp` **hoặc** biến `{{remote_ws}}` |

- **Chỉ có profile của chính app này.** `hideauto list-profiles` liệt kê chúng (có `path`). Các mode chính = app-internal: `defaultProfile` / `newProfile` / `existedProfile`. (`otherApp`/`remoteCdp` là app NGOÀI riêng biệt — hầu như không dùng.)
- "Chạy với **profile mới**" = `openConnectMode: "newProfile"` — thuần sửa file, không cần lệnh nào.
- Dùng profile có sẵn: `existedProfile` + `openConnectProfilePath` = `path` từ `list-profiles` (hoặc biến `{{profile_path}}`).
- `openConnectCloseWhenDone` (đóng trình duyệt khi chạy xong) tuỳ nhu cầu test.

Ví dụ openConnect dùng profile có sẵn của app:

```json
{ "id": "n-open", "type": "basic", "position": { "x": 380, "y": 150 },
  "data": { "code": "openConnect", "label": "Open & Connect",
            "openConnectMode": "existedProfile",
            "openConnectProfilePath": "{{profile_path}}" } }
```
```json
// trong "variables":
{ "id": "v-pf", "key": "profile_path", "value": "<path từ list-profiles>" }
```

---

## 6. Đọc log `run-test`

`run-test` in JSONL **gọn (compact, mặc định)** ra stdout để **tiết kiệm token**, block tới khi xong:

```jsonc
{ "type": "started", "runId": "...", "logFilePath": "...", "mode": "compact" }
{ "type": "step", "nodeId": "n-open", "nodeCode": "openConnect", "outcome": "success", "ms": 812 }
{ "type": "step", "nodeId": "n-goto", "nodeCode": "gotoUrl", "outcome": "error", "ms": 5, "message": "...", "variables": { } }
{ "type": "summary", "status": "failed", "steps": 2, "errors": 1,
  "elapsedMs": 1200, "firstError": { "nodeId": "n-goto", "message": "..." } }
```

- **Chẩn đoán:** xem `summary.firstError` (hoặc dòng `step` `outcome:"error"`) → `nodeId` + `message` cho biết bước nào hỏng, vì sao → sửa `.data` của node đó (selector/URL/biến…) hoặc edge dẫn tới nó. Node lỗi mới kèm `variables` (state lúc lỗi).
- **Compact bỏ** `variables` ở step success, bỏ `step-started`/debug → nhẹ token. Muốn xem đầy đủ: đọc file ở `logFilePath` (JSONL trên đĩa) bằng công cụ đọc file, **không** cần `--verbose`.
- **Flags token:** `--quiet` (chỉ lỗi + summary — rẻ nhất) · `--verbose` (full `{type:"log",line:RunLogLine}` mỗi step) · `--max-lines <n>` (cap dòng step, mặc định 1000; loop dài → `summary.stepsOmitted`).
- **Exit code:** `0` completed · `1` failed · `130` stopped.
- **Dừng:** kill process CLI (Ctrl-C) — app tự stop run.

`--scope`:
- `full` (mặc định) — chạy từ Start.
- `from-node` + `--start-node <id>` — chạy từ một node.
- `single-node` + `--start-node <id>` — chạy đúng một node (test nhanh một bước).

---

## 6b. Soi browser LIVE để tìm selector

Khi không chắc selector cho một node: **dùng chính browser mà run-test vừa mở** (không mở browser riêng). Sau `run-test`, browser **ở nguyên trang cuối** (hoặc trang node lỗi) → soi trực tiếp:

- `hideauto browser url <path>` → URL + title hiện tại (xác nhận đang ở đúng trang).
- `hideauto browser query <path> "<css>"` → `{ count, first:{tag,id,cls,text,visible,html} }`. **count=1 + đúng element** = selector tốt. count=0 → sai; count>1 → chưa đủ cụ thể.
- `hideauto browser eval <path> "<js expression>"` → chạy JS **CHỈ ĐỌC** trong page để dò (vd `document.querySelectorAll('.item').length`, `[...document.querySelectorAll('a')].map(a=>a.getAttribute('href'))`). **KHÔNG** click/submit/đổi state bằng eval — đó là việc của node trong workflow.
- `hideauto browser screenshot <path> [--full]` → lưu PNG vào OS temp, trả `{path}`. `Read` ảnh để xem trang, **rồi XOÁ file** (đừng để rác trên máy user).
- `hideauto browser goto <path> <url>` → điều hướng (nếu cần tới trang khác để soi).

Vòng làm: `run-test` (tới chỗ cần) → `browser query/eval` tìm selector → điền vào node → `run-test` lại verify. Chưa `run-test` (không có browser live) → lỗi `browser-unavailable`.

---

## 7. Đọc lỗi `validate`

Output: `{ ok, errors[], warnings[] }`. Mỗi issue: `{ severity, rule, message, nodeId?, edgeId?, index? }`. `rule` là slug ổn định:

| rule (một số) | Nghĩa & cách sửa |
|---------------|------------------|
| `graph-no-start` | Thiếu node `start` → thêm một node start |
| `node-unknown-code` | `data.code` không tồn tại → đối chiếu `list-node-types` |
| `node-missing-code` / `node-missing-label` | node thiếu `data.code` / `data.label` |
| `node-duplicate-id` / `edge-duplicate-id` | trùng `id` → đổi id |
| `edge-dangling-source` / `edge-dangling-target` | edge trỏ tới node không tồn tại → sửa `source`/`target` |
| `edge-invalid-handle` | tên handle sai → soi handle của node cùng loại |
| `node-invalid-position` | thiếu `position: {x, y}` số |

`warnings` không chặn (vd `graph-multiple-start`, `variable-invalid`) nhưng nên xử lý.

> `validate` chỉ kiểm **cấu trúc**, **không** kiểm nội dung cấu hình (selector có thật không, URL đúng chưa). Cái đó chỉ lộ ra khi `run-test`.

---

## 8. An toàn & ranh giới

- **Chỉ thao tác trong `workflows/`.** Không đụng file khác của app.
- **Không tự chạy campaign/schedule** (chạy hàng loạt profile thật). Chỉ `run-test`.
- Trước thao tác có side effect thật ngoài trình duyệt test (gửi mail, đăng bài, mua…) trong workflow: nêu rõ cho user, để user quyết định.
- Không tự tạo/xoá nhiều profile. `newProfile` khi test thì cân nhắc `openConnectDeleteProfileDirWhenDone` để không rác đĩa.
- Việc sửa file (đa số thao tác) → làm trực tiếp. Việc qua CLI → đúng các lệnh ở §3, không bịa lệnh khác.

---

## 9. Ghi chú

- **Template `data` cho từng node** lấy qua `get-node-template <code>` (list-node-types chỉ trả catalog nhẹ code/title/canvasType). Backup: nhân bản node cùng `code` đã có.
- **`{{key}}`** trong node có thể tham chiếu **global variable** (chạy `hideauto list-variables` để biết có gì) hoặc biến trong `.data.variables` của chính workflow.
- **`critic-review`** dùng câu tiếng Anh dựng sẵn (`message`+`fixHint`); `verdict:"blocked"` = nên sửa trước khi giao.

## 10. Liên quan

- Spec CLI: `docs/agent-command-spec.md`
- Spec từng node: `docs/*-node-spec.md`
- openConnect: `src/shared/open-connect.ts`
- AI agent panel nội bộ (khác skill này): `docs/ide-shell-plan.md`

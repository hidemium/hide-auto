---
name: "hideauto-shared-data-files"
description: "Quy tắc bắt buộc khi xây workflow HideAuto có node đọc/ghi file hoặc spreadsheet (readFile, writeFile, readSpreadsheet, writeSpreadsheet): đường dẫn để trong biến, và đánh dấu dòng dữ liệu (claim → verify → used) để nhiều luồng chạy song song không dùng trùng dữ liệu. Dùng mỗi khi workflow lấy dữ liệu từ danh sách (email, tài khoản, địa chỉ…) hoặc ghi kết quả ra file — kể cả khi user không nhắc tới chạy song song."
---

# hideauto-shared-data-files

Đọc khi workflow có **bất kỳ** node nào trong số: `readFile`, `writeFile`, `readSpreadsheet`, `writeSpreadsheet`.

## 0. Giả định nền: workflow LUÔN chạy nhiều luồng song song

Workflow HideAuto không chạy một lần rồi thôi — cùng một workflow sẽ được chạy **nhiều luồng cùng lúc** (mỗi luồng một profile, mỗi luồng là một process riêng). Các luồng **không chia sẻ biến**, nhưng **dùng chung file/sheet**. Hệ quả:

- Hai luồng đọc cùng một danh sách sẽ lấy **cùng một dòng** nếu không đánh dấu.
- Hai luồng ghi cùng một file local cùng lúc sẽ **ghi đè mất** dữ liệu của nhau.

→ **Luôn thiết kế như thể đang chạy song song, kể cả khi user không yêu cầu.** Lúc chạy thử (`run_test`) chỉ có 1 luồng nên lỗi race KHÔNG lộ ra — đừng lấy "test chạy ổn" làm bằng chứng là an toàn.

Khi xong, nói ngắn gọn với user: đã thêm cột/cơ chế đánh dấu gì và vì sao (để họ chuẩn bị file dữ liệu cho đúng).

---

## 1. Đường dẫn file / URL sheet phải nằm trong biến

- Khai đường dẫn file, thư mục output, URL/ID Google Sheet, tên sheet thành biến của workflow (`variable_manage`; hoặc dùng global variable nếu user đã có).
- Node chỉ tham chiếu `{{key}}`, **không hard-code** đường dẫn vào node.

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

- Dữ liệu đọc ra cũng phải vào **biến** (`readFileSaveTo`, `readSpreadsheetColumns[].variableKey`), các node sau dùng `{{biến}}` — không đọc lại file nhiều lần để lấy cùng một giá trị.

---

## 2. Danh sách dữ liệu dùng chung (email, tài khoản, địa chỉ…) → BẮT BUỘC đánh dấu

### 2a. Cấu trúc sheet cần có

| Cột | Mục đích | Giá trị |
|---|---|---|
| `status` | Cột đánh dấu | `new` = chưa ai dùng · `claimed:<token>` = luồng đang giữ · `used:<token>` = đã dùng xong · `failed:<token>` = dùng nhưng lỗi |
| `row` | Số dòng của chính nó trên sheet | Google Sheet: công thức `=ROW()`; xlsx: điền số 2, 3, 4… |
| (dữ liệu) | `email`, `password`, `address`… | |

Vì sao cần 2 cột này (giới hạn của node hiện tại):
- `readSpreadsheet` chế độ `findRow` chỉ so khớp **bằng chính xác** và **không cho giá trị so sánh rỗng** → không tìm được "ô trống". Dòng chưa dùng phải có `status` = `new` (user điền sẵn).
- `readSpreadsheet` **không trả về số dòng** tìm được → phải đọc cột `row` vào biến để `writeSpreadsheet` ghi đúng ô (`status!{{row_no}}`).

Xem cột thật của sheet bằng `spreadsheet_inspect` trước khi cấu hình node. Nếu sheet của user chưa có 2 cột này: **báo user thêm** (hoặc tự thêm nếu là file local bạn được phép sửa). Không bỏ qua bước đánh dấu.

### 2b. Luồng claim → verify → use → mark (đúng thứ tự, không chèn node khác vào giữa bước 2–3)

```
1. random        → claim_token (characters, dài 12)          ← token riêng của luồng này
2. readSpreadsheet findRow: status == "new"
                  cột: row → row_no, email → email, ...
                  error (hết dòng "new") → addLog "Hết dữ liệu" → stop
3. writeSpreadsheet (ghi ô): "status!{{row_no}}" = "claimed:{{claim_token}}"   ← ghi NGAY sau bước 2
4. pause random 1–3 giây                                       ← chờ các luồng khác ghi xong
5. readSpreadsheet findRow: status == "claimed:{{claim_token}}"
                  cột: row → row_no, email → email, ...       ← đọc lại, lấy giá trị từ lần đọc này
   success → dòng này là của mình → sang bước 6
   error   → luồng khác đã giành dòng này → quay lại bước 2 (giới hạn số lần thử, xem 2d)
6. ... làm việc với {{email}}, {{password}} ...
7a. thành công → writeSpreadsheet "status!{{row_no}}" = "used:{{claim_token}}"
7b. lỗi        → writeSpreadsheet "status!{{row_no}}" = "failed:{{claim_token}}"
```

Quy tắc:
- **Điều kiện tìm luôn là `status == "new"`.** Không bao giờ dùng `firstRow` cho danh sách dùng chung — nó luôn trả dòng đầu, mọi luồng lấy trùng nhau.
- **Bước 3 phải nằm ngay sau bước 2** — mỗi node chen giữa là thêm cửa sổ để luồng khác lấy trùng.
- **Bước 5 (verify) là bắt buộc.** Hai luồng có thể cùng tìm thấy một dòng `new` trước khi ai kịp ghi; người ghi sau thắng, bước verify giúp người thua phát hiện và đi tìm dòng khác.
- **Mọi nhánh `error` sau bước 5 đều phải dẫn tới bước 7b** (kể cả lỗi mở profile, lỗi click…). Dòng bị kẹt ở `claimed:` sẽ không luồng nào dùng lại được.
- Lỗi: ghi `failed:` thay vì trả về `new`, tránh một dòng hỏng bị thử lại vô hạn. Chỉ trả về `new` nếu user nói rõ muốn tự động thử lại.
- Muốn lưu thêm thông tin (thời điểm, profile đã dùng), ghi thêm ô ở **cùng node** bước 7, vd `used_at!{{row_no}}`, `profile!{{row_no}}`.

### 2c. Ví dụ node (bước 2, 3, 7a)

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

Bước 7a giống `n-claim`, đổi `value` thành `"used:{{claim_token}}"`. Field đầy đủ: luôn lấy từ template của node (`graph_get_templates`), đừng đoán.

### 2d. Giới hạn số lần thử lại

Vòng "verify thất bại → quay lại bước 2" phải có giới hạn: dùng `setVariables` tăng biến đếm `claim_try`, node `if` kiểm `claim_try` > 5 → `addLog` + stop. Không tạo vòng lặp vô hạn.

---

## 3. File local (.xlsx / .csv / .txt / .json) — cẩn thận khi nhiều luồng

**Mọi node ghi file local đều đọc cả file rồi ghi đè cả file** (kể cả `writeFile` chế độ `append` và `writeSpreadsheet` ghi ô / append trên file local). Hai luồng ghi cùng lúc → bản ghi sau xoá mất thay đổi của bản ghi trước (mất dòng kết quả, mất dấu `claimed`/`used` của dòng khác).

Vì vậy:

1. **Danh sách dữ liệu dùng chung nhiều luồng → ưu tiên Google Sheet.** Google ghi theo từng ô/append phía server, không ghi đè các dòng khác. Nếu user đưa file local và workflow sẽ chạy song song, **đề xuất chuyển sang Google Sheet** và giải thích lý do.
2. Nếu buộc phải dùng file local cho danh sách dùng chung: vẫn áp dụng đầy đủ §2 (claim → verify → mark) để giảm trùng, và **báo rõ cho user** là vẫn còn rủi ro khi nhiều luồng ghi cùng lúc.
3. **Ghi kết quả (output) không bao giờ cho nhiều luồng ghi chung một file local.** Chọn một trong hai:
   - Mỗi luồng một file: đặt tên theo token/profile, vd `writeFileOutputPath` = `{{result_dir}}/result-{{claim_token}}.csv` (hoặc `{{profile_uuid}}` nếu node `openConnect` đã chạy trước).
   - Hoặc append vào Google Sheet (`writeSpreadsheet`, `writeSpreadsheetAppendLastRow: true`, source `googleSheet`).
4. **`readFile` (danh sách .txt mỗi dòng một mục):**
   - Không dùng `readFileMode: "at"` với số dòng cố định cho danh sách dùng chung — mọi luồng lấy cùng dòng.
   - Nếu buộc dùng file .txt: `readFileMode: "random"` + `readFileDeleteLineAfterRead: true` (lấy rồi xoá khỏi file) để giảm trùng; nói với user đây chỉ là giảm, không loại bỏ hoàn toàn trùng/mất dòng. Không có dấu `used` nên muốn giữ lịch sử thì ghi mục đã dùng ra file riêng theo luồng (mục 3).
   - **Không bao giờ** `readFileMode: "all"` + `readFileDeleteLineAfterRead: true` — luồng đầu tiên xoá sạch file.

---

## 4. Checklist trước khi báo xong

- [ ] Mọi đường dẫn file / URL sheet / tên sheet nằm trong biến, node chỉ dùng `{{key}}`.
- [ ] Danh sách dùng chung: tìm theo `status == "new"`, claim ngay, verify bằng token, mark `used`/`failed` ở cả nhánh thành công lẫn mọi nhánh lỗi.
- [ ] Vòng thử lại claim có giới hạn.
- [ ] Không có hai luồng nào ghi chung một file local (output tách file theo luồng hoặc dùng Google Sheet).
- [ ] Đã chạy thử `run_test` (nhớ: test 1 luồng không chứng minh an toàn song song — kiểm bằng cách đọc lại sheet thấy `status` đổi đúng `claimed:` → `used:`).
- [ ] Đã nói với user: cột `status`/`row` cần có, giá trị `new` cần điền sẵn, và (nếu dùng file local) rủi ro còn lại.

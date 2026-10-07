---
name: "hideauto-data-resources"
description: "Dữ liệu vào/ra của workflow HideAuto. Dùng khi workflow có node cần DỮ LIỆU THẬT (đọc mail, đăng nhập, đăng ký, 2FA, điền form, dùng proxy, đăng bài theo nội dung có sẵn…) hoặc mỗi lần chạy tạo ra thông tin mới/có tác động thật (tạo tài khoản, lấy mã/token, đặt hàng, cào dữ liệu…): TRƯỚC khi xây, hỏi user đã có sẵn danh sách tài nguyên chưa và lưu ở đâu → lấy đường dẫn file/link sheet, phân tích cột rồi tự dựng node đọc file/spreadsheet hợp lý; đồng thời gợi ý ghi kết quả mỗi lượt chạy vào trang tính để theo dõi."
---

# hideauto-data-resources

Khi đã chốt dùng file/sheet thì áp dụng tiếp `hideauto-shared-data-files` (biến cho đường dẫn, đánh dấu dòng, ghi song song an toàn).

## 0. Khi nào áp dụng

**A. Workflow cần dữ liệu thật để chạy** — có node/bước dùng thông tin mà agent không thể tự bịa:

- Đăng nhập (email/username + password), đọc mail (`readHotmail…`, cần email + password/refresh token/client id), lấy mã 2FA (secret).
- Đăng ký cần thông tin có sẵn (email, số điện thoại, địa chỉ, tên, ảnh).
- Điền form, đăng bài/comment theo nội dung có sẵn, xử lý danh sách link, dùng proxy theo danh sách.

**B. Workflow mỗi lần chạy sinh ra thông tin mới hoặc có tác động thật** mà user cần tra lại: tạo tài khoản, đổi mật khẩu, bật 2FA, đăng bài, đặt hàng, lấy mã/token/cookie/link, cào dữ liệu, chạy hàng loạt trên nhiều profile.

Có A → làm §1–§3. Có B → làm thêm §4. Thường có cả hai.

**Không bao giờ** hard-code dữ liệu thật (email, mật khẩu…) vào node, kể cả khi user dán thẳng vào chat — đưa vào danh sách/biến.

## 1. Hỏi trước khi xây (một lượt hỏi gọn, không hỏi lắt nhắt)

Trước khi sửa graph, hỏi user bằng `ask_user` (có sẵn lựa chọn). Gộp vào **một lần hỏi**:

1. **Đã có sẵn tài nguyên chưa, lưu ở đâu?** Nêu cụ thể đúng thứ workflow cần, vd "danh sách Hotmail gồm email | password | refresh_token | client_id". Lựa chọn gợi ý:
   - "Có, Google Sheet" → xin link + tên tab.
   - "Có, file trên máy (.xlsx/.csv/.txt)" → xin đường dẫn đầy đủ.
   - "Chưa có" → hỏi workflow tự sinh (node `random`, chỉ hợp với việc như đăng ký) hay user sẽ chuẩn bị file theo **mẫu cột** (§2c).
2. (Nếu có phần B) **Có muốn ghi kết quả mỗi lượt chạy vào trang tính không?** Nêu lợi ích (xem lại cái đã làm, thành công/lỗi, không làm trùng, dễ lọc/xuất) và đề xuất sẵn cột (§4).
3. **Google Sheet hay file trên máy?** Khuyên Google Sheet nếu chạy nhiều luồng (ghi song song an toàn — `hideauto-shared-data-files` §3). Ghi vào Google Sheet cần Service Account JSON và phải chia sẻ sheet cho email của Service Account; đọc thì sheet phải để "ai có link đều xem được".

Nếu yêu cầu của user đã trả lời sẵn thì **không hỏi lại**. User từ chối ghi kết quả → tôn trọng, chỉ nhắc một lần là thêm sau được.

## 2. Có đường dẫn → phân tích file rồi tự dựng node đọc

### 2a. Xem cấu trúc thật (đừng bắt user gõ lại cột)

- **File .xlsx / .xls / .csv trên máy:** `spreadsheet_inspect` `mode: "overview"` với `source.path` = đường dẫn → các sheet, số dòng, header. Rồi `mode: "rows"` (`limit` nhỏ) để xem vài dòng mẫu.
- **Google Sheet:** `spreadsheet_inspect` chỉ đọc Google Sheet qua một node có sẵn → tạo node `readSpreadsheet` (source `googleSheet`, URL/tab qua biến) trước, rồi inspect bằng `source: { workflowId, nodeId }`. Lỗi không đọc được → thường do sheet chưa chia sẻ "ai có link đều xem được": báo user.
- **File .txt:** đọc vài dòng đầu bằng `file_read` (ngoài workspace có thể cần user cho phép). Không đọc được → hỏi user dán 2–3 dòng mẫu (che bớt mật khẩu). Xác định định dạng dòng, vd `email|password|refresh_token|client_id`.
- **Không in lại dữ liệu nhạy cảm** (mật khẩu, token) vào câu trả lời — chỉ nói tên cột và định dạng.

### 2b. Ghép cột → dữ liệu node cần

- Lập bảng ghép: dữ liệu node cần ↔ cột trong file, vd `password` ↔ cột `pass`. Cột rõ nghĩa thì tự ghép; cột mơ hồ (`col3`, không header, hai cột giống nhau) → hỏi lại user bằng `ask_user`.
- Thiếu cột bắt buộc (vd không có `refresh_token` cho node đọc mail) → báo user, đừng tự chế giá trị.
- Không có header (`hasHeader: false`) → dùng chữ cột `A`, `B`, `C`… làm `columnKey`.

### 2c. Dựng node

- Đường dẫn file / link sheet / tên tab → **biến** của workflow (`variable_manage`), node dùng `{{key}}`.
- **Spreadsheet (.xlsx/.csv/Google):** node `readSpreadsheet` đặt **trước** các node dùng dữ liệu; `readSpreadsheetColumns` map từng cột cần → biến tên rõ nghĩa (`email`, `password`…); các node sau dùng `{{email}}`, `{{password}}`.
- **Danh sách dùng cho nhiều lượt chạy** (gần như luôn vậy) → áp dụng claim → verify → mark của `hideauto-shared-data-files` §2: cần cột `status` (`new`) và `row`. File chưa có → báo user thêm (hoặc tự thêm nếu là file local được phép sửa). Dùng `spreadsheet_inspect` `mode: "match"` với điều kiện `status = new` để kiểm còn bao nhiêu dòng dùng được.
- **File .txt một mục một dòng:** `readFile` (`readFileMode: "random"` + `readFileDeleteLineAfterRead: true` cho danh sách dùng chung — xem giới hạn ở `hideauto-shared-data-files` §3), rồi tách các phần của dòng (`email|password|…`) ra biến bằng node phù hợp (`extractionInText` / `runJs`). Nếu workflow chạy nhiều luồng, đề xuất user chuyển danh sách sang Google Sheet để đánh dấu được.
- Nhánh `error` của node đọc (hết dữ liệu / không đọc được file) → `addLog` rõ lý do → stop. Không chạy tiếp với biến rỗng.
- Lấy field đầy đủ từ template node (`graph_get_templates`), đừng đoán. Chạy thử `run_test` để chắc dữ liệu đọc ra đúng (`addLog` in ra cột không nhạy cảm, vd `{{email}}`).

### 2d. Chưa có tài nguyên → mẫu cột

Đưa user mẫu theo đúng việc, luôn kèm `row` + `status` (đổi cột theo việc thật):

| row | status | email | password | recovery_email | proxy |
|---|---|---|---|---|---|
| 2 | new | a@mail.com | ... | ... | ... |

Google Sheet: cột `row` = `=ROW()`. Xây workflow sẵn theo mẫu (đường dẫn để trong biến) để user chỉ cần điền dữ liệu và đặt giá trị biến.

## 3. Kiểm trước khi báo xong (phần đầu vào)

- [ ] Không có dữ liệu thật nào hard-code trong node.
- [ ] Đường dẫn/link nằm trong biến; node đọc đặt trước node dùng dữ liệu; mọi cột cần đều map vào biến.
- [ ] Danh sách dùng chung có đánh dấu (`hideauto-shared-data-files` §2).
- [ ] Nhánh lỗi của node đọc dẫn tới `addLog` + stop.

## 4. Trang tính theo dõi kết quả (output)

Mỗi lượt chạy (hoặc mỗi mục xử lý) **append một dòng**. Cột đề xuất — chọn cột hợp với việc, đừng thừa:

| Cột | Nội dung |
|---|---|
| `time` | Thời điểm chạy |
| `profile` | `{{profile_uuid}}` (sau `openConnect`) |
| `input` | Khoá của tài nguyên đã dùng, vd `{{email}}` — để đối chiếu với danh sách đầu vào |
| (thông tin mới sinh ra) | vd `username`, `password`, `post_url`, `order_id`, `token`, `code` |
| `status` | `success` / `failed` |
| `note` | Lý do lỗi / ghi chú |

Cách làm:
- Node `writeSpreadsheet` với `writeSpreadsheetAppendLastRow: true`, `columnKey` = tên header, `value` = `{{biến}}`. Sheet nên tạo sẵn header trước.
- Thời điểm: không có biến thời gian sẵn → dùng `runJs` `return new Date().toLocaleString('sv-SE')` lưu vào `run_time` (`runJsSaveResultTo`).
- Ghi **cả khi lỗi**: nhánh `error` của các bước chính dẫn tới một node append với `status = failed` (và cập nhật `failed:` ở danh sách đầu vào). Dòng lỗi cũng quan trọng như dòng thành công.
- Ghi kết quả **ngay khi có** thông tin mới (vd vừa tạo xong tài khoản), đừng dồn tới cuối workflow — bước sau lỗi thì thông tin đã tạo vẫn không mất.
- Nhiều luồng: dùng Google Sheet append, hoặc mỗi luồng một file (`hideauto-shared-data-files` §3). Không cho nhiều luồng ghi chung một file local.
- Đường dẫn / URL sheet kết quả để trong biến (vd `result_sheet`, `result_tab`).

## 5. Lưu ý với user

- Sheet đầu vào và kết quả có thể chứa **mật khẩu, token, cookie** — nhắc user giữ quyền chia sẻ ở mức riêng tư (chỉ Service Account + người cần xem; sheet đầu vào nếu buộc phải "ai có link đều xem được" thì đừng phát tán link).
- Khi báo xong: nói rõ sheet/file nào là đầu vào, sheet nào là kết quả, cột đã map vào biến nào, và user cần chuẩn bị gì (header, cột `status`/`row`, giá trị `new`, chia sẻ cho Service Account).

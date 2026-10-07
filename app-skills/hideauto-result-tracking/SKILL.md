---
name: "hideauto-result-tracking"
description: "Khi user nhờ làm workflow/kịch bản/script mà mỗi lần chạy tạo ra thông tin mới hoặc có tác động thật (đăng ký tài khoản, tạo email, đăng bài, comment, đặt hàng, lấy mã/token, cào dữ liệu…): TRƯỚC khi xây, hỏi user đã có sẵn danh sách tài nguyên (email, tài khoản, proxy, địa chỉ, nội dung…) chưa, và chủ động gợi ý ghi kết quả mỗi lượt chạy vào trang tính để theo dõi/xem lại."
---

# hideauto-result-tracking

Khi đã chốt dùng file/sheet thì áp dụng tiếp `hideauto-shared-data-files` (biến cho đường dẫn, đánh dấu dòng, ghi song song an toàn).

## 0. Khi nào áp dụng

Áp dụng khi workflow mỗi lần chạy **sinh ra thông tin mới** hoặc **có tác động thật** mà user sau này cần tra lại, ví dụ:

- Đăng ký / tạo tài khoản, tạo email, đổi mật khẩu, bật 2FA.
- Đăng bài, comment, nhắn tin, follow, đặt hàng, nạp/rút, gửi form.
- Lấy mã, token, cookie, link, mã đơn hàng; cào/thu thập dữ liệu.
- Workflow sẽ chạy hàng loạt trên nhiều profile.

Không cần áp dụng với workflow chỉ xem/đọc một lần, không tạo ra gì đáng lưu.

## 1. Hỏi trước khi xây (một lượt hỏi gọn, không hỏi lắt nhắt)

Trước khi sửa graph, hỏi user bằng `ask_user` (có sẵn lựa chọn). Gộp các câu sau vào **một lần hỏi**:

1. **Đã có sẵn danh sách tài nguyên chưa?** Liệt kê cụ thể theo đúng việc user nhờ, vd: danh sách email (+ mật khẩu), tài khoản, số điện thoại, proxy, địa chỉ, tên/ảnh, nội dung bài đăng, link cần xử lý…
   - Có → nằm ở đâu (Google Sheet / .xlsx / .csv / .txt)? Xem cột thật bằng `spreadsheet_inspect` thay vì bắt user gõ lại.
   - Chưa → workflow tự sinh (node `random`: email, tên, mật khẩu…), hay user sẽ chuẩn bị file? Nếu user chuẩn bị, đưa họ **mẫu cột** (mục 2).
2. **Có muốn ghi kết quả mỗi lượt chạy vào trang tính để theo dõi không?** Gợi ý rõ lợi ích: xem lại tài khoản đã tạo, cái nào thành công/lỗi, không làm trùng, dễ lọc và xuất. Đề xuất sẵn danh sách cột (mục 3) để user chỉ cần đồng ý hoặc sửa.
3. **Dùng Google Sheet hay file trên máy?** Khuyên Google Sheet nếu sẽ chạy nhiều luồng (lý do: ghi song song an toàn — xem `hideauto-shared-data-files` §3). Google Sheet cần Service Account JSON để ghi và phải chia sẻ sheet cho email của Service Account.

Nếu user đã nói rõ các điểm trên trong yêu cầu thì **không hỏi lại** — dùng luôn. Nếu user từ chối ghi kết quả thì tôn trọng, chỉ nhắc một lần rằng có thể thêm sau.

## 2. Danh sách đầu vào (tài nguyên)

- Mỗi tài nguyên một dòng, dòng đầu là header. Ngoài cột dữ liệu, **luôn có** `status` (giá trị ban đầu `new`) và `row` (số dòng, Google Sheet dùng `=ROW()`) để đánh dấu đã dùng — chi tiết ở `hideauto-shared-data-files` §2.
- Mẫu gợi ý (đổi cột theo việc thật):

| row | status | email | password | recovery_email | proxy |
|---|---|---|---|---|---|
| 2 | new | a@mail.com | ... | ... | ... |

## 3. Trang tính theo dõi kết quả (output)

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
- Ghi **cả khi lỗi**: nhánh `error` của các bước chính dẫn tới một node append với `status = failed` (và cập nhật `failed:` ở danh sách đầu vào). Dòng kết quả lỗi cũng quan trọng như dòng thành công.
- Ghi kết quả **ngay khi có** thông tin mới (vd vừa tạo xong tài khoản), đừng dồn tới cuối workflow — nếu bước sau lỗi, thông tin đã tạo vẫn không mất.
- Nhiều luồng: dùng Google Sheet append, hoặc mỗi luồng một file (`hideauto-shared-data-files` §3). Không cho nhiều luồng ghi chung một file local.
- Đường dẫn / URL sheet kết quả để trong biến (vd `result_sheet`, `result_tab`).

## 4. Lưu ý với user

- Sheet kết quả có thể chứa **mật khẩu, token, cookie** — nhắc user giữ quyền chia sẻ sheet ở mức riêng tư (chỉ Service Account + người cần xem).
- Khi báo xong: nói rõ sheet nào là đầu vào, sheet nào là kết quả, các cột đã dùng, và user cần chuẩn bị gì (header, giá trị `new`, chia sẻ cho Service Account).

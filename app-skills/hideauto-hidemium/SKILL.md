---
name: "hideauto-hidemium"
description: "Workflow dùng profile Hidemium (Open & Connect chế độ otherApp / Hidemium), tra cứu profile Hidemium, lập lịch cho chúng, và chuyển file script Hidemium (.txt) thành workflow HideAuto (hidemium_import)."
---

# hideauto-hidemium

## Chạy workflow trên profile Hidemium

`openConnectMode: 'otherApp'` = Hidemium: profile nằm trong app Hidemium, app đó phải đang chạy.

- Tìm profile bằng `hidemium_profiles` (chỉ đọc, không mở/đóng được profile):
  - `sources` → gói Hidemium của user cho phép nguồn `local` / `cloud` nào;
  - `list{source, search?, folderIds?, tagIds?}` → danh sách profile [{uuid, name}];
  - `folders` / `tags` để lấy id lọc (id thuộc riêng một nguồn).
- Chạy thử: điền `openConnectBrowserUuid` = `uuid` và `openConnectBrowserName` = `name` của profile, rồi `run_test`.
- Lập lịch cho workflow đó: `schedule_manage` với `hidemiumProfiles` [{uuid, name}], không bao giờ dùng `profiles` (xem skill `hideauto-schedule-runs`).
- Lỗi kết nối ("Cannot connect to Hidemium at …") nghĩa là Hidemium chưa chạy hoặc sai địa chỉ: mở Cài đặt bằng `ui_navigate` mục `hidemium` và nhờ user kiểm tra — không thử lại liên tục.

## Nhập file script Hidemium (.txt)

Tool `hidemium_import`:

1. `preview{path}` trước — không ghi gì. Nói cho user bằng lời thường sẽ tạo gì: thư mục, các workflow, bao nhiêu bước cần kiểm tra.
2. `import{path, folder?}` — lưu mọi script trong file vào một thư mục MỚI (không ghi đè). `role: child` = script được script khác trong cùng file gọi (Run Other Workflow).
3. Bước Hidemium không có tương đương thành bước giữ chỗ có gắn cờ cần kiểm tra. Với mỗi workflow có `reviewNodeCount` > 0: `critic_review`, rồi sửa các bước đó trước khi chạy thử.

Nếu tool nhắc tới ở đây không có trong danh sách tool của bạn, app đang cũ hơn hướng dẫn này: báo user cập nhật app.

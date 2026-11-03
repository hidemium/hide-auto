---
name: "hideauto-schedule-runs"
description: "Tạo/sửa lịch chạy (schedule_manage) để chạy hàng loạt thật: gửi loại profile nào, lịch cho workflow Hidemium, vị trí cửa sổ browser khi lịch chạy (window_arrangement_manage), và xem lại các lần chạy trước (run_history)."
---

# hideauto-schedule-runs

Chạy hàng loạt thật trên profile thật luôn đi qua `schedule_manage` và thẻ xin duyệt của user. Không bao giờ tự lập lịch khi user không yêu cầu.

## Trước khi tạo lịch

- Workflow phải chạy thử sạch trước (`run_test` / `critic_review`).
- Cần đủ: workflow, profile, thời gian. Thiếu cái nào → `ask_user` (một lần gọi, nhiều câu hỏi). Đoán sai ở đây rất tốn kém.
- Loại profile theo chế độ Open & Connect của workflow:
  - chế độ có sẵn trong app → `profiles` = id/path từ `profile_manage list`;
  - chế độ Hidemium (`openConnectMode: 'otherApp'`) → `hidemiumProfiles` [{uuid, name}] từ `hidemium_profiles list`, không bao giờ dùng `profiles` (xem skill `hideauto-hidemium`).
- `run_now` trả `started:false` là CHƯA chạy — báo đúng `reason`, không được nói "đang chạy".
- `run_stop` dừng được lần chạy của lịch nhưng cần user duyệt — chỉ đề xuất khi user muốn.

## Vị trí cửa sổ browser khi lịch chạy

Chỉ browser do LỊCH mở mới được sắp xếp (chạy thử và mở tay thì không). Mục Cài đặt `displays`.

- Cài đặt chung của máy: `window_arrangement_manage`. Gọi `get` trước để biết các màn hình (`key`), `maxGrid` và `capacity`; `set` chỉ đổi phần gửi lên; số cột/hàng bị giới hạn theo `maxGrid`.
- `capacity` = cột × hàng × số màn đang dùng, chia chung cho MỌI lịch chạy cùng lúc. Nếu các lịch chạy chồng nhau mở nhiều cửa sổ hơn thế, báo user.
- Riêng một lịch: `schedule_manage` `arrangement` = `inherit` (mặc định) · `off` · `custom{các field layout cần đổi}`.

## Các lần chạy trước

Khi user hỏi "cái gì lỗi / lần chạy trước thế nào / dạo này có ổn không":

1. `run_history` (`list` / `get` / `stats`, lọc `scheduleId` nếu hỏi về một lịch);
2. `run_log_read` với `runId` và kind: chạy thử → `test`, chạy theo lịch → `campaign`.

Không đoán từ graph.

Nếu tool nhắc tới ở đây không có trong danh sách tool của bạn, app đang cũ hơn hướng dẫn này: báo user cập nhật app.

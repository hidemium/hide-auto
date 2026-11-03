---
name: "hideauto-proxy"
description: "Proxy trên node Open & Connect (proxy tự nhập hoặc proxy xoay HideProxy: field quốc gia/bang/thành phố) và kết nối tài khoản HideProxy của user (hideproxy_manage)."
---

# hideauto-proxy

## Proxy trên `openConnect`

Với ba chế độ có sẵn (`defaultProfile`, `newProfile`, `existedProfile`): đặt `openConnectProxyEnabled: true` và chọn `openConnectProxyProvider`:

- `manual` → `openConnectProxyType` = `http` / `socks` và `openConnectProxyServer` dạng `host:port`.
- `hideproxy` (mặc định) → proxy xoay từ tài khoản HideProxy đã kết nối ở Cài đặt → Tài khoản HideProxy. Chưa kết nối thì node lỗi lúc chạy (xem phần kết nối bên dưới).
  - `openConnectHideproxyCountry` KHÔNG bắt buộc và viết HOA (vd `"US"`). Để trống = quốc gia ngẫu nhiên, khi đó `openConnectHideproxyState` / `openConnectHideproxyCity` cũng PHẢI trống (bang mà không có quốc gia sẽ bị bỏ).
  - `openConnectHideproxyProxyId` chọn proxy xoay nào của tài khoản — để TRỐNG cho tự chọn (proxy còn nhiều dung lượng nhất) trừ khi user chỉ định.
  - Có thể thu hẹp bằng `openConnectHideproxyState`, rồi `openConnectHideproxyCity` (thành phố cần có bang). Giá trị là mã máy viết thường (vd `"california"`); không chắc mã thì chỉ đặt quốc gia.
  - KHÔNG đặt `openConnectProxyType` / `Server` / `Username` / `Password` khi provider là `hideproxy`.

## Kết nối tài khoản HideProxy

Mục Cài đặt `hideproxy-integration`.

1. Bắt đầu bằng `hideproxy_manage status` và làm theo `nextStep` + `guide`, MỖI LẦN MỘT BƯỚC: nói user bấm gì, chờ họ làm, rồi kiểm tra `status` lại.
2. Mọi link trong câu trả lời viết dạng markdown link, và nhắc nút "Mở trang HideProxy" dưới bước tool (nút này tự đăng nhập khi được).
3. Chỉ đưa URL HideProxy đúng như `hideproxy_manage link` trả về — không tự gõ hay đoán.
4. API key là bí mật: `credentials_manage` `request` target `hideproxy_api_key` — không bao giờ bảo user gõ vào khung chat.

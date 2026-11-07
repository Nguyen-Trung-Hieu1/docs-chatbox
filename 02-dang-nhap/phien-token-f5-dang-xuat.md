# 02.2. Phiên đăng nhập, F5 và đăng xuất

Sau đăng nhập, **access token của user ở RAM của tab**, còn **refresh token ở cookie HttpOnly** do backend đặt. Backend không lưu hai token này vào MongoDB trong luồng đăng nhập Keycloak hiện tại.

## Khi FE gọi một API bình thường

1. [`authenticatedFetch()`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/services/api.js) lấy access token trong RAM và gửi `Authorization: Bearer <access token>` tới BE, ví dụ `GET /backend/api/me`.
2. Nginx frontend chuyển thành `GET /api/me`. Middleware `authn` ở [`server.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/server.go) kiểm tra access token bằng public key Keycloak lấy từ endpoint `/protocol/openid-connect/certs`. BE kiểm tra chữ ký, issuer, hạn token, client/type và lấy `sub`.
3. BE dùng `sub` để tìm user MongoDB rồi cho handler chạy. **BE không gửi access token tới Keycloak token endpoint trong mỗi request**. Nếu bật đồng bộ role thì có thêm request Admin API riêng; phần này nằm ở mục phân quyền.

## Khi người dùng bấm F5

```text
F5 → RAM mất access token → FE gọi BE /api/auth/refresh
   → browser tự gửi refresh cookie → BE gọi Keycloak token endpoint
   → Keycloak trả access token mới → BE trả về FE → trang chat hiện lại
```

1. [`App.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/App.js) chạy lại, hiện “Đang kiểm tra phiên đăng nhập...” và gọi `refreshSession()`.
2. FE gửi `POST /backend/api/auth/refresh` với `credentials: include`. Browser tự gắn cookie `chatbox_refresh`; JavaScript không đọc được giá trị cookie vì cookie có cờ `HttpOnly`.
3. Handler `refreshKeycloakToken()` ở [`keycloak_session.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/keycloak_session.go) đọc cookie rồi gọi `POST https://auth.chatbox.cyou/realms/chatbox/protocol/openid-connect/token` với `grant_type=refresh_token`, `client_id=chatbox-web`, `refresh_token=<giá trị cookie>`.
4. Keycloak trả access token mới. Nếu Keycloak xoay vòng refresh token, BE cập nhật cookie. BE kiểm tra access token mới, lấy user MongoDB và đồng bộ role nếu được bật. BE trả `{user, access_token, expires_in}` cho FE.
5. FE giữ access token mới trong RAM, đặt lại `currentUser`. Nếu refresh token đã hết hạn hoặc bị thu hồi, request thất bại và người dùng phải đăng nhập lại.

Cookie refresh trong BE có `HttpOnly`, path `/api/auth`, thời gian cookie tối đa 30 ngày; HTTPS production bật `Secure`. [Nginx](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/nginx.conf) đổi cookie path thành `/backend/api/auth`, đúng với URL browser gọi. **30 ngày của cookie không có nghĩa phiên Keycloak chắc chắn sống 30 ngày**; Keycloak có thời hạn phiên riêng.

## Nếu access token hết hạn khi đang dùng

Khi API trả `401` và FE đang có access token, `authenticatedFetch()` gọi `/api/auth/refresh` rồi thử lại request ban đầu **một lần**. Các request refresh đồng thời trong cùng tab dùng chung một `refreshInFlight`. Nếu làm mới thất bại, FE xóa access token và có thể quay về màn hình đăng nhập.

## Khi người dùng bấm Đăng xuất

1. FE gọi `POST /backend/api/auth/logout` kèm cookie.
2. BE gửi refresh token tới `POST https://auth.chatbox.cyou/realms/chatbox/protocol/openid-connect/revoke`, xóa refresh cookie và trả `logout_url`.
3. FE xóa access token trong RAM rồi chuyển trình duyệt sang trang logout của Keycloak. Keycloak kết thúc phiên và chuyển về trang chủ Chatbox.

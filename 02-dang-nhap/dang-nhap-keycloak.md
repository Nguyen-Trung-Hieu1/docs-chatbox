# 02.1. Người dùng đăng nhập bằng Keycloak

Đây là luồng khi người dùng **bấm Đăng nhập** trên Chatbox. Nút **Tạo tài khoản** đi cùng luồng, chỉ mở trang đăng ký của Keycloak thay cho trang đăng nhập.

```text
Người dùng bấm Đăng nhập
  → FE chuyển trình duyệt sang Keycloak
  → người dùng nhập tài khoản ở Keycloak
  → Keycloak chuyển về FE kèm authorization code
  → FE gửi code cho BE
  → BE đổi code lấy token ở Keycloak
  → BE xác định user trong MongoDB
  → BE trả user + access token cho FE, đặt refresh cookie
  → FE mở trang chat
```

## Từng bước

1. Nút Đăng nhập trong [`Auth.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/Auth.js) gọi `startKeycloakLogin(false)` ở [`keycloak.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/keycloak.js). FE tạo `state`, `nonce` và cặp PKCE `verifier/challenge`. Nó chỉ giữ `state`, `nonce`, `verifier` tạm trong `sessionStorage` để kiểm tra callback. Đây chưa phải token đăng nhập.
2. FE chuyển trình duyệt tới `GET https://auth.chatbox.cyou/realms/chatbox/protocol/openid-connect/auth`. Request có `client_id=chatbox-web`, `response_type=code`, `redirect_uri=https://chatbox.cyou/auth/callback`, `scope=openid profile email`, `state`, `nonce` và `code_challenge`. Với nút Tạo tài khoản, đường dẫn cuối là `/registrations` thay vì `/auth`.
3. Người dùng nhập thông tin trên trang Keycloak. Keycloak kiểm tra tài khoản của nó, sau đó chuyển trình duyệt về `https://chatbox.cyou/auth/callback?code=...&state=...`. `code` là mã ngắn hạn dùng để đổi token, không phải access token.
4. FE gọi `finishKeycloakLogin()` trong [`keycloak.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/keycloak.js): so `state` nhận về với giá trị đã lưu, kiểm tra thời gian không quá 5 phút, rồi gửi `POST /backend/api/auth/exchange` tới BE. Body gồm `code`, `code_verifier`, `redirect_uri`, `nonce`. Nginx bỏ tiền tố `/backend`, nên route Go nhận là `/api/auth/exchange`.
5. Handler `exchangeKeycloakCode()` ở [`keycloak_session.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/keycloak_session.go) kiểm tra dữ liệu và `redirect_uri`. BE gửi form tới `POST https://auth.chatbox.cyou/realms/chatbox/protocol/openid-connect/token` với `grant_type=authorization_code`, `client_id=chatbox-web`, `code`, `code_verifier`, `redirect_uri`. Đây là request **BE → Keycloak**. Luồng này không dùng secret của client `chatbox-backend-admin`.
6. Keycloak trả `access_token`, `id_token`, `refresh_token`, `expires_in`. BE kiểm tra chữ ký và claims của ID token, access token trong [`keycloak.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/delivery/http/keycloak.go), gồm `nonce`, issuer, thời hạn, client ID và cùng một `sub`. BE lấy `sub` để tìm hoặc tạo user MongoDB; xem [liên kết user](lien-ket-user-keycloak-mongo.md).
7. Nếu chế độ role Keycloak được bật, BE còn đồng bộ role/plan trước khi trả kết quả. Chi tiết đọc role nằm ở mục `03-phan-quyen`, không thuộc bước đổi authorization code.
8. BE trả JSON `{user, access_token, expires_in}` và đặt `refresh_token` vào cookie HttpOnly. FE giữ `access_token` trong biến RAM của [`services/api.js`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/frontend/src/services/api.js), đưa `user` vào React state và mở trang chat.

## Ba token nhận được ở bước 6

| Token | Được dùng ở đâu |
| --- | --- |
| `id_token` | BE kiểm tra danh tính và `nonce` lúc hoàn tất đăng nhập; không trả token này cho FE. |
| `access_token` của user | BE trả cho FE; FE giữ trong RAM và gửi `Authorization: Bearer ...` khi gọi API Chatbox. |
| `refresh_token` | BE đặt vào cookie HttpOnly của browser; dùng để xin access token mới. |

Nếu `state` sai hoặc thiếu `code`, FE dừng trước khi gọi BE. Nếu Keycloak không chấp nhận code/verifier, token không hợp lệ, hoặc bước tạo/tìm user thất bại, BE không trả đăng nhập thành công.

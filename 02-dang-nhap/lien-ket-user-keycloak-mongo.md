# 02.3. User Keycloak liên kết với user MongoDB như thế nào?

Keycloak và Chatbox có **hai ID user khác nhau**:

```text
Keycloak user: sub = "abc-123"
MongoDB users:  _id = ObjectID("...")
                keycloak_sub = "abc-123"
```

`sub` nằm trong token Keycloak và xác định tài khoản Keycloak. `_id` là ID bản ghi `users` trong MongoDB. Chatbox dùng Mongo `_id` để nối user với hội thoại, tin nhắn, ví/credit và API key; nó không thay Mongo `_id` bằng `sub`.

## Lúc đăng nhập hoặc refresh

1. BE kiểm tra token rồi đọc `sub`.
2. [`Auth.ResolveKeycloak()`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/usecase/auth.go) gọi [`UserRepository.UpsertKeycloak()`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/repositories.go).
3. Repository tìm Mongo user bằng `keycloak_sub = sub`. Nếu đã có, trả về **đúng bản ghi Mongo đó** và cập nhật một số thông tin profile. Nếu chưa có, tạo Mongo user mới với `keycloak_sub` và username nội bộ dạng `kc-<sub>`.
4. BE dùng Mongo user vừa tìm được cho các chức năng Chatbox. Index unique của `keycloak_sub` trong [`mongo.go`](https://github.com/Nguyen-Trung-Hieu1/Chatbox/blob/main/backend/internal/infrastructure/persistence/mongo.go) ngăn cùng một `sub` gắn cho hai Mongo user.

Code **không tự ghép user chỉ vì email hoặc username giống nhau**. Vì vậy, nếu một người đã có tài khoản Mongo cũ rồi tạo tài khoản Keycloak cùng email, lần đăng nhập Keycloak có thể tạo Mongo user **mới**. Cuộc trò chuyện và credit cũ vẫn gắn với Mongo `_id` cũ. Muốn giữ dữ liệu cũ, phải xác minh tài khoản đúng chủ rồi nối `keycloak_sub` vào **bản ghi Mongo cũ**, không tạo bản ghi khác.

Hiện trong repository vẫn có code đăng nhập legacy và route `/api/auth/legacy-login`, nhưng giao diện đã bỏ nút đăng nhập tài khoản cũ. Khi `KEYCLOAK_ROLES_ENABLED=true`, route legacy bị từ chối. Chưa có cơ chế tự chuyển toàn bộ tài khoản Mongo cũ sang Keycloak trong luồng đăng nhập này.

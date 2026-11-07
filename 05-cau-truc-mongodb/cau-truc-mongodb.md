# 05. Cấu trúc MongoDB của Chatbox

MongoDB lưu dữ liệu **của ứng dụng Chatbox**. Tài khoản, mật khẩu và role do Keycloak quản lý nằm trong PostgreSQL riêng của Keycloak. Backend Chatbox hiện dùng sáu collection chính trong MongoDB:

```text
MongoDB Chatbox
├─ users          tài khoản nội bộ của Chatbox
├─ conversations  thông tin các cuộc trò chuyện
├─ messages       từng tin nhắn trong cuộc trò chuyện
├─ wallets        plan, credit và lịch sử sử dụng
├─ api_keys       API key để gọi Chatbox từ IDE/ứng dụng khác
└─ sessions       phiên đăng nhập kiểu cũ
```

## Các collection chứa gì?

| Collection | Dữ liệu quan trọng | Dùng để làm gì? |
| --- | --- | --- |
| `users` | `_id`, `username`, `keycloak_sub`, tên hiển thị, email, ngày tạo | Mỗi người dùng có một ID Chatbox trong MongoDB. `keycloak_sub` nối bản ghi này với ID `sub` bên Keycloak. User cũ có thể chưa có trường này. |
| `conversations` | `_id`, `owner_id`, `title`, `summary`, ngày tạo và ngày cập nhật | Một bản ghi cho mỗi cuộc trò chuyện. `owner_id` trỏ tới `users._id`; `summary` giữ bản tóm tắt phần hội thoại cũ khi cần. |
| `messages` | `_id`, `conversation_id`, `role`, `content`, `summarized`, ngày tạo | Một bản ghi cho mỗi tin nhắn. `role` là `user` hoặc `assistant`; `content` hiện là text. `summarized` cho biết tin đó đã được gộp vào bản tóm tắt để làm ngữ cảnh AI. |
| `wallets` | `_id`, `plan`, `plan_source`, `balance`, `credit_reset_at`, `usage` | Ví credit của user. `_id` của ví chính là `users._id`. `usage` chứa các lượt sử dụng gần nhất. |
| `api_keys` | `_id`, `user_id`, `name`, `prefix`, `hash`, ngày tạo, lần dùng cuối, ngày thu hồi | API key của user. MongoDB giữ **hash**, không giữ nguyên chuỗi key bí mật đã cấp. |





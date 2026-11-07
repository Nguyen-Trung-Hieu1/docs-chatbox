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
| `sessions` | `_id`, `user_id`, `token_hash`, hạn và ngày tạo | Dữ liệu phiên đăng nhập **kiểu cũ**. Luồng Keycloak hiện tại không lưu access token hoặc refresh token vào collection này. |

Trong `users` còn có trường `password` và `is_admin` phục vụ tài khoản/luồng cũ. Khi dùng Keycloak để phân quyền, quyền admin hiện hành được đọc từ role `chatbox-admin`; plan hiện hành được đồng bộ từ role Keycloak vào `wallets`. `wallets.paid_until` là trường của cách cấp Paid có thời hạn trước đây; plan do Keycloak quản lý dùng `plan_source=keycloak` thay vì dựa vào thời hạn đó.

## Các bản ghi liên kết với nhau thế nào?

```text
Keycloak user.sub
        │
        └─ users.keycloak_sub
               │
               └─ users._id = A
                    ├─ conversations.owner_id = A
                    │    └─ messages.conversation_id = conversations._id
                    ├─ wallets._id = A
                    ├─ api_keys.user_id = A
                    └─ sessions.user_id = A  (phiên kiểu cũ)
```

Ví dụ: khi user đăng nhập qua Keycloak, backend lấy `sub`, tìm `users.keycloak_sub` để biết Mongo user ID `A`. Từ `A`, backend mới lấy đúng cuộc trò chuyện, ví và API key của người đó. Email giống nhau **không tự nối** hai bản ghi user. Nếu một user Keycloak bị tạo thành Mongo user mới, dữ liệu đang gắn với Mongo `_id` cũ không tự chuyển sang.

MongoDB ở đây không có khóa ngoại tự động như SQL. Backend phải tự kiểm tra `owner_id` khi đọc, đổi tên hoặc xóa cuộc trò chuyện. Khi xóa cuộc trò chuyện, backend xóa các `messages` của nó rồi mới xóa bản ghi `conversations`; hai bước đó không nằm trong một transaction chung.

## `wallets.usage` lưu một lượt dùng AI như thế nào?

Mỗi lượt sử dụng ghi model, loại lượt gọi (`chat`, `summary` hoặc `api`), số token đầu vào/đầu ra, credit đã dùng, trạng thái và thời gian. `summary` là lượt gọi AI để tóm tắt hội thoại cũ; `api` là lượt gọi qua API key. Ví chỉ giữ **1.000 bản ghi usage gần nhất**, nên thống kê trên trang admin dựa vào phần lịch sử đang còn trong ví, không phải toàn bộ lịch sử vô hạn.

`balance` là credit còn lại. Khi backend chuẩn bị gọi AI, nó tạm giữ credit; sau khi có kết quả thì tính số đã dùng và hoàn phần còn dư. `credit_reset_at` là mốc cấp lại credit. Mốc này được xử lý khi có request sau thời điểm đó, không phải một cron quét toàn bộ ví đúng từng giây.

## Index chính

- `users.username` là duy nhất. `users.keycloak_sub` cũng là duy nhất **khi trường này có giá trị string**.
- `conversations` có index theo `owner_id` và thời gian cập nhật để tải sidebar của một user theo thứ tự mới nhất.
- `messages` có index theo `conversation_id` và thời gian tạo để tải lịch sử tin nhắn.
- `api_keys.hash` là duy nhất; còn có index theo `user_id` và ngày tạo để liệt kê key của user.
- `sessions.token_hash` là duy nhất; `sessions.expires_at` có TTL index để xóa phiên kiểu cũ khi hết hạn.

Hiện collection `messages` chỉ có `content` dạng text. Chưa có cấu trúc lưu ảnh hoặc file đính kèm trong luồng chat hiện tại.

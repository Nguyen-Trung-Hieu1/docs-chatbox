# 03. Phân quyền plan và trang admin

Chatbox dùng ba role trong Keycloak:

| Role | Ý nghĩa |
| --- | --- |
| `chatbox-free` | Tài khoản Free. Đây là role mặc định cho user mới. |
| `chatbox-paid` | Tài khoản Paid, được dùng thêm các model Paid. |
| `chatbox-admin` | Được vào trang admin và đổi plan cho user khác. |


## Luồng 1: Chatbox xác định user đang dùng plan nào

```text
FE gửi request kèm access token của user
  → BE xác thực user, lấy Keycloak sub
  → BE dùng quyền của backend hỏi Keycloak role của sub đó
  → BE xác định Free/Paid/Admin
  → BE đồng bộ plan vào ví MongoDB
  → BE trả danh sách model và credit phù hợp cho FE
```

1. Sau đăng nhập, FE gửi access token của user khi gọi API Chatbox. BE kiểm tra token và lấy `sub`, tức ID của user bên Keycloak. BE dùng `sub` tìm đúng user Chatbox trong MongoDB.
2. BE dùng `client ID` và `client secret` của **backend service account** để xin một access token riêng từ Keycloak. Token này đại diện cho backend và có quyền đọc/sửa role.
3. BE gửi token service account trong một request khác tới Keycloak Admin API. Keycloak trả danh sách role của user. 
.

## Luồng 2: Admin xem user và đổi plan

```text
Admin mở trang admin → FE hỏi BE danh sách user
Admin chọn user → FE hỏi BE thống kê sử dụng
Admin gạt Free/Paid → FE yêu cầu BE đổi plan
  → BE kiểm tra quyền admin
  → BE tìm Keycloak sub của user được chọn
  → BE dùng service account sửa role trong Keycloak
  → BE xác nhận role đã đổi và cập nhật ví MongoDB
  → BE trả kết quả → FE hiện plan mới
```

1. Admin mở trang quản trị. FE lấy danh sách user từ BE. BE đọc role Keycloak để hiện đúng plan hiện tại. FE chỉ hiện trang admin cho tài khoản có quyền quản trị.
2. Khi admin chọn một user rồi gạt sang Paid hoặc Free, FE gửi  Mongo user ID và plan muốn đổi cho API quản trị của Chatbox.
3. BE xác thực access token của admin, đọc role `chatbox-admin` hiện tại và chỉ tiếp tục nếu admin thật sự có quyền. 
4. BE lấy `keycloak_sub` của user được chọn từ MongoDB. 
5. BE đọc role cũ để biết plan trước khi đổi. Sau đó BE dùng service account gọi Keycloak Admin API: **đổi sang Paid thì gán `chatbox-paid`**; **đổi về Free thì bảo đảm có `chatbox-free` và gỡ `chatbox-paid`**. BE đọc role lại để xác nhận thao tác thành công.
6. BE cập nhật plan và credit trong ví MongoDB theo plan mới. Khi bước này thành công, BE trả thông tin user đã cập nhật; FE đổi nhãn Free/Paid trên trang admin. 


## Khi người dùng gửi tin nhắn với một model
1. Frontend gửi tin nhắn và model người dùng chọn tới backend, kèm access token của họ.
2. Backend xác thực người dùng, đọc role hiện tại từ Keycloak và đồng bộ plan Free/Paid vào MongoDB.
3. Backend lấy danh sách model được phép theo plan đó, rồi kiểm tra lại model trong request. Danh sách hiện trên giao diện chỉ để người dùng chọn; quyền sử dụng do backend quyết định.
Nếu model nằm trong quyền: backend kiểm tra credit còn đủ, sau đó gửi câu hỏi tới 9Router bằng model đã chọn. Khi nhận câu trả lời, backend lưu tin nhắn, tính credit đã dùng và trả kết quả về frontend. Nếu credit không đủ, request dừng trước khi gọi 9Router.

Nếu model nằm ngoài quyền: backend trả lỗi “gói của bạn không được dùng model này”. Request dừng trước khi gọi 9Router; tin nhắn đó không được lưu và không bị trừ credit. Việc sửa request từ trình duyệt để gửi tên model Paid cũng không vượt qua được bước kiểm tra này.

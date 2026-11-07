# 03. Phân quyền plan và trang admin

Chatbox dùng ba role trong Keycloak:

| Role | Ý nghĩa |
| --- | --- |
| `chatbox-free` | Tài khoản Free. Đây là role mặc định cho user mới. |
| `chatbox-paid` | Tài khoản Paid, được dùng thêm các model Paid. |
| `chatbox-admin` | Được vào trang admin và đổi plan cho user khác. |

Một user có thể thấy cả `chatbox-free` và `chatbox-paid` vì Free là role mặc định được thừa kế. **Nếu có `chatbox-paid` thì backend coi là Paid; nếu không có thì coi là Free.** `chatbox-admin` là quyền quản trị riêng, không tự biến user thành Paid.

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
2. Nếu chế độ quản lý role bằng Keycloak được bật, BE cần biết **role hiện tại**, kể cả khi admin vừa đổi role sau lúc user đăng nhập. BE dùng `client ID` và `client secret` của **backend service account** để xin một access token riêng từ Keycloak. Token này đại diện cho backend và có quyền đọc/sửa role; nó không phải access token đăng nhập của user.
3. BE gửi token service account trong một request khác tới Keycloak Admin API, hỏi: “User có `sub` này hiện có những role nào?”. Keycloak trả danh sách role của user. BE nhìn xem có `chatbox-paid` và `chatbox-admin` không.
4. BE đồng bộ plan vừa đọc vào ví của user trong MongoDB. MongoDB vẫn giữ số credit, lịch sử sử dụng và bản plan phục vụ tính credit. **Nguồn quyết định quyền Free/Paid là role Keycloak**; nếu ai chỉ sửa plan trong MongoDB, lần đồng bộ tiếp theo có thể ghi đè theo role Keycloak.
5. Khi FE hỏi thông tin gói, BE đọc ví MongoDB sau khi đã đồng bộ, trả về plan, credit còn lại và các model được phép. FE dùng kết quả đó để hiện bộ chọn model. Khi user gửi tin nhắn, BE **kiểm tra lại** model và credit trước khi gọi 9Router; FE tự hiện thêm model không làm BE cho phép dùng nó.

Luồng đọc role xảy ra lúc hoàn tất đăng nhập, lúc khôi phục phiên và trên các API có xác thực bằng access token. Nếu Keycloak Admin API lỗi khi chế độ role đang bật, request bị lỗi thay vì âm thầm dùng quyền cũ trong MongoDB.

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

1. Admin mở trang quản trị. FE lấy danh sách user từ BE; danh sách user và thống kê sử dụng nằm trong MongoDB. Với user đã liên kết Keycloak, BE đọc role Keycloak để hiện đúng plan hiện tại. FE chỉ hiện trang admin cho tài khoản có quyền quản trị.
2. Khi admin chọn một user rồi gạt sang Paid hoặc Free, FE gửi **Mongo user ID** và plan muốn đổi cho API quản trị của Chatbox. FE không gọi thẳng Keycloak, cũng không có client secret của backend.
3. BE xác thực access token của admin, đọc role `chatbox-admin` hiện tại và chỉ tiếp tục nếu admin thật sự có quyền. Việc ẩn/hiện nút ở FE không phải bước kiểm tra quyền cuối cùng.
4. BE lấy `keycloak_sub` của user được chọn từ MongoDB. Đây là ID mà Keycloak hiểu; nếu user chưa được liên kết với Keycloak thì BE không thể sửa role theo luồng này và trả lỗi.
5. BE đọc role cũ để biết plan trước khi đổi. Sau đó BE dùng service account gọi Keycloak Admin API: **đổi sang Paid thì gán `chatbox-paid`**; **đổi về Free thì bảo đảm có `chatbox-free` và gỡ `chatbox-paid`**. BE đọc role lại để xác nhận thao tác thành công.
6. BE cập nhật plan và credit trong ví MongoDB theo plan mới. Khi bước này thành công, BE trả thông tin user đã cập nhật; FE đổi nhãn Free/Paid trên trang admin. Ở request kế tiếp của chính user đó, BE đọc role Keycloak mới và áp dụng model tương ứng.
7. Nếu cập nhật MongoDB thất bại sau khi Keycloak đã đổi role, BE cố gắng trả role Keycloak về plan cũ và báo lỗi. Hai hệ thống không có một transaction chung, nên nếu cả bước hoàn tác cũng lỗi thì cần kiểm tra và sửa lại trạng thái thực tế.

**Tóm lại:** Keycloak giữ quyền Free/Paid/Admin. MongoDB giữ user Chatbox, credit và thống kê. Backend là bên kiểm tra quyền và làm cầu nối giữa FE, Keycloak, MongoDB; FE không tự quyết định user được dùng model nào.

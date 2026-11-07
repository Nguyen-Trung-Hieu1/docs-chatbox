# 02. Luồng đăng nhập và duy trì phiên

## Người dùng đăng nhập

```text
Người dùng → Chatbox → Keycloak → Chatbox frontend → Chatbox backend
                                      ↑                    ↓
                                      └──── kết quả ── Keycloak + MongoDB
```

1. Người dùng bấm Đăng nhập trên Chatbox. Trình duyệt được chuyển sang trang đăng nhập của Keycloak. Nếu bấm **Tạo tài khoản**, trình duyệt chuyển sang trang đăng ký của Keycloak.
2. Người dùng nhập tài khoản và mật khẩu trên trang Keycloak. Keycloak kiểm tra thông tin đăng nhập.
3. Đăng nhập thành công, Keycloak chuyển trình duyệt về Chatbox kèm một mã đăng nhập tạm thời gọi là `code`.
4. Frontend Chatbox kiểm tra đây có đúng là lần đăng nhập nó vừa bắt đầu không, rồi gửi `code` cho backend Chatbox.
5. Backend gửi `code` tới Keycloak. Keycloak kiểm tra mã và trả về ba token: ID token (thông tin xác nhận người đăng nhập), access token(dùng khi gọi API) và refresh token(dùng để xin access token mới).
6. Backend kiểm tra token, lấy ID người dùng Keycloak là `sub`. Nó tìm user Chatbox trong MongoDB bằng `keycloak_sub = sub`. Nếu chưa có, backend tạo một user Chatbox mới. Hai hệ thống có ID riêng: Keycloak dùng `sub`, MongoDB dùng `_id`.
7. Nếu bật quản lý plan bằng Keycloak, backend kiểm tra role của người dùng để biết họ là Free/Paid và có quyền admin hay không. Phần role và plan sẽ được giải thích ở mục `03-phan-quyen`.
8. Backend trả thông tin user và **access token** cho frontend, đồng thời đặt **refresh token** vào cookie `HttpOnly` của trình duyệt. Frontend giữ access token trong RAM và hiển thị trang chat.

## Sau khi đã đăng nhập

Khi frontend gọi API Chatbox, nó gửi access token kèm request. Backend kiểm tra token, xác định người dùng qua `sub`, tìm user tương ứng trong MongoDB rồi mới xử lý request. Frontend không phải gửi lại mật khẩu. Backend cũng không cần đổi `code` hay xin token đăng nhập mới cho **mỗi** request.

## Khi bấm F5 hoặc access token hết hạn

1. F5 làm trang tải lại nên access token trong RAM bị mất. Cookie chứa refresh token vẫn còn nếu chưa hết hạn.
2. Frontend yêu cầu backend khôi phục phiên. Trình duyệt tự gửi refresh cookie theo request; JavaScript trên trang không đọc được giá trị cookie `HttpOnly`.
3. Backend dùng refresh token đó xin token mới từ Keycloak, kiểm tra token và xác định lại user Chatbox.
4. Backend trả access token mới cho frontend. Frontend giữ nó trong RAM, nên người dùng tiếp tục sử dụng mà không cần đăng nhập lại.
5. Nếu refresh token hoặc phiên Keycloak đã hết hạn, việc khôi phục thất bại và người dùng phải đăng nhập lại. Khi access token hết hạn giữa lúc đang dùng, frontend cũng thử làm mới phiên rồi gửi lại request một lần.

## Khi đăng xuất

1. Frontend yêu cầu backend đăng xuất.
2. Backend yêu cầu Keycloak thu hồi refresh token và xóa refresh cookie trong trình duyệt.
3. Frontend xóa access token trong RAM rồi chuyển trình duyệt qua trang đăng xuất Keycloak. Sau đó người dùng trở về Chatbox ở trạng thái chưa đăng nhập.

**Lưu ý về tài khoản cũ:** user MongoDB có cùng email với user Keycloak không tự động được ghép thành một tài khoản. Để giữ hội thoại và credit cũ, cần liên kết đúng `sub` với **bản ghi MongoDB cũ**; luồng đăng nhập hiện tại chưa tự làm bước chuyển dữ liệu này.

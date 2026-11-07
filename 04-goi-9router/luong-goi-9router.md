# 04. Luồng gọi 9Router

9Router là cổng AI đứng giữa **backend Chatbox** và các model. Trình duyệt không gọi trực tiếp 9Router. Một lần người dùng bấm **Gửi** đi theo đường sau:

```text
Người dùng → frontend → backend Chatbox → 9Router → model AI
                         ↑                    ↓
                         └── câu trả lời + usage
                   backend → frontend → người dùng
```

## Từng bước khi gửi câu hỏi

1. Frontend gửi nội dung câu hỏi, cuộc trò chuyện và model đã chọn tới backend Chatbox. Backend xác thực người dùng, xác nhận cuộc trò chuyện thuộc người đó và kiểm tra model có nằm trong plan hiện tại không. Nếu model không được phép, request dừng tại backend, chưa tới 9Router.
2. Backend chuẩn bị nội dung gửi AI: bản tóm tắt hội thoại cũ nếu có, những tin nhắn gần đây và câu hỏi mới. 
3. Backend ước lượng lượng credit cần cho lượt trả lời và tạm giữ trong ví MongoDB. Nếu không đủ credit, backend báo lỗi và không gửi request  tới 9Router.
4. Backend gửi request tới địa chỉ 9Router được cấu hình trong backend. Request có model đã được kiểm tra, danh sách tin nhắn/ngữ cảnh và giới hạn câu trả lời. 
5. 9Router nhận request, định tuyến tới model tương ứng rồi chuyển câu trả lời về backend. Backend nhận từng phần nội dung ngay khi model tạo ra, sau đó chuyển tiếp từng phần cho frontend. 
6. Khi 9Router kết thúc, backend lấy số token đầu vào/đầu ra mà gateway báo. Backend tính credit đã dùng, hoàn lại phần tạm giữ còn dư và ghi lịch sử sử dụng vào MongoDB.
7. Backend lưu câu trả lời AI vào cuộc trò chuyện rồi báo frontend lượt trả lời đã hoàn tất. Frontend thay câu trả lời tạm trên màn hình bằng bản đã lưu.


## Nếu request bị lỗi hoặc người dùng bấm Dừng

- Nếu 9Router/model trả lỗi trước khi có câu trả lời, backend báo lỗi cho frontend. Khoản credit tạm giữ cho lượt trả lời chính được quyết toán theo usage thực tế; nếu chưa có nội dung và không có usage thì được hoàn lại.
- Nếu đã có một phần câu trả lời rồi kết nối bị ngắt hoặc người dùng bấm Dừng, phần đã tạo được lưu; usage của phần đó vẫn có thể được tính. Tin nhắn người dùng có thể đã được lưu trước lúc ngắt.


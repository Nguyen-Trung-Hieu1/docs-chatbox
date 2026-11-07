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
2. Backend chuẩn bị nội dung gửi AI: hướng dẫn chung cho trợ lý, bản tóm tắt hội thoại cũ nếu có, những tin nhắn gần đây và câu hỏi mới. Nhờ vậy model biết ngữ cảnh cuộc trò chuyện thay vì chỉ nhìn câu hỏi cuối.
3. Backend ước lượng lượng credit cần cho lượt trả lời và tạm giữ trong ví MongoDB. Nếu không đủ credit, backend báo lỗi và không gửi **request trả lời chính** tới 9Router.
4. Backend gửi request tới địa chỉ 9Router được cấu hình trong backend. Request có model đã được kiểm tra, danh sách tin nhắn/ngữ cảnh và giới hạn câu trả lời. Nếu 9Router yêu cầu API key, backend tự gắn key từ cấu hình bí mật; frontend không nhận key này.
5. 9Router nhận request, định tuyến tới model tương ứng rồi chuyển câu trả lời về backend. Backend nhận từng phần nội dung ngay khi model tạo ra, sau đó chuyển tiếp từng phần cho frontend. Vì vậy người dùng thấy câu trả lời hiện dần, không cần chờ toàn bộ câu trả lời xong.
6. Khi 9Router kết thúc, backend lấy số token đầu vào/đầu ra mà gateway báo. Nếu không có usage, backend ước lượng từ nội dung. Backend tính credit đã dùng, hoàn lại phần tạm giữ còn dư và ghi lịch sử sử dụng vào MongoDB.
7. Backend lưu câu trả lời AI vào cuộc trò chuyện rồi báo frontend lượt trả lời đã hoàn tất. Frontend thay câu trả lời tạm trên màn hình bằng bản đã lưu.

## Vì sao một lần bấm Gửi có thể gọi 9Router hơn một lần?

Nếu hội thoại quá dài, backend có thể gửi **một request phụ** tới 9Router để tóm tắt các tin nhắn cũ trước khi gửi request trả lời chính. Bản tóm tắt được lưu để dùng làm ngữ cảnh cho những lượt sau. Request tóm tắt cũng có usage và credit riêng. Vì vậy số lượt gọi 9Router có thể nhiều hơn số lần người dùng bấm Gửi.

## Nếu request bị lỗi hoặc người dùng bấm Dừng

- Nếu 9Router/model trả lỗi trước khi có câu trả lời, backend báo lỗi cho frontend. Khoản credit tạm giữ cho lượt trả lời chính được quyết toán theo usage thực tế; nếu chưa có nội dung và không có usage thì được hoàn lại.
- Nếu đã có một phần câu trả lời rồi kết nối bị ngắt hoặc người dùng bấm Dừng, phần đã tạo có thể được lưu; usage của phần đó vẫn có thể được tính. Tin nhắn người dùng có thể đã được lưu trước lúc ngắt.
- Nếu 9Router không báo usage, số token/credit trên Chatbox là **ước lượng của backend**, không phải số đo chính xác do gateway trả về.

Credit ở đây là **đơn vị nội bộ của Chatbox**, không phải số tiền mà 9Router hay nhà cung cấp model thu trực tiếp. Plan quyết định model được phép; MongoDB giữ số credit còn lại và lịch sử sử dụng.

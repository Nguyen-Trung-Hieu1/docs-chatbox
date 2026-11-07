# 06. Luồng chạy khi người dùng gửi một request chat

File này đi theo **một lần bấm Gửi trên trang Chatbox**, từ trình duyệt tới lúc câu trả lời hiện xong. Nếu người dùng đang ở cuộc trò chuyện mới, sẽ có thêm request tạo cuộc trò chuyện trước request chat.

```text
Người dùng bấm Gửi
  → frontend (React)
  → Nginx frontend
  → backend Chatbox (Go)
  ├─ Keycloak: backend dùng khóa công khai kiểm tra token, hỏi role hiện tại
  ├─ MongoDB: user, cuộc trò chuyện, ví, tin nhắn
  └─ 9Router → model AI
       └─ nội dung trả lời + usage → backend → frontend → người dùng
```

## A. Frontend chuẩn bị và gửi request

1. Người dùng nhập câu hỏi, chọn model rồi bấm **Gửi**. Nếu ô nhập rỗng hoặc đang có một lượt trả lời khác, frontend không gửi thêm request.
2. Frontend xóa nội dung ô nhập, hiện tạm câu hỏi của người dùng và một ô trả lời AI đang trống. Đây mới là **trạng thái trên màn hình**, chưa có nghĩa dữ liệu đã được lưu thành công trong MongoDB.
3. Nếu người dùng vừa bấm **Đoạn chat mới**, frontend chưa có ID cuộc trò chuyện. Nó gửi request tạo cuộc trò chuyện tới backend; backend gắn cuộc trò chuyện với user hiện tại, lưu vào MongoDB và trả về ID. Nếu đã chọn một cuộc trò chuyện cũ, frontend dùng ID sẵn có.
4. Frontend gửi request chat gồm **ID cuộc trò chuyện, nội dung câu hỏi và model đã chọn**, kèm access token đăng nhập của user. Trên production, request đi qua domain Chatbox và Nginx frontend rồi mới tới Go backend. Trình duyệt không gọi thẳng MongoDB, Keycloak Admin API hay 9Router.

## B. Backend xác định ai gửi và có được gửi không

5. Backend kiểm tra access token. Nếu token hết hạn và frontend nhận lỗi `401`, frontend thử dùng refresh cookie để xin access token mới rồi gửi lại request **một lần**. Nếu vẫn thất bại, lượt chat dừng và người dùng có thể phải đăng nhập lại.
6. Từ token hợp lệ, backend lấy `sub` của Keycloak, tìm user Chatbox tương ứng trong MongoDB. Khi chế độ role Keycloak được bật, backend đọc role hiện tại từ Keycloak và đồng bộ plan Free/Paid vào ví MongoDB. Như vậy admin vừa đổi plan thì request mới dùng quyền mới.
7. Backend kiểm tra nội dung câu hỏi không rỗng, ID cuộc trò chuyện có hợp lệ và cuộc trò chuyện đó **thuộc đúng user đang gửi**. User không thể đọc hoặc gửi tin vào cuộc trò chuyện của người khác bằng cách sửa ID trong request.
8. Backend lấy plan và danh sách model được phép, kiểm tra model trong request. Nếu model không thuộc plan, backend báo lỗi và **không gửi request trả lời tới 9Router, không lưu câu hỏi mới**. Nếu một cuộc trò chuyện mới đã được tạo ở bước 3, bản ghi cuộc trò chuyện trống đó có thể vẫn tồn tại.

## C. Backend chuẩn bị ngữ cảnh và credit

9. Backend lấy các tin nhắn trước đó của cuộc trò chuyện và bản tóm tắt cũ (nếu có), rồi thêm câu hỏi mới vào ngữ cảnh gửi AI. Ngữ cảnh không phải toàn bộ lịch sử vô hạn; backend ưu tiên các tin gần đây trong giới hạn đã cấu hình.
10. Nếu lịch sử quá dài, backend **có thể gọi 9Router thêm một lần để tóm tắt** phần cũ. Bản tóm tắt được lưu ở cuộc trò chuyện và lượt tóm tắt có usage/credit riêng. Vì vậy một lần bấm Gửi đôi khi tạo hơn một request tới 9Router.
11. Với request trả lời chính, backend ước lượng token và tính lượng credit cần tạm giữ theo model. Ví MongoDB chỉ cho tạm giữ nếu số dư còn đủ. Nếu không đủ, backend báo lỗi; câu hỏi mới chưa được lưu và request trả lời chính chưa được gửi tới 9Router. Lượt tóm tắt ở bước 10, nếu đã chạy, có thể đã phát sinh usage riêng.
12. Khi tạm giữ credit thành công, backend lưu câu hỏi của user vào collection `messages`. Nó cập nhật thời gian hoạt động của cuộc trò chuyện; nếu tiêu đề còn là “Cuộc trò chuyện mới”, phần đầu câu hỏi được dùng làm tiêu đề.

## D. Backend gọi 9Router và trả stream

13. Backend gửi ngữ cảnh, model đã qua kiểm tra và giới hạn độ dài câu trả lời tới địa chỉ 9Router trong cấu hình backend. Nếu cần API key cho 9Router, key nằm ở backend, không đi tới JavaScript của frontend.
14. 9Router chuyển request tới model tương ứng. Model tạo nội dung từng phần; 9Router chuyển các phần đó và số token sử dụng (nếu có) về backend.
15. Backend chuyển ngay từng phần nội dung về frontend qua response đang mở. Frontend nối các phần đó vào ô trả lời AI tạm trên màn hình, nên người dùng thấy chữ xuất hiện dần. Đây là hai chặng stream: **9Router → backend**, rồi **backend → frontend**.

## E. Kết thúc lượt chat

16. Khi 9Router trả xong, backend lấy usage do 9Router báo. Nếu không có usage, backend ước lượng từ ngữ cảnh và nội dung trả lời. Nó tính credit thực dùng, hoàn lại phần credit đã tạm giữ nhưng không dùng và ghi lượt usage vào ví MongoDB.
17. Backend lưu nội dung trả lời AI thành một tin nhắn mới trong `messages`, cập nhật cuộc trò chuyện rồi báo cho frontend rằng lượt chat đã hoàn tất.
18. Frontend thay ô trả lời tạm bằng tin nhắn đã lưu, tải lại danh sách cuộc trò chuyện và thông tin plan/credit để màn hình phản ánh dữ liệu mới.

## Nếu lỗi hoặc người dùng bấm Dừng giữa chừng

- **Lỗi trước bước lưu câu hỏi:** không có tin nhắn mới và request trả lời chính không tới 9Router. Riêng cuộc trò chuyện trống hoặc lượt tóm tắt đã tạo trước đó vẫn có thể tồn tại.
- **Lỗi sau khi đã lưu câu hỏi:** câu hỏi có thể vẫn nằm trong lịch sử. Nếu AI đã trả một phần nội dung, backend có thể lưu phần trả lời đó và tính usage tương ứng; lượt dùng được ghi là bị gián đoạn.
- **Người dùng bấm Dừng:** frontend hủy request đang mở. Backend dừng nhận stream theo kết nối bị hủy, cố quyết toán credit và lưu phần trả lời đã có. Vì các bước lưu dữ liệu và tính credit diễn ra lần lượt, một lượt chat không phải một transaction MongoDB duy nhất.

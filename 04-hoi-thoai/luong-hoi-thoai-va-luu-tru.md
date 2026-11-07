# 04. Luồng hội thoại và cách lưu trong MongoDB

Một cuộc trò chuyện có **tiêu đề, chủ sở hữu và thời gian cập nhật**. Các tin nhắn được lưu riêng và gắn với cuộc trò chuyện đó. Mỗi thao tác đi từ **frontend → backend → MongoDB**; khi cần tạo câu trả lời AI, backend mới gọi thêm 9Router.

## 1. Khi người dùng vào trang chat

1. Frontend yêu cầu backend lấy danh sách cuộc trò chuyện.
2. Backend xác thực người dùng, lấy ID user Chatbox trong MongoDB rồi chỉ tìm các cuộc trò chuyện có `owner_id` bằng ID đó.
3. MongoDB trả danh sách theo thời gian cập nhật, mới nhất trước. Frontend hiển thị danh sách ở thanh bên. Lịch sử tin nhắn của từng cuộc trò chuyện chưa cần tải hết ở bước này.

## 2. Khi bấm “Đoạn chat mới”

1. Frontend bỏ lựa chọn cuộc trò chuyện cũ và hiển thị khung chat trống.
2. **Lúc này chưa có cuộc trò chuyện mới trong MongoDB.** Chỉ khi người dùng gửi tin nhắn đầu tiên, frontend mới yêu cầu backend tạo cuộc trò chuyện.
3. Backend tạo một bản ghi trong `conversations`, gắn `owner_id` với user hiện tại và đặt tiêu đề ban đầu là “Cuộc trò chuyện mới”. MongoDB trả ID cuộc trò chuyện để frontend dùng khi gửi tin nhắn.

## 3. Khi gửi một tin nhắn

```text
Người dùng bấm Gửi
  → FE có ID cuộc trò chuyện (nếu chưa có thì tạo mới)
  → FE gửi nội dung + model đã chọn tới BE
  → BE kiểm tra chủ sở hữu, plan/model và credit
  → BE lưu tin nhắn người dùng vào MongoDB
  → BE gọi 9Router và chuyển dần câu trả lời về FE
  → BE lưu câu trả lời AI, cập nhật credit và cuộc trò chuyện
```

1. Frontend hiện tạm câu hỏi và một vị trí trống cho câu trả lời, để người dùng thấy tin nhắn đã được gửi. Nếu đây là chat mới, frontend tạo cuộc trò chuyện trước rồi dùng ID vừa nhận.
2. Backend xác thực user và kiểm tra cuộc trò chuyện có thuộc user đó không. Backend kiểm tra nội dung tin nhắn và model có nằm trong plan hiện tại không. Nếu model bị từ chối, backend không lưu tin nhắn và không gọi 9Router. Riêng trường hợp chat mới, bản ghi cuộc trò chuyện có thể đã được tạo ở bước trước.
3. Backend lấy các tin nhắn trước đó và bản tóm tắt nếu có để chuẩn bị ngữ cảnh. Khi lịch sử quá dài, bước chuẩn bị này có thể tạo một request AI riêng để tóm tắt tin cũ. Sau đó backend tính phần credit cần tạm giữ cho câu trả lời chính; nếu không đủ, nó dừng trước khi lưu câu hỏi mới hoặc gửi request trả lời chính tới 9Router.
4. Nếu đủ credit, backend tạm giữ credit, lưu tin nhắn của người dùng vào `messages`, rồi cập nhật thời gian hoạt động của cuộc trò chuyện. Với chat còn tiêu đề mặc định, backend lấy phần đầu câu hỏi đầu tiên làm tiêu đề. Backend gọi 9Router để trả lời; khi AI trả từng phần, backend chuyển từng phần về frontend để hiện dần. Cách trao đổi với 9Router và tính usage nằm ở mục `05-goi-9router`.
5. Khi hoàn tất, backend tính credit thực tế, hoàn lại phần tạm giữ chưa dùng, lưu câu trả lời AI vào `messages` và báo cho frontend biết lượt trả lời đã xong. Frontend thay câu trả lời tạm bằng dữ liệu đã lưu, đồng thời tải lại danh sách cuộc trò chuyện và số credit.

Nếu người dùng bấm **Dừng** hoặc kết nối bị ngắt sau khi backend đã lưu câu hỏi, câu hỏi có thể vẫn còn trong lịch sử. Nếu AI đã trả được một phần nội dung, phần đó cũng có thể được lưu; lượt dùng được ghi là bị gián đoạn. Một lần bấm Gửi không phải một giao dịch MongoDB duy nhất.

## 4. Khi mở lại một cuộc trò chuyện

1. Người dùng chọn cuộc trò chuyện ở thanh bên. Frontend gửi ID cuộc trò chuyện cho backend.
2. Backend kiểm tra ID đó thuộc user đang đăng nhập. Nếu không thuộc, backend từ chối thay vì trả lịch sử của người khác.
3. MongoDB trả các tin nhắn của cuộc trò chuyện theo thứ tự thời gian. Frontend hiển thị lại cả câu hỏi lẫn câu trả lời đã lưu.

## 5. Khi đổi tên hoặc xóa

- **Đổi tên:** frontend gửi ID và tiêu đề mới. Backend kiểm tra chủ sở hữu, kiểm tra tiêu đề hợp lệ rồi cập nhật `title` và thời gian cập nhật trong MongoDB. Frontend đổi tên tương ứng ở thanh bên.
- **Xóa:** frontend gửi ID cuộc trò chuyện. Backend kiểm tra chủ sở hữu, xóa các tin nhắn thuộc cuộc trò chuyện, sau đó xóa bản ghi cuộc trò chuyện. Frontend bỏ nó khỏi thanh bên. Hai bước xóa ở MongoDB chạy tuần tự, không nằm trong một transaction chung.

## 6. Dữ liệu nằm ở đâu?

```text
users._id = ID user Chatbox
  └─ conversations.owner_id = users._id
       ├─ title, summary, created_at, updated_at
       └─ messages.conversation_id = conversations._id
            └─ role (user/assistant), content, created_at
```

`users` xác định chủ sở hữu. `conversations` giữ thông tin từng cuộc trò chuyện. `messages` giữ nội dung từng tin; tin nhắn không được nhét hết vào một bản ghi conversation. Khi hội thoại dài, backend có thể lưu một bản tóm tắt trong `conversations.summary` và đánh dấu những tin cũ đã được tóm tắt để giảm ngữ cảnh gửi AI; **tin nhắn gốc vẫn còn** để người dùng xem lịch sử.

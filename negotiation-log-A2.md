# Biên bản đàm phán hợp đồng API

- **Cặp đàm phán**: Pair 01 — Camera Stream (A2/B2) ↔ AI Vision (A4/B4)
- **Product**: Product A
- **Provider**: AI Vision (A4) - Đại diện: Nguyễn Minh Mạnh
- **Consumer**: Camera Stream (A2) - Đại diện: [Điền tên bạn đại diện nhóm A2 vào đây]
- **Phiên**: v1.0
- **Ngày**: 28-05-2026

---

## Issue #1: Tối ưu hóa định dạng truyền tải hình ảnh đầu vào (imageUrl vs imageBase64)

- **Raised by**: Provider (AI Vision)
- **Endpoint**: `POST /vision/detect`
- **Concern**: Thiết kế sơ khai ban đầu cho phép truyền luồng ảnh thô mã hóa dạng chuỗi ký tự `imageBase64`. Provider lo ngại rằng các camera an ninh chụp hình liên tục ở độ phân giải cao sẽ tạo ra kích thước gói tin request khổng lồ (gây nghẽn băng thông mạng nội bộ LAN phòng Lab). Việc giải mã chuỗi Base64 liên tục ở tầng Application của server AI cũng sẽ gây quá tải CPU/RAM trước khi đưa ảnh vào GPU để suy luận.
- **Proposal**: Provider đề xuất loại bỏ hoàn toàn `imageBase64` ở điều kiện hoạt động thông thường. Nhóm A2 sẽ tải ảnh lên một hệ thống lưu trữ đệm Object Storage/CDN dùng chung của trường để lấy một đường dẫn Internet ngắn gọn (`imageUrl`) rồi mới truyền link này sang cho nhóm A4 bốc ảnh.
- **Resolution**: Accepted
- **Rationale**: Truyền link hình ảnh giúp giảm dung lượng request body xuống mức tối thiểu (chỉ vài trăm bytes), tối ưu hóa tốc độ truyền gửi gói tin và giúp server AI Vision nạp luồng dữ liệu vào GPU nhanh nhất.
- **Impact**:
  - **AI Vision**: Cấu hình trường `imageUrl` là bắt buộc (`required`), xóa bỏ trường `imageBase64` khỏi luồng xử lý chính trong file thiết kế OpenAPI.
  - **Camera Stream**: Tích hợp thêm module SDK Storage để thực hiện upload frame ảnh lấy link URL trước khi gọi API nhận diện.

---

## Issue #2: Kiểm soát tần suất gửi yêu cầu phân tích (Rate Limiting & Sampling Rate)

- **Raised by**: Provider (AI Vision)
- **Endpoint**: `POST /vision/detect`
- **Concern**: Phần cứng của hệ thống camera stream chạy ở tốc độ ghi hình tiêu chuẩn lên tới 24 - 30 khung hình trên giây (fps). Nếu nhóm A2 ép mỗi khung hình đều phải gọi API sang nhóm A4 phân tích, cụm máy chủ phần cứng GPU của server AI Vision chắc chắn sẽ rơi vào trạng thái quá tải nghiêm trọng, tràn hàng đợi và sập dịch vụ ngay lập tức.
- **Proposal**: Provider yêu cầu áp dụng kỹ thuật lấy mẫu định kỳ (Sampling Rate). Nhóm A2 sẽ khống chế tần suất đẩy request, chỉ trích xuất duy nhất 1 frame ảnh đại diện trong chu kỳ 1 giây (1 fps) từ luồng stream của camera để gửi sang dịch vụ AI.
- **Resolution**: Modified
- **Rationale**: Tần suất 1 giây/ảnh là hoàn toàn đáp ứng tốt các bài toán nghiệp vụ giám sát an ninh Smart Campus (phát hiện người xâm nhập trái phép hoặc phương tiện di chuyển sai quy định). Để tăng tính phòng thủ, hai bên thống nhất bổ sung thêm cơ chế giới hạn tần suất cứng (Rate Limiting) bằng thuật toán Token Bucket ở tầng cổng kết nối của Provider.
- **Impact**:
  - **Camera Stream**: Sử dụng các hàm lập lịch (cron-job hoặc setInterval) khống chế chu kỳ gửi request đúng khoảng cách giãn cách 1000ms.
  - **AI Vision**: Cấu hình Middleware giới hạn tối đa 1 request/giây cho mỗi mã định danh IP của camera. Nếu vượt ngưỡng, hệ thống tự động từ chối xử lý bằng mã lỗi `429 Too Many Requests`.

---

## Issue #3: Cơ chế bảo mật luồng dữ liệu bằng chuỗi khóa xác thực (Bearer Token)

- **Raised by**: Provider (AI Vision)
- **Endpoint**: Tất cả endpoints
- **Concern**: API xử lý nhận diện hình ảnh `/vision/detect` tiêu tốn rất nhiều tài nguyên hạ tầng đắt đỏ và xử lý thông tin hình ảnh an ninh nhạy cảm của nhà trường. Nếu không có cơ chế bảo vệ, bất kỳ thiết bị lạ nào kết nối vào mạng Wi-Fi nội bộ của trường cũng có thể gửi request phá hoại hoặc spam dữ liệu giả mạo gây nghẽn server AI.
- **Proposal**: Provider đề xuất tích hợp cơ chế bảo mật tiêu chuẩn, yêu cầu nhóm A2 bắt buộc phải đính kèm chuỗi khóa bảo mật định danh dạng mã hóa trong HTTP Header của mỗi request gửi sang.
- **Resolution**: Accepted
- **Rationale**: Đảm bảo an toàn thông tin theo các tiêu chuẩn thiết kế hệ thống API kết nối. Mọi request không có token hợp lệ sẽ bị chặn đứng ngay vòng kiểm duyệt ngoài để tránh làm phiền tới tầng xử lý deep learning phía sau.
- **Impact**:
  - **Cả hai bên**: Thống nhất khai báo cấu hình thuộc tính `securitySchemes` dạng `bearerAuth` (HTTP Bearer Token) với chuỗi khóa mặc định chạy local là `local-dev-token`.
  - **Camera Stream**: Khi thực hiện code hoặc cấu hình Postman, bắt buộc đính kèm Header: `Authorization: Bearer local-dev-token`.
  - **AI Vision**: Từ chối và phản hồi ngay mã lỗi số `401 Unauthorized` nếu request thiếu hoặc sai lệch token.

---

## Issue #4: Chuẩn hóa dữ liệu siêu định danh vị trí phần cứng (Camera Metadata)

- **Raised by**: Consumer (Camera Stream)
- **Endpoint**: `POST /vision/detect`
- **Concern**: Khi mô hình AI Vision nhận diện thành công một rủi ro (ví dụ: phát hiện kẻ gian leo rào), nếu dữ liệu trả về cho hệ thống nghiệp vụ (A6) chỉ thông báo là "có người" mà không biết bức ảnh đó được chụp từ chiếc camera cụ thể nào, đặt tại tòa nhà nào thì luồng xử lý an ninh phía sau sẽ bất lực, không thể ra quyết định điều động bảo vệ tới hiện trường.
- **Proposal**: Consumer yêu cầu bổ sung cấu trúc dữ liệu bắt buộc truyền mã định danh vật lý của camera (`camera_id`) đi kèm song song với liên kết hình ảnh gửi lên.
- **Resolution**: Accepted
- **Rationale**: Mã `camera_id` đóng vai trò là siêu dữ liệu (metadata) cốt lõi để liên kết không gian tọa độ vật lý trên bản đồ số Smart Campus, giúp chuỗi dữ liệu có giá trị sử dụng thực tế cho các phân hệ phía sau.
- **Impact**:
  - **Camera Stream**: Thiết lập cấu trúc JSON gửi lên bắt buộc phải chứa trường dữ liệu cụ thể, ví dụ: `"camera_id": "CAM_GATE_01"`.
  - **AI Vision**: Cập nhật mã nguồn tiếp nhận trường dữ liệu này và đính kèm nguyên vẹn nó vào cấu trúc kết quả đầu ra (Response Body) để chuyển tiếp thông tin an toàn cho các nhóm sau.

---

## Issue #5: Cơ chế kiểm tra trạng thái hoạt động trực tuyến (Health Check Endpoint)

- **Raised by**: Consumer (Camera Stream)
- **Endpoint**: Khởi tạo endpoint mới: `GET /health`
- **Concern**: Luồng dữ liệu camera stream hoạt động liên tục 24/7, nhóm A2 cần có một cơ chế tự động để nhận biết xem server ứng dụng AI Vision của nhóm A4 có đang còn sống (online) hay không. Tránh tình trạng server AI bị sập ngầm do quá nhiệt GPU nhưng hệ thống camera vẫn liên tục upload ảnh lên CDN và bắn request vô ích, gây lãng phí tài nguyên lưu trữ đám mây.
- **Proposal**: Consumer đề xuất Provider cung cấp một API kiểm tra sức khỏe hệ thống ở dạng siêu nhẹ (Lightweight Endpoint) để chạy vòng lặp kiểm tra định kỳ ngầm.
- **Resolution**: Accepted
- **Rationale**: Giúp các bên giám sát tính sẵn sàng cao (High Availability) của dịch vụ theo thời gian thực một cách trực quan và tốn cực kỳ ít chi phí phần cứng (không chạy qua model AI nặng).
- **Impact**:
  - **AI Vision**: Phát triển riêng biệt endpoint `GET /health`. Khi được gọi, API chỉ phản hồi ngay mã trạng thái `200 OK` kèm JSON ngắn gọn: `{"status": "up", "timestamp": "2026-05-28T..."}`.
  - **Camera Stream**: Thiết lập luồng Worker ngầm gọi định kỳ vào endpoint GET này mỗi 5 phút một lần để tự động kiểm soát trạng thái kết nối.

---

## Issue #6: Thống nhất cấu trúc thông báo lỗi dữ liệu đầu vào (Validation Error Schema)

- **Raised by**: Provider (AI Vision)
- **Endpoint**: `POST /vision/detect`
- **Concern**: Khi nhóm A2 truyền sai định dạng dữ liệu (ví dụ: truyền mã camera trống hoặc định dạng link ảnh không hợp lệ), Provider cần trả về một cấu trúc lỗi rõ ràng để hệ thống phần mềm của nhóm A2 tự động phân tích lý do thất bại, tránh việc crash ứng dụng camera.
- **Proposal**: Provider đề xuất thiết kế cấu trúc Schema lỗi chi tiết áp dụng cho mã trạng thái lỗi dữ liệu `400 Bad Request`.
- **Resolution**: Accepted
- **Rationale**: Đảm bảo tính nhất quán trong xử lý lỗi ngoại lệ, giúp quá trình tích hợp và gỡ lỗi (debugging) giữa hệ thống của hai nhóm diễn ra nhanh chóng.
- **Impact**:
  - **AI Vision**: Cấu hình phản hồi lỗi `400` trả về định dạng JSON chứa mảng `errors` mô tả chính xác trường dữ liệu nào bị nhập sai cấu trúc hợp đồng.
  - **Camera Stream**: Viết bộ lọc logic bắt lỗi (Try-Catch block) để xử lý log khi nhận được mã lỗi `400` từ hệ thống AI Vision.

---

# Chốt hợp đồng v1.0

- **Provider sign-off**: Nguyễn Minh Mạnh (Đại diện nhóm A4 AI Vision)  
- **Consumer sign-off**: [Bùi Đình Phúc nhóm A2]  
- **Witness (GV/TA)**: Lê Thái Bảo  
- **Date**: 28-05-2026  

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| Không có warning nào | Hợp đồng API đã sạch hoàn toàn lỗi và cảnh báo sau khi tiến hành kiểm thử qua bộ công cụ Spectral Linter. | Không cần sửa đổi bổ sung. |
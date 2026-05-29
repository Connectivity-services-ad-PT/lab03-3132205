# Nhật ký Thương lượng Hợp đồng API (Negotiation Log) — Nhóm A4 (AI Vision)

## 1. Thông tin chung
- **Môn học:** Phát triển dịch vụ kết nối (FIT4110) — Bài thực hành số 03 (Lab 03)
- **Hệ thống:** Smart Campus — Phân hệ Product A
- **Bên cung cấp (Provider):** Nhóm A4 — AI Vision API Service
- **Bên tiêu thụ (Consumers):** - Nhóm A2 (Camera Stream — Phân hệ đẩy luồng ảnh vào phân tích)
  - Nhóm A5 (Analytics — Phân hệ nhận kết quả để thống kê, báo cáo)
  - Nhóm A6 (Core Business — Phân hệ xử lý nghiệp vụ trung tâm và kích hoạt cảnh báo)

---

## 2. Nhật ký chi tiết các phiên thương lượng hợp đồng

### 📌 Phiên họp số 1: Thống nhất giao diện đầu vào (Request Body) với Nhóm A2
- **Thời gian:** 28/05/2026
- **Hình thức:** Thảo luận trực tiếp tại phòng Lab và thống nhất qua kênh chat.
- **Vấn đề đặt ra:** File OpenAPI thiết kế mẫu ban đầu của nhà trường cung cấp hai phương thức truyền ảnh: sử dụng `image_url` (đường dẫn trực tuyến) hoặc `image_base64` (chuỗi mã hóa base64 cấu trúc nặng). Nhóm A2 cần chốt phương án tối ưu để lập trình module Camera Stream.
- **Nội dung thương lượng:**
  1. Nhóm A2 đề xuất chỉ sử dụng phương thức truyền link ảnh trực tuyến (`image_url`) để giảm tải băng thông mạng nội bộ, tránh việc mã hóa/giải mã chuỗi Base64 dung lượng lớn làm chậm tốc độ phản hồi real-time của AI.
  2. Nhóm A4 đồng ý và yêu cầu nhóm A2 phải cung cấp kèm theo mã định danh của camera (`camera_id`) để phục vụ cho việc tracking vị trí vật thể của các nhóm phía sau (A5, A6).
  3. Nhóm A2 đã cung cấp cấu trúc JSON mẫu chuẩn để nhóm A4 thực hiện cấu hình Mock Server:
     ```json
     {
       "camera_id": "CAM01",
       "image_url": "[https://example.com/frame.jpg](https://example.com/frame.jpg)"
     }
     ```
- **Kết quả thay đổi trong file thiết kế (`ai-vision.openapi.yaml`):**
  - Chỉnh sửa schema `DetectionRequest`: Đưa cả `camera_id` và `image_url` vào danh sách các thuộc tính bắt buộc (`required: [camera_id, image_url]`).
  - Loại bỏ hoàn toàn thuộc tính không sử dụng: `image_base64`.
  - Cập nhật thông báo lỗi `400 Bad Request` chi tiết khi thiếu một trong hai trường trên: `"detail: camera_id and image_url are required fields"`.

---

### 📌 Phiên họp số 2: Thống nhất giao diện đầu ra (Response Body) với Nhóm A5 & Nhóm A6
- **Thời gian:** 29/05/2026
- **Hình thức:** Thảo luận nhóm tập trung thông qua việc chia sẻ file thiết kế.
- **Vấn đề đặt ra:** Thống nhất các trường thông tin trong kết quả nhận diện đối tượng (`POST /detect`) để nhóm A5 có thể viết logic đếm số lượng (người/xe) và nhóm A6 có thể kích hoạt các kịch bản nghiệp vụ an ninh bảo mật tương ứng.
- **Nội dung thương lượng:**
  1. **Nhóm A5 (Analytics)** yêu cầu kết quả trả về bắt buộc phải có thuộc tính phân loại đối tượng (`label`) rõ ràng để thực hiện tổng hợp dữ liệu mật độ giao thông/sinh viên theo thời gian.
  2. **Nhóm A6 (Core Business)** yêu cầu bổ sung mức độ rủi ro (`risk_level`) được phân loại sẵn bởi phân hệ AI nhằm giảm tải logic tính toán cho hệ thống cốt lõi khi xử lý các tình huống khẩn cấp.
  3. Nhóm A4 đề xuất giới hạn tập dữ liệu hợp lệ (enum) cho các trường để tránh lỗi đồng bộ kiểu chuỗi giữa các ngôn ngữ lập trình khác nhau của các nhóm.
- **Kết quả thay đổi trong file thiết kế (`ai-vision.openapi.yaml`):**
  - Chốt cấu trúc thành công cho `DetectionResponse` (Response `200 OK`) bao gồm: `detection_id`, `camera_id`, `label`, `confidence`, `risk_level`.
  - Ràng buộc dữ liệu trường `label` cố định trong tập: `[person, vehicle, unknown]`.
  - Ràng buộc dữ liệu trường `risk_level` cố định trong tập: `[low, medium, high]`.
  - Cấu hình chuẩn hóa tài liệu hiển thị ví dụ mẫu (examples) trực quan khớp với kịch bản phát hiện đối tượng là con người (`label: person`, `risk_level: medium`, `confidence: 0.91`).

---

## 3. Trạng thái hợp đồng và Kết quả kiểm thử (Smoke Test)

- **Phiên bản hợp đồng hiện tại:** `0.3.0` (Đã đóng băng giao diện thiết kế - Frozen).
- **Môi trường giả lập nội bộ:** Đã triển khai chạy thử nghiệm Mock Server thành công bằng công cụ Prism CLI trên cổng `4011` qua địa chỉ mạng nội bộ LAN: `http://172.20.240.1:4011`.
- **Kết quả kiểm thử tự động (Postman Smoke Test):**
  - Thực hiện gửi Request `POST /detect` kèm theo Body JSON từ nhóm A2 và Bearer Token mẫu thành công.
  - Kết quả trả về đạt trạng thái **`200 OK`**.
  - Đạt tiêu chuẩn **3/3 Tests Passed** trên Postman kiểm tra cấu trúc phản hồi.
  - Hệ thống Mock Server ghi nhận Log xử lý hợp lệ (`[VALIDATOR] ✓ success`, `[NEGOTIATOR] ✓ success`).

---

## 4. Kế hoạch tích hợp và Kết luận
- File hợp đồng API và Nhật ký thương lượng này đã được upload công khai lên Repository GitHub của nhóm A4 để các nhóm tiêu thụ (Consumers) chủ động kéo (pull) về lập trình song song.
- **Cam kết:** Nhóm A4 sẽ giữ nguyên cấu trúc phân tách các trường dữ liệu này trong suốt quá trình phát triển dự án thật tiếp theo, đảm bảo không phá vỡ kiến trúc kết nối đã thống nhất giữa 4 nhóm (A2 - A4 - A5 - A6).

**Đại diện ký duyệt hợp đồng API nhóm A4:**
*Thành viên phụ trách thiết kế hệ thống*
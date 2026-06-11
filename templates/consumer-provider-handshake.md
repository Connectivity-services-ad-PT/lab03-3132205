# Consumer-Provider API Contract Handshake

## 1. Thành phần tham gia
* **Provider (Bên cung cấp):** `team-vision` (AI Vision Service)
* **Consumer (Bên tiêu thụ):** `team-camera` / `team-iot` (Hệ thống gọi nhận diện)

## 2. Endpoint Thỏa Thuận Bàn Giao
* **API:** `POST /api/v1/vision/detect`
* **Mục đích:** Nhận diện thực thể, khuôn mặt từ luồng Camera gửi về thời gian thực.

## 3. Cam kết kiểm thử (Mock Agreement)
1. Bên **Provider** cam kết cung cấp Mock Server chạy ổn định tại cổng port quy định trên môi trường CI để phục vụ Consumer Smoke Test.
2. Cấu trúc dữ liệu Payload (`cameraId`, `imageRaw`) và dữ liệu trả về (`status`, `detectionId`) đã được đồng bộ hóa, kiểm tra qua linter đạt trạng thái tuyệt đối không có cảnh báo đỏ.
3. Toàn bộ các ca kiểm thử tích hợp (Contract Tests) đã chạy thành công thông qua Newman CLI.
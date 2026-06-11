# Consumer-Provider API Contract Handshake

## 1. Thành phần tham gia
* **Provider (Bên cung cấp):** `team-vision` (AI Vision Service)
* **Consumer (Bên tiêu thụ):** `team-camera` / `team-iot`

## 2. Endpoint Thỏa Thuận Bàn Giao
* **API:** `POST /api/v1/vision/detect`
* **Mục đích:** Nhận diện thực thể, khuôn mặt từ luồng dữ liệu Camera gửi về thời gian thực.

## 3. Cam kết kiểm thử (Mock Agreement)
1. Bên **Provider** cam kết cấu hình Mock Server chạy ổn định trên môi trường CI để phục vụ Consumer Smoke Test.
2. Toàn bộ cấu trúc thực thể đã được dọn sạch lỗi linter, đảm bảo tiến trình tích hợp không bị gián đoạn.
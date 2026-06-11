# KIỂM THỬ BÊN CONSUMER — Consumer-side Testing

## 1. Mục đích

Consumer-side testing giúp nhóm gọi API:

- xác nhận mock provider hoạt động đúng với contract,
- phát hiện sớm vấn đề trước khi provider hoàn thành code thật,
- giảm rủi ro khi tích hợp với hệ thống khác.

Trong hệ thống Smart Campus, các luồng thường phụ thuộc lẫn nhau:

- Camera Stream → AI Vision → Core Business → Notification
- IoT Ingestion → Core Business
- IoT Ingestion → Analytics
- Access Gate → Core Business

Nếu consumer phải chờ provider hoàn thiện, tiến độ toàn bộ chuỗi bị đình trệ. Vì vậy, consumer cần có test gọi mock API ngay từ đầu.

---

## 2. Quy trình handshake

### Bước 1 — Provider chia sẻ contract

Provider phải cung cấp cho consumer:

- file `openapi.yaml`
- `mock_base_url`
- quy tắc xác thực (`auth rule`)
- ví dụ request
- ví dụ response

### Bước 2 — Consumer xây dựng smoke test

Consumer cần tạo ít nhất một smoke test gọi đến mock API của provider.

Ví dụ Camera gọi AI Vision mock:

```http
POST {{aiVisionMockUrl}}/detect
Authorization: Bearer {{authToken}}
Content-Type: application/json

{
  "camera_id": "CAM01",
  "image_url": "https://example.com/frame.jpg"
}
```

Kết quả kỳ vọng phải trả về response hợp lệ và có thể dùng được:

```json
{
  "detection_id": "DET001",
  "label": "person",
  "confidence": 0.91,
  "risk_level": "medium"
}
```

### Bước 3 — Ghi biên bản handshake

Ghi lại các thông tin đã thỏa thuận bằng template:

- `templates/consumer-provider-handshake.md`

Nội dung biên bản nên bao gồm:

- đường dẫn mock API
- định nghĩa endpoint
- yêu cầu xác thực
- ví dụ request/response
- các tiêu chí test

---

## 3. Tiêu chí đánh giá

Consumer-side smoke test được coi là pass khi:

- Gọi đúng endpoint và base URL của provider.
- Request body đúng schema theo contract.
- Response trả về chứa các field consumer cần dùng.
- Consumer có thể xử lý ít nhất một trường hợp lỗi 4xx hoặc 5xx.
- Kèm bằng chứng thực thi: ảnh chụp màn hình, log test hoặc Newman report.

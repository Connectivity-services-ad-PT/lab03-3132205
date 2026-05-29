# BIÊN BẢN ĐÀM PHÁN HỢP ĐỒNG API (NEGOTIATION LOG)

- **Môn học:** Phát triển dịch vụ kết nối (FIT4110) — Bài thực hành số 03 (Lab 03)
- **Hệ thống:** Smart Campus Platform
- **Phân hệ thuộc dự án:** Product A
- **Cặp đàm phán:** Pair 02 — Phân hệ Core Business (A6/B6) ↔ Phân hệ AI Vision (A4/B4)
- **Bên cung cấp (Provider):** Nhóm A4 — AI Vision API Service  
  - *Đại diện kỹ thuật:* Nguyễn Minh Mạnh
- **Bên tiêu thụ (Consumer):** Nhóm A6 — Core Business System  
  - *Đại diện kỹ thuật:* Nguyễn Thị Hồng Duyên
- **Phiên bản hợp đồng chốt:** v1.0 (Trạng thái: Frozen — Đóng băng thiết kế)
- **Ngày đàm phán kết thúc:** 28-05-2026
- **Người giám sát / Nhân chứng (GV/TA):** Lê Thái Bảo

---

## MỤC LỤC
1. Issue #1: Định dạng dữ liệu hình ảnh đầu vào (imageUrl vs imageBase64)
2. Issue #2: Cơ chế xử lý Đồng bộ (Synchronous) vs Bất đồng bộ (Asynchronous)
3. Issue #3: Bổ sung tọa độ đối tượng nhận dạng (Bounding Box)
4. Issue #4: Chuẩn hóa định dạng phản hồi lỗi (Problem Details - RFC 7807)
5. Issue #5: Chống trùng lặp yêu cầu xử lý tài nguyên (Idempotency)
6. Issue #6: Đồng bộ hóa nhãn nhận dạng (Labels) và phiên bản Model AI
7. Trạng thái kiểm thử Mock Server & Cam kết triển khai

---

## CHI TIẾT CÁC VẤN ĐỀ ĐÀM PHÁN (ISSUES LOG)

### 📌 Issue #1: Định dạng dữ liệu hình ảnh đầu vào (imageUrl vs imageBase64)

- **Bên đặt vấn đề (Raised by):** Provider (AI Vision — Nhóm A4)
- **Endpoint bị tác động:** `POST /vision/detect`
- **Nội dung lo ngại (Concern):** Ban đầu, hợp đồng API dự kiến cho phép truyền trực tiếp chuỗi mã hóa `imageBase64` trong request body. Tuy nhiên, Provider lo ngại rằng khi hệ thống Camera Stream gửi lên các frame ảnh chất lượng cao, kích thước request body của chuỗi Base64 sẽ phình to (tăng khoảng 33% dung lượng gốc, dễ dàng vượt ngưỡng 10MB+ cho mỗi request). Điều này gây nghẽn băng thông mạng nội bộ nghiêm trọng và làm quá tải bộ nhớ đệm (Buffer Memory) của server AI Vision khi phải xử lý giải mã đồng thời nhiều luồng stream. Do đó, Provider đề xuất chỉ hỗ trợ `imageUrl` (ảnh đã được upload lên hệ thống CDN/Object Storage dùng chung của Smart Campus).
- **Giải pháp đề xuất từ Consumer (Proposal):** Consumer giải trình một kịch bản nghiệp vụ khẩn cấp (Edge-case): Trong các tình huống thiên tai, sự cố kết nối ngoại vi hoặc khi CDN gặp sự cố sập trục trặc mạng, các camera giám sát tại cổng an ninh chỉ có thể lưu ảnh tạm thời tại bộ nhớ cục bộ (local frame buffer) và bắt buộc phải đẩy trực tiếp dữ liệu thô dạng Base64 lên Core Business để xử lý khẩn cấp. Do đó, Consumer đề xuất API bắt buộc phải hỗ trợ linh hoạt cả 2 cấu trúc dữ liệu: `imageUrl` (luồng ưu tiên mặc định) và `imageBase64` (luồng dự phòng thảm họa).
- **Quyết định cuối cùng (Resolution):** Accepted with Modification (Chấp nhận có chỉnh sửa).
- **Lý do kỹ thuật (Rationale):** Hai bên thống nhất sử dụng cấu trúc đa hình **`oneOf`** kết hợp với từ khóa hệ thống **`discriminator`** thông qua một thuộc tính phân loại cụ thể mang tên `imageType` (với hai giá trị enum cố định là `URL` và `BASE64`) bên trong schema `VisionDetectRequest`. Đồng thời, để bảo vệ server AI khỏi các cuộc tấn công từ chối dịch vụ (DoS/Out of memory), hai bên thống nhất thiết lập cấu hình chặn cứng (Max Body Size Limit) ở tầng Web Server/Middleware của AI Vision là tối đa 10MB.
- **Tác động mã nguồn (Impact):**
  - **AI Vision (A4):** Viết thêm Middleware kiểm tra `Content-Length` và kích thước string base64 đầu vào, cấu hình chặn >10MB. 
  - **Core Business (A6):** Thiết kế mã nguồn module tích hợp Camera: Luôn ưu tiên gọi API upload CDN trước để lấy `imageUrl`, chỉ kích hoạt luồng encode Base64 khi cấu phần CDN phản hồi lỗi gateway.

---

### 📌 Issue #2: Cơ chế xử lý Đồng bộ (Synchronous) vs Bất đồng bộ (Asynchronous)

- **Bên đặt vấn đề (Raised by):** Consumer (Core Business — Nhóm A6)
- **Endpoint bị tác động:** `POST /vision/detect`
- **Nội dung lo ngại (Concern):** Consumer yêu cầu API xử lý và trả về kết quả nhận diện ngay lập tức (Xử lý đồng bộ - Synchronous) để hệ thống cốt lõi kịp thời đưa ra quyết định tự động: đóng/mở cổng barrier kiểm soát xe, kích hoạt còi hú rào chắn hoặc gửi thông báo đẩy (push notification) thời gian thực đến điện thoại của lực lượng bảo vệ. Ngược lại, Provider làm rõ giới hạn hạ tầng: Việc phân tích hình ảnh bằng các mô hình học sâu (Deep Learning) như YOLOv8x đòi hỏi tài nguyên tính toán lớn. Thời gian inference (suy luận AI) dao động từ 200ms đến hơn 2.000ms tùy vào số lượng đối tượng trong ảnh và tải trọng GPU tại thời điểm đó. Nếu bắt buộc xử lý đồng bộ trong khung giờ cao điểm (sinh viên đến trường đông) sẽ gây nghẽn hàng đợi kết nối, dẫn đến lỗi Gateway Timeout (`504`) hàng loạt.
- **Giải pháp đề xuất từ Consumer (Proposal):** Consumer đề xuất thiết kế một kiến trúc API lai (Hybrid Mechanism) hỗ trợ linh hoạt dựa trên trạng thái chịu tải thực tế của server AI, cung cấp đồng thời hai mã phản hồi HTTP trạng thái:
  1. Trả về mã **`201 Created`** kèm theo kết quả nhận dạng chi tiết ngay trong Response Body (Áp dụng khi hàng đợi GPU đang trống - luồng Đồng bộ).
  2. Trả về mã **`202 Accepted`** kèm theo một HTTP Header **`Location`** chứa URI trỏ đến đường dẫn tra cứu kết quả độc lập (Áp dụng khi GPU đang quá tải hoặc đang bận xử lý tác vụ phân tích nặng - luồng Bất đồng bộ).
- **Quyết định cuối cùng (Resolution):** Accepted (Chấp nhận hoàn toàn).
- **Lý do kỹ thuật (Rationale):** Giải pháp kiến trúc lai này đảm bảo tính co giãn (Scalability) tối đa cho toàn hệ thống. Nó vừa giúp Core Business nhận được kết quả xử lý real-time tức thì trong điều kiện vận hành bình thường, vừa bảo vệ hệ thống không bị đổ vỡ, sập nghẽn khi có lượng yêu cầu đột biến từ hàng trăm camera gửi về cùng lúc.
- **Tác động mã nguồn (Impact):**
  - **AI Vision (A4):** Triển khai cấu trúc hàng đợi ngầm (Background Worker Queue như BullMQ hoặc Celery/Redis) để xử lý các request bất đồng bộ, đồng thời phát triển thêm một endpoint phụ trách tra cứu trạng thái: `GET /vision/detect/{requestId}`.
  - **Core Business (A6):** Phát triển bộ logic Rule Engine thông minh: Khi nhận về HTTP Code `201`, xử lý nghiệp vụ ngay; nếu nhận về HTTP Code `202`, tự động chuyển đổi sang cơ chế lập lịch Polling (truy vấn định kỳ mỗi 500ms) gửi tới endpoint GET của nhóm A4 để lấy kết quả khi sẵn sàng.

---

### 📌 Issue #3: Bổ sung tọa độ đối tượng nhận dạng (Bounding Box)

- **Bên đặt vấn đề (Raised by):** Consumer (Core Business — Nhóm A6)
- **Endpoint bị tác động:** `POST /vision/detect` & `GET /vision/detect/{requestId}`
- **Nội dung lo ngại (Concern):** Bản thiết kế API ban đầu do nhóm A4 cung cấp rất sơ sài, chỉ trả ra một mảng các đối tượng nhận diện gồm nhãn phân loại (`type`) và độ tin cậy (`confidence`). Tuy nhiên, Consumer chỉ ra rằng thông tin này là chưa đủ đối với phân hệ Dashboard Giám sát của Quản trị viên (Admin UI). Để làm bằng chứng an ninh trực quan cho các nghiệp vụ như phát hiện xâm nhập vùng cấm hoặc nhận diện kẻ gian, hệ thống bắt buộc phải vẽ lại một khung viền hình học bao quanh vật thể thực tế trên màn hình giám sát.
- **Giải pháp đề xuất từ Consumer (Proposal):** Consumer yêu cầu Provider can thiệp vào cấu trúc dữ liệu đầu ra, bổ sung thêm một object mô tả chi tiết hệ tọa độ hình học của khung giới hạn đối tượng (`boundingBox`) cho từng thực thể được phát hiện.
- **Quyết định cuối cùng (Resolution):** Accepted (Chấp nhận hoàn toàn).
- **Lý do kỹ thuật (Rationale):** Khung giới hạn (Bounding Box) là dữ liệu tiêu chuẩn và thiết yếu của mọi dịch vụ thị giác máy tính (Computer Vision API). Việc cung cấp tọa độ giúp nâng cao trải nghiệm người dùng trên UI Admin và hỗ trợ lưu trữ siêu dữ liệu (metadata) chính xác phục vụ công tác điều tra hậu kiểm an ninh.
- **Tác động mã nguồn (Impact):**
  - **AI Vision (A4):** Chỉnh sửa tầng hậu xử lý (Post-processing Layer) của mã nguồn AI để bốc tách 4 tham số hình học pixel từ output đồ thị của mô hình bao gồm: tọa độ điểm neo đầu (`x`, `y`), chiều rộng (`width`), và chiều cao (`height`) của khung. Sau đó, định nghĩa một Schema mới mang tên `BoundingBox` trong file OpenAPI.
  - **Core Business (A6):** Chỉnh sửa cấu trúc bảng trong Cơ sở dữ liệu (Processing Database) để lưu trữ thêm 4 trường tọa độ hình học này đi kèm mỗi bản ghi nhận diện đối tượng.

---

### 📌 Issue #4: Chuẩn hóa định dạng phản hồi lỗi (Problem Details - RFC 7807)

- **Bên đặt vấn đề (Raised by):** Consumer (Core Business — Nhóm A6)
- **Endpoint bị tác động:** Toàn bộ các Endpoints trong hệ thống.
- **Nội dung lo ngại (Concern):** Consumer nhận thấy cấu trúc phản hồi lỗi của hệ thống ban đầu không có quy chuẩn đồng nhất. Khi xảy ra lỗi xác thực dữ liệu (Validation Error) thì trả về chuỗi kí tự plain text, khi lỗi hệ thống nội bộ (Internal Server Error) lại trả về cấu trúc JSON thiếu trường. Điều này khiến việc viết mã nguồn bắt ngoại lệ (Exception Handling) tự động ở phía Backend của Core Business trở nên cực kỳ phức tạp và dễ phát sinh lỗi crash hệ thống do không đồng bộ cấu trúc dữ liệu.
- **Giải pháp đề xuất từ Consumer (Proposal):** Consumer yêu cầu hai bên áp dụng triệt để tiêu chuẩn quốc tế **Problem Details for HTTP APIs (RFC 7807)** cho toàn bộ các phản hồi mã lỗi lớp `4xx` (Client Errors) và `5xx` (Server Errors), sử dụng định dạng header bắt buộc là `Content-Type: application/problem+json`.
- **Quyết định cuối cùng (Resolution):** Accepted (Chấp nhận hoàn toàn).
- **Lý do kỹ thuật (Rationale):** Việc chuẩn hóa theo tiêu chuẩn RFC 7807 giúp toàn bộ kiến trúc Smart Campus Platform trở nên nhất quán, chuyên nghiệp, hỗ trợ các lập trình viên dễ dàng gỡ lỗi (debug) nhanh chóng nhờ cấu trúc thông điệp giàu ngữ cảnh thông tin.
- **Tác động mã nguồn (Impact):**
  - **Cả hai bên (A4 & A6):** Đồng thuận thiết kế và áp dụng một Schema chung mang tên `Problem` trong file thiết kế hợp đồng. Cấu trúc Schema này bắt buộc phải bao gồm các thuộc tính định danh: `type` (URI liên kết đến tài liệu mô tả lỗi), `title` (Tiêu đề ngắn gọn của lỗi), `status` (Mã trạng thái HTTP), `detail` (Mô tả chi tiết nguyên nhân lỗi xảy ra trong ngữ cảnh), `instance` (URI chính xác của endpoint xảy ra lỗi), và một mảng tùy chọn `errors` chứa danh sách chi tiết các trường dữ liệu bị vi phạm định dạng.

---

### 📌 Issue #5: Chống trùng lặp yêu cầu xử lý tài nguyên (Idempotency)

- **Bên đặt vấn đề (Raised by):** Provider (AI Vision — Nhóm A4)
- **Endpoint bị tác động:** `POST /vision/detect`
- **Nội dung lo ngại (Concern):** Trong môi trường mạng nội bộ của trường học, hiện tượng chập chập chờn tín hiệu mạng Wi-Fi/LAN hoặc độ trễ phản hồi hệ thống là hoàn toàn có thể xảy ra. Khi Core Business gửi đi một request phân tích ảnh nhưng không nhận được phản hồi kịp thời do timeout giả lập, cơ chế retry tự động của hệ thống cốt lõi sẽ kích hoạt bắn lại đúng request đó lần thứ 2. Nếu không có cơ chế chống trùng lặp, server AI Vision sẽ vô tình phải chạy lại toàn bộ mô hình Deep Learning từ đầu trên cùng một khung hình ảnh cũ, gây lãng phí tài nguyên GPU cực kỳ lớn và làm giảm hiệu năng chung của toàn hệ thống.
- **Giải pháp đề xuất từ Provider (Proposal):** Provider đề xuất đưa ra quy định bắt buộc: Trong mọi request body gửi lên từ Consumer, Core Business phải đính kèm một chuỗi định danh duy nhất mang tên `requestId` (Sử dụng định dạng chuẩn UUID v4). AI Vision sẽ sử dụng khóa này để thực hiện cơ chế lọc trùng và cache kết quả.
- **Quyết định cuối cùng (Resolution):** Accepted (Chấp nhận hoàn toàn).
- **Lý do kỹ thuật (Rationale):** Đảm bảo tính Idempotent cho phương thức `POST` xử lý tài nguyên nặng. Giúp bảo vệ tối đa tài nguyên tính toán phần cứng đắt đỏ (GPU vRAM) không bị lãng phí bởi các yêu cầu retry trùng lặp từ phía Consumer.
- **Tác động mã nguồn (Impact):**
  - **Core Business (A6):** Chỉnh sửa logic mã nguồn, đảm bảo mỗi khi chuẩn bị phát động một yêu cầu phân tích frame hình ảnh mới, hệ thống phải sinh ngẫu nhiên một chuỗi UUID v4 không trùng lặp và gán vào trường `requestId`.
  - **AI Vision (A4):** Triển khai tầng lưu trữ đệm phân tán (Distributed In-Memory Cache như Redis) để lưu giữ các `requestId` đang hoặc đã xử lý. Nếu một `requestId` trùng lặp gửi tới khi request trước đang chạy, hệ thống lập tức trả về mã lỗi `409 Conflict`. Nếu request trước đã hoàn thành, hệ thống bốc ngay dữ liệu từ Database trả về tức thì mà không cần chạy lại mô hình AI.

---

### 📌 Issue #6: Đồng bộ hóa nhãn nhận dạng (Labels) và phiên bản Model AI

- **Bên đặt vấn đề (Raised by):** Consumer (Core Business — Nhóm A6)
- **Endpoint bị tác động:** Khởi tạo endpoint mới hoàn toàn: `GET /vision/models`
- **Nội dung lo ngại (Concern):** Phân hệ AI Vision liên tục tiến hành nâng cấp, tinh chỉnh và huấn luyện lại các mô hình nhận dạng đối tượng theo thời gian (ví dụ: cập nhật từ kiến trúc YOLOv8 lên YOLOv9, hoặc chuyển đổi từ nhận diện `person` chung chung sang phân loại chi tiết `student`, `staff`). Việc nâng cấp này dẫn đến sự thay đổi hoặc mở rộng của tập hợp các nhãn nhận diện đầu ra (ví dụ xuất hiện thêm nhãn cảnh báo nguy hiểm mới như `WEAPON` hoặc `FIRE`). Nếu Core Business không nhận biết được sự thay đổi này một cách tự động, bộ máy xử lý quy tắc nghiệp vụ (Rule Engine) của hệ thống cốt lõi sẽ bỏ qua các cảnh báo nguy hiểm hoặc biên dịch sai lệch dữ liệu.
- **Giải pháp đề xuất từ Consumer (Proposal):** Consumer đề xuất Provider phải cung cấp một endpoint chuyên biệt để trả về danh sách toàn bộ các mô hình AI đang kích hoạt cùng tập hợp các nhãn tương thích của chúng. Đồng thời, trong mọi kết quả phản hồi phân tích ảnh thành công, bắt buộc phải đính kèm trường thông tin chỉ rõ phiên bản model đã xử lý.
- **Quyết định cuối cùng (Resolution):** Accepted (Chấp nhận hoàn toàn).
- **Lý do kỹ thuật (Rationale):** Giúp hệ thống Core Business tự động kiểm soát cấu hình tính tương thích của dữ liệu (Schema Compatibility) và cập nhật động các quy tắc của Rule Engine một cách lập trình, loại bỏ hoàn toàn việc phải sửa đổi mã nguồn thủ công mỗi khi đội ngũ AI cập nhật phiên bản model mới.
- **Tác động mã nguồn (Impact):**
  - **AI Vision (A4):** Phát triển endpoint mới `GET /vision/models` trả về danh sách chi tiết các model kèm mảng `supportedLabels`. Đồng thời thêm thuộc tính chuỗi `modelVersion` vào cấu trúc kết quả của schema `VisionDetectResult`.
  - **Core Business (A6):** Viết module tự động đồng bộ hóa dữ liệu (Sync Worker) định kỳ truy vấn endpoint `/vision/models` để cập nhật danh mục cấu hình trên giao diện Admin Dashboard và ghi nhận trường `modelVersion` vào Nhật ký kiểm toán hệ thống (Audit Log).

---

## 3. CHỐT HỢP ĐỒNG VÀ TRẠNG THÁI KIỂM THỬ (SIGN-OFF)

- **Đánh giá chất lượng thiết kế (Linter Tool):** Hợp đồng API được viết hoàn chỉnh trong file `ai-vision.openapi.yaml` đã được chạy kiểm tra qua bộ công cụ thiết kế **Spectral Linter**. Kết quả đạt trạng thái tuyệt đối: **0 Errors (Không có lỗi thiết kế), 0 Warnings (Không có cảnh báo vi phạm chuẩn thiết kế OpenAPI v3)**. Giao diện hợp đồng chính thức đóng băng (Frozen).
- **Trạng thái kiểm thử môi trường giả lập (Smoke Test):** Nhóm A4 đã tiến hành khởi chạy thành công Mock Server mạng nội bộ LAN bằng công cụ chuyên dụng **Prism CLI** (`npx prism mock ai-vision.openapi.yaml -p 4011 -h 0.0.0.0`) lắng nghe tại địa chỉ IP của máy Provider: `http://172.20.240.1:4011`.
- **Kết quả kiểm định tự động Postman:** Toàn bộ kịch bản kiểm thử tích hợp tự động (Automation Test Script) cấu hình trên Postman bao gồm việc kiểm tra tính hợp lệ của Token bảo mật Bearer, định dạng kiểu dữ liệu Response Body JSON trả về đều đạt chỉ số hoàn hảo **3/3 Tests Passed** (Tất cả các điều kiện kiểm thử đều vượt qua). Hệ thống Mock Server Prism ghi nhận log validator xử lý mượt mà (`[VALIDATOR] ✓ success`).

### KÝ DUYỆT ĐIỆN TỬ BAN ĐẠM PHÁN (SIGN-OFF)

| Vai trò | Đại diện phân hệ | Chữ ký điện tử (Sign-off) | Trạng thái phê duyệt |
|---|---|---|---|
| **Provider** | Nhóm A4 — AI Vision API Service | **Nguyễn Minh Mạnh** | Đã ký duyệt (Approved) |
| **Consumer** | Nhóm A6 — Core Business System | **Nguyễn Thị Hồng Duyên** | Đã ký duyệt (Approved) |
| **Nhân chứng** | Giảng viên / Trợ giảng giám sát môn học | **Lê Thái Bảo** | Đã nghiệm thu (Verified) |

---
*Mọi thay đổi phát sinh sau ngày 28-05-2026 đối với cấu trúc hợp đồng v1.0 này bắt buộc phải lập một phiên bản phụ lục đàm phán mới và phải có sự phê duyệt trực tiếp từ Giảng viên giám sát Lê Thái Bảo.*
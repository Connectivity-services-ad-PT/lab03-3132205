# Reliability and Contract Test Checklist — Team Vision

## 1. Functional tests
- [x] **Có test cho endpoint health:** Đã cấu hình kiểm tra trạng thái hoạt động của hệ thống thông qua endpoint `/health`.
- [x] **Có test happy path cho endpoint chính:** Đã viết kịch bản kiểm thử luồng chạy thành công (Happy Path) cho endpoint lấy dữ liệu nhận diện diện v1/v2 và endpoint thực thi nhận diện real-time.
- [x] **Có kiểm tra status code 2xx:** Tất cả các ca kiểm thử functional thành công đều bắt buộc assert `pm.response.to.have.status(200)` hoặc `201`.
- [x] **Có kiểm tra field quan trọng trong response:** Đã kiểm tra sự tồn tại của các trường dữ liệu cốt lõi như `data`, `hasMore`, `nextCursor` trong API lấy danh sách và `detectionId`, `status` trong API xử lý ảnh.
- [x] **Có ít nhất 1 test đọc dữ liệu danh sách hoặc chi tiết:** Đã cấu hình ca kiểm thử đọc danh sách lịch sử nhận diện (`GET /api/v1/vision/detections`) hỗ trợ phân trang Cursor-based.

## 2. Auth tests
- [x] **Có test thiếu token:** Đã giả lập kịch bản gửi yêu cầu nhưng không đính kèm trường `Authorization` trong Header.
- [x] **Có test sai token hoặc token rỗng:** Đã giả lập kịch bản gửi token không hợp lệ (ví dụ: chuỗi rỗng hoặc chuỗi ký tự ngẫu nhiên bừa bãi) để kiểm tra tính bảo mật.
- [x] **Endpoint public được khai báo rõ nếu không cần auth:** Thỏa thuận rõ ràng trong OpenAPI contract rằng endpoint `/health` là public hoàn toàn, không yêu cầu `security` (mảng rỗng).
- [x] **Test thể hiện đúng expected status 401/403:** Đảm bảo khi không vượt qua vòng kiểm tra quyền truy cập, hệ thống phản hồi lỗi chuẩn xác bằng mã lỗi `401 Unauthorized` chứ không đánh tráo sang mã lỗi khác.

## 3. Negative tests
- [x] **Có test thiếu field bắt buộc:** Đã thử nghiệm gửi dữ liệu POST thiếu các thuộc tính bắt buộc cấu hình trong OpenAPI như `cameraId` hoặc `imageRaw`.
- [x] **Có test sai kiểu dữ liệu:** Đã gửi kiểm thử với tham số query `limit` mang giá trị dạng chuỗi chữ (String) thay vì số nguyên (Integer).
- [x] **Có test sai enum hoặc giá trị ngoài miền:** Đã cấu hình kiểm thử dải dữ liệu vượt ngưỡng cho phép đối với các trường nghiệp vụ được định nghĩa sẵn.
- [x] **Lỗi trả về theo cùng một error model:** Toàn bộ các phản hồi lỗi từ 400, 401 cho đến 429 đều được đồng bộ hóa cấu trúc theo chuẩn định dạng `ProblemDetails` (RFC 7807) bao gồm các trường: `type`, `title`, `status`, và `detail`.

## 4. Boundary tests
- [x] **Có test min/max hoặc dữ liệu sát ngưỡng:** Đã đặt ngưỡng biên tối đa cho số lượng bản ghi muốn lấy ra trên một trang.
- [x] **Có test limit/pagination nếu endpoint có danh sách:** Đã triển khai kiểm thử tham số `limit` ở mức tối đa cho phép là `100` để xác minh khả năng chịu tải và thuật toán phân trang của hệ thống.
- [x] **Có test payload lớn hoặc metadata thiếu:** Gửi thử nghiệm ảnh chuỗi base64 thô kích thước lớn để bảo vệ tính toàn vẹn dữ liệu đầu vào.
- [x] **Có ghi chú kỳ vọng xử lý dữ liệu biên:** Ghi chú rõ ràng trong tài liệu kỹ thuật và mã script kiểm thử để máy chủ biên dịch hiểu được cách thức chặn lọc dữ liệu sai lệch ngay từ vòng gửi request ban đầu.

## 5. Reliability tests cơ bản
- [x] **Có kiểm tra response time:** Đã nhúng đoạn mã kiểm thử thời gian phản hồi của dịch vụ đảm bảo độ trễ hệ thống nằm trong ngưỡng chấp nhận được.
- [x] **Có mô tả timeout mong muốn:** Cấu hình rõ ràng thời gian chờ tối đa (timeout) cho dịch vụ ở môi trường CI/CD (ví dụ: `wait-on` được cấu hình ngắt sau tối đa 30,000ms nếu mock server treo).
- [x] **Có test hoặc ghi chú retry/idempotency nếu phù hợp:** API phiên bản v2 (`GET /api/v2/vision/detections`) đã hỗ trợ nhận diện và kiểm soát trùng lặp thông qua Header `X-Idempotency-Key`.
- [x] **Có consumer-side smoke test với ít nhất 1 mock của nhóm khác:** Đã thiết lập thư mục kiểm thử số `05_Consumer_side_Smoke` gọi trực tiếp sang Mock Server của dịch vụ phụ thuộc để đảm bảo tính tích hợp liên tục.

## 6. Evidence
- [x] **Collection export JSON:** Đã đóng gói và lưu tệp tại đường dẫn `postman/collections/FIT4110_lab03_iot_ingestion.postman_collection.json`.
- [x] **Environment mock export JSON:** Đã cấu hình và lưu tại `postman/environments/FIT4110_lab03_mock.postman_environment.json`.
- [x] **Environment local export JSON:** Đã cấu hình và lưu tại `postman/environments/FIT4110_lab03_local.postman_environment.json`.
- [x] **Newman report XML/HTML:** Hệ thống CI tự động xuất toàn bộ kết quả báo cáo chạy thử nghiệm trực quan vào thư mục tập trung `reports/` ngay sau khi kết thúc tiến trình.
- [x] **Test-case matrix đã điền:** Đã hoàn thiện việc khai báo ma trận liên kết tại tệp `templates/test-case-matrix.csv`.
- [x] **Biên bản handshake đã điền:** Đã ký kết biên bản thống nhất bàn giao hợp đồng API tại tệp `templates/consumer-provider-handshake.md`.
# Reliability and Contract Test Checklist — Team Vision

## 1. Functional tests
- [x] **Có test cho endpoint health:** Đã định nghĩa và kiểm tra endpoint `/health`.
- [x] **Có test happy path cho endpoint chính:** Đã cấu hình cho API lấy danh sách v1/v2 và API detect.
- [x] **Có kiểm tra status code 2xx:** Assert thành công mã trạng thái phản hồi `200/201`.
- [x] **Có kiểm tra field quan trọng trong response:** Kiểm tra các trường `data`, `hasMore`, `nextCursor`.
- [x] **Có ít nhất 1 test đọc dữ liệu danh sách hoặc chi tiết:** Thực hiện trên luồng `GET /api/v1/vision/detections`.

## 2. Auth tests
- [x] **Có test thiếu token:** Gửi request không đính kèm header `Authorization`.
- [x] **Có test sai token hoặc token rỗng:** Xác thực cơ chế chặn truy cập trái phép.
- [x] **Endpoint public được khai báo rõ nếu không cần auth:** Định nghĩa rõ `/health` không bắt buộc bảo mật.
- [x] **Test thể hiện đúng expected status 401/403:** Phản hồi mã lỗi chuẩn xác theo ProblemDetails.

## 3. Negative tests
- [x] **Có test thiếu field bắt buộc:** Thử nghiệm gửi dữ liệu thiếu `cameraId` hoặc `imageRaw`.
- [x] **Có test sai kiểu dữ liệu:** Truyền tham số query `limit` ở dạng chuỗi chữ bừa bãi.
- [x] **Có test sai enum hoặc giá trị ngoài miền:** Ràng buộc chặt chẽ dữ liệu đầu vào.
- [x] **Lỗi trả về theo cùng một error model:** Đồng bộ hóa cấu trúc theo chuẩn `ProblemDetails` (RFC 7807).

## 4. Boundary tests
- [x] **Có test min/max hoặc dữ liệu sát ngưỡng:** Đặt ngưỡng chặn biên cho tham số phân trang.
- [x] **Có test limit/pagination nếu endpoint có danh sách:** Thử nghiệm tham số `limit=100` sát ngưỡng tối đa.
- [x] **Có test payload lớn hoặc metadata thiếu:** Gửi chuỗi ảnh thô mã hóa dung lượng cao.
- [x] **Có ghi chú kỳ vọng xử lý dữ liệu biên:** Máy chủ xử lý chặn lọc ngay từ vòng ngoài.

## 5. Reliability tests cơ bản
- [x] **Có kiểm tra response time:** Xác minh hiệu năng ở môi trường cục bộ.
- [x] **Có mô tả timeout mong muốn:** Cấu hình tự động ngắt nếu mock đơ cứng quá 30 giây.
- [x] **Có test hoặc ghi chú retry/idempotency nếu phù hợp:** Áp dụng trường `X-Idempotency-Key` trên phiên bản v2.
- [x] **Có consumer-side smoke test với ít nhất 1 mock của nhóm khác:** Thực thi trong thư mục test nhóm số `05`.

## 6. Evidence
- [x] **Collection export JSON:** Lưu trữ đầy đủ tại thư mục quy định.
- [x] **Environment mock export JSON:** Đóng gói lưu cấu hình máy chủ giả lập.
- [x] **Environment local export JSON:** Đóng gói lưu cấu hình máy chủ cục bộ.
- [x] **Newman report XML/HTML:** Xuất tự động toàn bộ bằng chứng kiểm thử ra thư mục `reports/`.
- [x] **Test-case matrix đã điền:** Đã điền chi tiết tại tệp CSV.
- [x] **Biên bản handshake đã điền:** Đã ký kết thỏa thuận điện tử giữa các đội.
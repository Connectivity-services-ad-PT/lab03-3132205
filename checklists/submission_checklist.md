# Submission Checklist — Lab 03 — Team Vision

Trước khi nộp bài lên LMS, nhóm đã rà soát cấu trúc thư mục repo và đảm bảo đầy đủ các thành phần bắt buộc sau:

## 📂 1. Trạng thái các tệp tin bắt buộc (Artifact Status)

- [x] **`contracts/ai-vision.openapi.yaml`** — Hợp đồng API chuẩn của nhóm, đã được dọn sạch lỗi cú pháp và đồng bộ hóa.
- [x] **`postman/collections/FIT4110_lab03_iot_ingestion.postman_collection.json`** — Bộ Postman Collection chứa đủ 6 thư mục kiểm thử từ `01` đến `06`.
- [x] **`postman/environments/FIT4110_lab03_mock.postman_environment.json`** — Tệp cấu hình môi trường Mock (`baseUrl` trỏ về port `4010`).
- [x] **`postman/environments/FIT4110_lab03_local.postman_environment.json`** — Tệp cấu hình môi trường Local (`baseUrl` trỏ về port `8000`).
- [x] **`reports/`** — Thư mục báo cáo tự động sinh ra khi chạy Newman Test, đã có bằng chứng test chạy thành công.
- [x] **`checklists/reliability_checklist.md`** — Bảng tự đánh giá độ tin cậy của API đã được điền chi tiết.
- [x] **`templates/test-case-matrix.csv`** — Ma trận ánh xạ các ca kiểm thử với endpoint thực tế.
- [x] **`templates/consumer-provider-handshake.md`** — Biên bản bàn giao và cam kết tích hợp API giữa Consumer và Provider.

---

## 🛠️ 2. Quy ước Commit & Minh chứng nộp bài

- [x] **Quy ước Commit chuẩn:** Nhóm sử dụng thông điệp commit tường minh, phản ánh đúng tiến độ hoàn thành cấu trúc Lab 03.
- [x] **Hình thức nộp bài:** Cam kết nộp đúng link GitHub Repository chính thức của nhóm lên hệ thống LMS, không nộp file rời, không nộp file nén `.zip`/`.rar`.
- [x] **Tích xanh CI/CD:** Quy trình kiểm tra tự động bao gồm `Spectral Lint` và `Newman Run` trên GitHub Actions chạy trơn tru, không gặp lỗi ngắt quãng tiến trình.
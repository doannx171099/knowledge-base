# CI/CD và Observability

Chiến lược test (unit/integration/E2E, test pyramid, kỹ thuật thiết kế test) được trình bày ở `miscellaneous/writing-tests.md`.

## 1. Thiết kế CI/CD pipeline và chọn chiến lược deploy

Một pipeline hợp lý đi từ nhanh tới chậm để phản hồi sớm:

```text
Commit
  -> Lint + type check
  -> Unit test
  -> Build
  -> Integration test (DB/queue thật)
  -> Security scan (dependency, secret, image)
  -> Deploy staging
  -> E2E test trên staging
  -> Deploy production
  -> Monitoring + alert
```

Nguyên tắc: bước nào rẻ và hay fail thì đặt trước. Không chạy E2E trước unit test, không deploy khi chưa scan secret.

Các chiến lược deploy và khi nào dùng:

- **Rolling**: thay dần instance. Đơn giản, mặc định cho nhiều hệ thống; nhược điểm là trong thời gian deploy tồn tại hai version cùng lúc, nên API/schema phải tương thích ngược.
- **Blue-Green**: hai môi trường song song, chuyển traffic khi bản mới sẵn sàng. Rollback nhanh (chuyển lại), nhưng tốn gấp đôi hạ tầng và vẫn cần migration tương thích.
- **Canary**: chuyển một phần nhỏ traffic sang bản mới, theo dõi metric rồi tăng dần. Giới hạn blast radius; cần metric đủ tốt để phát hiện khác biệt.
- **Feature flag**: deploy code nhưng tắt tính năng, bật theo nhóm người dùng. Tách deploy khỏi release; cần kỷ luật dọn flag cũ.

Với Lambda, cơ chế tương ứng là **version + alias + weighted routing**: publish version bất biến, trỏ alias sang version mới theo tỉ lệ phần trăm, và rollback bằng cách trỏ alias về version cũ.

Điểm quan trọng đi kèm: migration database phải **backward compatible** (thêm cột nullable, không đổi tên/xoá ngay) để hai version cùng chạy được trong lúc deploy. Đây là lỗi khiến rolling/canary thất bại dù code đúng.

## 2. Observability: logs, metrics và traces

Ba nhóm tín hiệu, mỗi loại trả lời một câu hỏi khác nhau:

- **Logs**: chuyện gì đã xảy ra với một request cụ thể. Cần structured (JSON), có `correlationId`/`traceId`, mức log hợp lý, không chứa dữ liệu nhạy cảm.
- **Metrics**: hệ thống đang khỏe hay không, theo thời gian. Bốn tín hiệu vàng: latency, traffic, errors, saturation. Dùng để alert và xem xu hướng; chi phí thấp nên thu thập rộng.
- **Traces**: thời gian đi qua từng tầng/service trong một request. Đây là thứ trả lời câu hỏi "chậm ở đâu" nhanh nhất.

Ví dụ khi latency là 4 giây:

```text
API Gateway       20 ms
Order Service    100 ms
Payment Service 3600 ms
Database          50 ms
```

Chỉ nhìn log thì khó thấy; nhìn metric thấy service chậm nhưng không biết tầng nào; trace chỉ ra ngay dependency bên ngoài. Trong hệ AWS thường dùng CloudWatch (logs/metrics), X-Ray (trace), hoặc OpenTelemetry để không bị khoá vào một vendor.

Hai điều dễ thiếu:

- **Correlation ID** phải được tạo ở entry point và truyền qua mọi lời gọi, kể cả async (queue message mang theo id). Không có nó, log của một request nằm rải rác không nối được.
- **Alert phải hành động được**. Alert trên "CPU > 80%" thường ít giá trị hơn alert trên "P99 latency vượt SLO" hoặc "DLQ có message". Mỗi alert nên có runbook: ai xử lý, làm gì đầu tiên.

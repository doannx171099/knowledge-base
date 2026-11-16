# AWS và Serverless

## 1. Cold start của Lambda là gì và tối ưu thế nào?

Một Lambda invocation có thể chạy trên một execution environment đã tồn tại (warm) hoặc phải tạo mới (cold). Cold start gồm các bước: cấp phát environment, khởi tạo runtime, tải code và dependencies, chạy phần khởi tạo ngoài handler, rồi mới gọi handler.

Thời gian cold start phụ thuộc: runtime (Java/.NET chậm hơn Node/Python), kích thước package, công việc trong init phase (kết nối DB, load config, khởi tạo SDK), cấu hình VPC, và lượng memory (memory quyết định luôn CPU).

Cách giảm:

- Giảm kích thước bundle, loại dependencies không dùng, tránh import nặng ở top-level.
- Khởi tạo client và kết nối ở ngoài handler để tái sử dụng giữa các invocation trên cùng environment.
- Tránh đưa Lambda vào VPC nếu không thật sự cần (VPC làm tăng thời gian khởi tạo, dù ENI được tái sử dụng).
- Dùng provisioned concurrency khi latency yêu cầu chặt và traffic có baseline ổn định.

Điều quan trọng về mặt tư duy: đừng tối ưu cold start nếu nó không phải bottleneck thực tế. Với API có P99 yêu cầu 200ms và traffic thấp, provisioned concurrency có thể còn đắt hơn giá trị nó mang lại. Cần đo trước.

## 2. Vì sao Lambda + PostgreSQL dễ gây sự cố?

Lambda scale theo số request đồng thời, gần như không giới hạn theo mặc định (có thể lên tới hàng nghìn). PostgreSQL thì scale theo connection, và mỗi connection tốn memory cùng tài nguyên của server.

Nếu mỗi invocation mở một connection:

```text
100 concurrent Lambda  -> 100 connections
1000 concurrent Lambda -> 1000 connections
```

Vượt `max_connections` thì kết nối mới bị từ chối; trước khi tới ngưỡng đó, database đã chậm đi vì context switch và memory pressure. Hệ quả dây chuyền: query timeout → Lambda timeout → retry → thêm connection → sập hẳn.

Các lớp bảo vệ:

- **RDS Proxy**: pool connection tập trung, Lambda kết nối tới proxy thay vì DB. Giải quyết trực tiếp vấn đề connection storm.
- **Giới hạn reserved concurrency** của Lambda để số invocation đồng thời không vượt quá khả năng của DB.
- **Tái sử dụng connection** giữa các invocation (khởi tạo pool ngoài handler), nhưng lưu ý mỗi environment vẫn giữ pool riêng.
- **Đệm bằng queue**: API đẩy job vào SQS, consumer xử lý với concurrency cố định.

Kiến trúc an toàn hơn:

```text
API -> Lambda -> RDS Proxy -> RDS
```

Nguyên tắc chung: **concurrency của compute phải bị chặn bởi khả năng của tài nguyên có state**, không được để nó tự do scale.

## 3. SQS, SNS, EventBridge và Kinesis khác nhau ở đâu?

Chọn service theo mô hình giao tiếp, không theo độ quen thuộc:

| Service | Mô hình | Dùng khi |
|---|---|---|
| SQS | Hàng đợi công việc, một consumer nhóm | Background job, giảm tải, retry, buffer |
| SNS | Pub/sub fan-out | Một event cần nhiều nơi nhận độc lập |
| EventBridge | Event bus có rule routing | Tích hợp nhiều nguồn, lọc theo nội dung event |
| Kinesis | Stream, có thứ tự trong shard | Throughput cao, replay, analytics, CDC |

Cụ thể hơn:

- **SQS**: producer đẩy message, một nhóm consumer xử lý. Không có khái niệm subscriber; message bị xoá sau khi xử lý xong. Đây là lựa chọn mặc định cho "làm việc này sau".
- **SNS**: một message được đẩy tới nhiều subscriber (SQS, Lambda, HTTP endpoint). Không lưu trữ, không retry lâu dài; nếu subscriber chết thì mất. Thường dùng SNS + SQS để fan-out bền vững.
- **EventBridge**: bus trung tâm với rule matching theo `source`/`detail-type`/nội dung. Phù hợp khi nhiều team, nhiều service cùng phát và nhận event, cần thêm target mà không sửa producer.
- **Kinesis**: stream có retention, đọc lại được, thứ tự đảm bảo trong một shard theo partition key. Phù hợp clickstream, IoT, CDC, nơi cần nhiều consumer đọc cùng dữ liệu hoặc replay.

Ví dụ chọn nhanh: gửi email sau khi tạo order → SQS. Một `OrderCreated` cần email + analytics + fulfillment → SNS/SQS fan-out hoặc EventBridge. Phân tích hành vi người dùng theo thời gian thực → Kinesis.

## 4. SQS Standard và FIFO khác nhau ở đâu?

**Standard** ưu tiên throughput và khả năng scale: at-least-once, thứ tự best-effort, có thể có duplicate. Với hầu hết workload, đây là lựa chọn đúng, miễn là consumer idempotent.

**FIFO** đảm bảo thứ tự trong phạm vi message group và có deduplication theo `MessageDeduplicationId`. Throughput thấp hơn, và bị giới hạn theo số message group đang hoạt động.

Câu hỏi quan trọng không phải "có cần FIFO không" mà là **cần thứ tự tới mức nào**:

- Cần thứ tự **theo từng entity** (event của order A phải đúng thứ tự, nhưng A và B xử lý song song được): dùng FIFO với `MessageGroupId = orderId`. Các group khác nhau vẫn xử lý song song.
- Cần thứ tự **toàn hệ thống**: dùng một group duy nhất, nhưng biến toàn bộ pipeline thành tuần tự và giảm throughput rất mạnh. Hiếm khi thật sự cần.

Nếu chọn sai, hệ quả không chỉ là chậm. FIFO có thể tạo head-of-line blocking: một message độc hại trong group làm kẹt toàn bộ group đó cho tới khi hết retry.

## 5. Lambda xử lý batch SQS, một message lỗi thì sao?

Với SQS trigger, Lambda nhận một batch (mặc định tới 10 message). Nếu handler throw exception mà không báo cáo cụ thể message nào lỗi, **toàn bộ batch** được coi là thất bại và tất cả message quay lại queue — kể cả những message đã xử lý thành công. Kết quả là duplicate processing.

Cách xử lý đúng:

- Bật **partial batch response** (`ReportBatchItemFailures`), trả về danh sách `itemIdentifier` của các message thất bại. Lambda chỉ trả lại những message đó.
- Vẫn phải đảm bảo consumer idempotent, vì message đã xử lý nhưng chưa kịp báo cáo vẫn có thể bị giao lại.
- Cấu hình `maxReceiveCount` để message lỗi mãi được chuyển sang DLQ thay vì quay vòng vô hạn.

```text
SQS -> Lambda (batch) -> xử lý từng message
                          ├── thành công -> không trả về
                          └── thất bại   -> itemIdentifier
     sau N lần -> DLQ
```

Một chi tiết dễ bỏ sót: nếu một message trong batch làm handler crash (OOM, timeout), không có partial response nào được trả về, cả batch bị coi là thất bại. Vì vậy kích thước batch và thời gian xử lý cần được chọn theo khả năng thực tế của handler.

## 6. DLQ dùng để làm gì?

DLQ (Dead Letter Queue) là nơi nhận các message không xử lý được sau số lần thử cho phép. Ví dụ message có `orderId: null` khiến consumer luôn lỗi; nếu không có DLQ, nó quay vòng mãi, chiếm throughput, làm nhiễu metric và có thể chặn cả queue FIFO.

Luồng:

```text
Main queue -> retry N lần -> DLQ
```

Giá trị thật của DLQ nằm ở quy trình xung quanh nó:

- **Alarm trên số message trong DLQ**, nếu không nó chỉ là nơi chôn lỗi.
- Công cụ để xem nội dung, xác định nguyên nhân, sửa dữ liệu.
- **Redrive** để đưa message trở lại sau khi fix.
- Runbook ghi rõ ai xử lý, trong bao lâu.

DLQ không thay thế cho việc phân loại lỗi: message lỗi do dữ liệu sai nên được sửa và redrive, còn lỗi do bug nên được fix code. Nếu DLQ luôn đầy, đó là tín hiệu chất lượng dữ liệu hoặc logic consumer có vấn đề.

## 7. Vì sao dùng CDK thay vì tạo resource trên Console?

Console phù hợp để khám phá và thử nghiệm, nhưng không phù hợp làm nguồn sự thật cho production: không có review, không có lịch sử thay đổi, không tái tạo được môi trường, và rất khó rollback.

CDK đưa hạ tầng vào cùng quy trình với code:

```text
Git -> Pull Request -> Review -> CI -> cdk synth -> CloudFormation -> AWS
```

Lợi ích thực tế: version control, tái lập môi trường (dev/staging/prod giống nhau), review được thay đổi hạ tầng trước khi áp dụng, tự động hoá deploy, và khôi phục sau sự cố.

Cần hiểu rõ CDK **không trực tiếp tạo resource**. Nó tổng hợp (synth) ra CloudFormation template, và CloudFormation mới là thứ gọi API AWS. Điều này giải thích nhiều hành vi: drift khi có người sửa tay trên Console, thời gian deploy, và cách rollback hoạt động.

## 8. Construct, Stack và App khác nhau thế nào?

Ba khái niệm cần phân biệt:

- **Construct**: đơn vị cấu hình, từ một resource (`lambda.Function`) tới một cụm hoàn chỉnh (API + queue + worker).
- **Stack**: đơn vị deploy, tương ứng một CloudFormation stack.
- **App**: container chứa nhiều stack.

Ví dụ tách stack:

```text
App
 ├── NetworkStack
 ├── DatabaseStack
 ├── BackendStack
 └── MonitoringStack
```

Tách stack giúp giới hạn phạm vi ảnh hưởng khi deploy và tránh chạm tới database khi chỉ sửa code. Cái giá là phải quản lý cross-stack reference và thứ tự deploy; tách quá nhỏ sẽ khiến việc deploy rối hơn lợi ích.

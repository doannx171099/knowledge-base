# Kafka và RabbitMQ

## 1. Hai triết lý khác nhau

Cả hai đều bị gọi chung là message broker, nhưng mô hình tư duy khác nhau:

- **RabbitMQ**: "gửi message cho consumer xử lý". Message là một công việc; xử lý xong thì biến mất.
- **Kafka**: "ghi event vào một distributed log để consumer đọc". Message là lịch sử; tồn tại theo retention, nhiều consumer đọc độc lập.

Sự khác biệt này kéo theo gần như toàn bộ khác biệt về storage, scaling, ordering, replay và cách xử lý lỗi.

## 2. Mô hình lưu trữ

**RabbitMQ** — queue:

```text
Producer → Queue → Consumer → ACK → message bị xoá
```

Queue thể hiện "những việc đang chờ xử lý".

**Kafka** — partitioned log:

```text
Producer → Topic/Partition → append
Consumer đọc theo offset, message vẫn còn
```

Topic thể hiện "lịch sử event", retention quyết định giữ bao lâu (7 ngày, 30 ngày, hoặc lâu hơn).

Hệ quả: RabbitMQ tối ưu cho việc phân phối việc; Kafka tối ưu cho việc nhiều nơi đọc cùng một dòng sự kiện, kể cả đọc lại.

## 3. Consumer model

**RabbitMQ**: nhiều consumer cùng đọc một queue (competing consumers). Mỗi message thường được một consumer xử lý. Dễ hình dung, thêm worker là tăng throughput.

**Kafka**: consumer group. Mỗi partition được assign cho đúng một consumer trong group tại một thời điểm.

```text
4 partitions, 2 consumers → C1: P0,P1; C2: P2,P3
```

Hệ quả quan trọng:

```text
parallelism tối đa ≈ số partition
```

10 partition mà chạy 50 consumer thì 10 consumer làm việc, 40 ngồi không. Vì vậy số partition là quyết định kiến trúc, không phải tham số tuỳ chỉnh sau này (tăng partition làm thay đổi mapping key → partition).

## 4. Fan-out

RabbitMQ: exchange route message vào nhiều queue, mỗi queue có consumer riêng.

Kafka: nhiều consumer group đọc cùng topic, mỗi group có offset riêng. Payment, inventory, analytics đều đọc toàn bộ stream mà không ảnh hưởng nhau.

Hai cách đều fan-out được, nhưng Kafka xem "nhiều độc giả của cùng lịch sử" là tính năng cốt lõi.

## 5. Routing

RabbitMQ mạnh hơn rõ rệt: exchange (direct, topic, fanout, headers) và routing key cho phép route theo pattern:

```text
order.*        → order.created, order.cancelled
*.failed       → payment.failed, order.failed
```

Kafka chỉ có topic, partition và key. Muốn route theo nội dung, phải tự lọc ở consumer hoặc tách topic. Nếu yêu cầu chính là định tuyến phức tạp, RabbitMQ phù hợp hơn.

## 6. Ordering

Kafka đảm bảo thứ tự trong một partition. Dùng `key = orderId` để mọi event của một order vào cùng partition.

RabbitMQ giữ thứ tự trong queue, nhưng khi có nhiều consumer thì **thứ tự hoàn thành** không còn đảm bảo: M1 giao cho A, M3 giao cho C, C xong trước A. Delivery order có thể đúng, processing order thì không.

Vì vậy với RabbitMQ, muốn ordering theo entity phải dùng một queue cho một key (hoặc single consumer), thường tốn kém hơn về hạ tầng.

## 7. Replay và event sourcing

Kafka cho phép `seek(offset)` về đầu và đọc lại toàn bộ. Ứng dụng thực tế:

- Analytics service có bug → sửa code → replay từ offset 0 để rebuild.
- Database read model bị mất → rebuild từ event.
- Thêm consumer mới → đọc lại lịch sử để khởi tạo state.

RabbitMQ không được thiết kế cho việc này: message đã ACK là bị xoá khỏi queue.

Vì vậy Kafka phù hợp với event sourcing, CQRS, CDC, data pipeline; RabbitMQ không phải lựa chọn đầu tiên cho các use case đó.

## 8. Task queue, retry và DLQ

RabbitMQ có sẵn mô hình phù hợp cho job: `prefetch` để giới hạn unacked, `nack/reject` với `requeue`, TTL, dead-letter exchange (DLX).

```text
Main queue → consumer fail → retry queue (delay) → main queue
                            → sau N lần → DLQ
```

Kafka không có abstraction này. Muốn retry phải tự tổ chức topic: `orders`, `orders.retry`, `orders.dlq`, hoặc dùng cơ chế riêng của framework. Làm được, nhưng nhiều việc hơn.

Kết luận thực dụng: workload "job" (gửi email, render PDF, resize ảnh) thường tự nhiên hơn với RabbitMQ; workload "event stream" tự nhiên hơn với Kafka.

## 9. Pull, push, throughput

Kafka chủ yếu **pull**: consumer chủ động fetch, nhờ đó tự điều tiết tốc độ và không bị broker đẩy quá tải. Throughput cao nhờ ghi tuần tự, batching, partition, nén và zero-copy.

RabbitMQ **push** message tới consumer theo mô hình delivery, kết hợp `prefetch` để giới hạn số message chưa ACK.

Về throughput, Kafka thường cao hơn ở quy mô lớn, nhưng "nhanh hơn" không phải lý do duy nhất để chọn — semantics mới là lý do chính.

## 10. Command hay event: mô hình để chọn

- **Command** (`SendEmail`, `GenerateInvoice`, `ResizeImage`): "hãy làm việc này". Một nơi xử lý, cần retry, DLQ, có thể cần priority hoặc delay → RabbitMQ.
- **Event** (`OrderCreated`, `PaymentCompleted`): "điều này đã xảy ra". Nhiều consumer độc lập, cần replay, lịch sử → Kafka.

Bảng chọn nhanh:

| Yêu cầu | Cân nhắc |
|---|---|
| Background job, email, image processing | RabbitMQ |
| Routing phức tạp, priority, delay | RabbitMQ |
| Retry/DLQ tự nhiên | RabbitMQ |
| Event streaming, nhiều consumer group | Kafka |
| Replay, event sourcing, audit history | Kafka |
| CDC, analytics pipeline, throughput lớn | Kafka |

Đây là heuristic, không phải luật. Nếu chỉ cần xử lý vài job nền, dựng cả Kafka cluster là overengineering.

## 11. Delivery semantics

Cả hai đều có thể giao trùng. Kết luận thực tế: giả định **at-least-once** và thiết kế consumer idempotent.

Câu "Kafka đảm bảo exactly-once nên không cần idempotency" là sai ở phạm vi hệ thống. Kafka có transaction và exactly-once trong pipeline nội bộ Kafka, nhưng khi consumer gọi payment API hoặc ghi sang database khác thì không còn đảm bảo đó.

Ví dụ idempotency:

```text
PaymentCompleted(paymentId = ABC123)
consumer kiểm tra processed_event(ABC123) → đã có → bỏ qua
```

## 12. CDC và data pipeline

Kafka rất phù hợp với CDC: PostgreSQL WAL → Debezium → Kafka → Elasticsearch/warehouse/cache/analytics. Event có dạng `before/after`, giữ được lịch sử thay đổi, nhiều downstream đọc độc lập.

Với RabbitMQ, mô hình này khó hơn vì message bị xoá sau khi xử lý và không có replay.

## 13. Ví dụ hệ thống dùng cả hai

Một hệ thương mại điện tử thực tế thường dùng hybrid:

- **RabbitMQ**: `SendEmail`, `GenerateInvoicePDF`, `ResizeProductImage` — command, cần retry/DLQ, không cần lịch sử.
- **Kafka**: `OrderCreated`, `OrderPaid`, `InventoryChanged` — event, nhiều consumer (shipping, analytics, loyalty, fraud, notification), cần replay và throughput.

Chọn theo **semantics và access pattern**, không theo độ phổ biến của công nghệ.

## 14. Câu trả lời mẫu khi được hỏi chọn cái nào

"Tôi chọn Kafka khi message thực chất là event stream cần lưu giữ, replay và được nhiều hệ thống consume độc lập — ví dụ `OrderCreated` cho shipping, analytics, fraud và notification. Kafka cũng phù hợp khi cần throughput lớn và ordering theo entity key.

Tôi ưu tiên RabbitMQ khi workload là task queue: một công việc cần một worker xử lý, đặc biệt khi cần routing phức tạp, retry, DLQ, priority hoặc delayed message.

Tôi không chọn Kafka chỉ vì throughput cao hơn; tôi chọn dựa trên messaging semantics."

# Microservices

## 1. Vì sao dùng Microservices, và vì sao không nên mặc định?

Câu hỏi "tại sao không dùng monolith" không nhằm kiểm tra bạn thuộc ưu điểm của microservices, mà để xem bạn có nhận ra **chi phí của distributed system** hay không.

Microservices có lợi khi:

- Các domain có vòng đời phát triển độc lập.
- Nhiều team cần deploy độc lập.
- Một số thành phần cần scale rất khác nhau (search 50.000 req/s so với payment 500 req/s).
- Cần fault isolation.
- Tổ chức đã đủ lớn để chi phí vận hành trở nên hợp lý.

Đổi lại phải xử lý: network failure, distributed transaction, eventual consistency, service discovery, observability phân tán, versioning, đồng bộ dữ liệu, testing phức tạp và pipeline deploy nhân N.

Ví dụ cụ thể về scaling: monolith phải nhân bản cả app 20 lần, trong khi microservices có thể cho search 30 instance, payment 5, order 10. Nhưng nếu traffic chưa đủ lớn để sự khác biệt này có ý nghĩa, bạn đang trả giá vận hành cho một vấn đề chưa tồn tại.

Cách trả lời: "Tôi chọn microservices khi lợi ích từ independent deployment, team autonomy, independent scaling và bounded context đủ lớn để bù lại complexity; nếu chưa, modular monolith là mặc định" — xem `backend/modular-monolith.md`.

## 2. Chia service theo tiêu chí nào?

Sai lầm phổ biến là chia theo bảng dữ liệu:

```text
users table   → User Service
orders table  → Order Service
address table → Address Service
```

Kết quả là **distributed monolith**: các service gọi nhau liên tục (`Order → User → Address → Country → Currency`), deploy độc lập nhưng không thể thay đổi độc lập.

Chia theo **business capability / bounded context**: Identity, Catalog, Ordering, Payment, Inventory, Shipping. Tiêu chí để quyết định một boundary:

- **Business ownership**: ai chịu trách nhiệm nghiệp vụ này.
- **Data ownership**: dữ liệu nào thuộc service nào.
- **Change together**: hai phần có luôn phải sửa cùng lúc không (nếu có, có thể cùng service).
- **Consistency requirement**: có cần nhất quán tức thời giữa chúng không.
- **Independent scaling**: có nhu cầu tải rất khác nhau không.

Ví dụ "Product" trong Catalog (tên, ảnh, mô tả) và trong Inventory (SKU, số lượng, kho) không cần cùng model. Việc tách model không phải trùng lặp xấu, mà là hệ quả tự nhiên của hai bounded context.

## 3. Database per service: ownership quan trọng hơn vị trí vật lý

Nguyên tắc: mỗi service sở hữu dữ liệu của mình, service khác không đọc trực tiếp.

```text
Order Service    → order schema
Payment Service  → payment schema
Inventory        → inventory schema
```

Không nhất thiết ba server vật lý; có thể cùng một PostgreSQL cluster với schema riêng. Điều bắt buộc là **ownership**: Order không được `SELECT * FROM payment.payments`, vì khi Payment đổi schema, Order sẽ vỡ mà không ai cảnh báo.

Khi Order cần trạng thái payment, có hai cách:

- **Gọi API** `GET /payments/{id}`: dữ liệu mới nhất, nhưng tạo runtime coupling — Payment down thì Order bị ảnh hưởng.
- **Nghe event** `PaymentCompleted` và lưu `payment_status` trong order: giảm coupling runtime, nhưng eventual consistency.

Duplicate dữ liệu có chủ đích là bình thường. Order có thể lưu snapshot `shippingAddress` thay vì chỉ `addressId` — cần thiết để địa chỉ lịch sử không đổi khi người dùng cập nhật địa chỉ. Điều phải xác định rõ là **source of truth** cho từng trường.

## 4. Distributed transaction và compensating transaction

Không thể `BEGIN ... COMMIT` xuyên nhiều database. Các hướng:

- **Two-phase commit**: hiếm dùng trong microservices vì blocking, phụ thuộc coordinator, giảm availability.
- **Saga**: chia thành các local transaction, mỗi bước có hành động bù.

Ví dụ checkout:

```text
Create Order ✓ → Reserve Inventory ✓ → Charge Payment ✗
```

Không rollback được bước 1 và 2 bằng database; phải chạy compensating transaction: `ReleaseInventory`, `CancelOrder`. Cơ chế saga (choreography/orchestration) và outbox được trình bày chi tiết ở `backend/distributed-systems.md`; ở đây chỉ cần nắm hệ quả: **saga không đảm bảo ACID toàn hệ thống**, nó hướng tới consistency qua local transaction + compensation, và compensation cũng phải idempotent.

## 5. At-most-once, at-least-once và exactly-once

Ba mức delivery:

- **At-most-once**: gửi tối đa một lần, có thể mất message. Phù hợp khi mất dữ liệu không nghiêm trọng (metric, log).
- **At-least-once**: không mất nhưng có thể trùng. Đây là mức phổ biến của SQS, Kafka, Kinesis, RabbitMQ khi có retry.
- **Exactly-once**: chỉ đúng một lần.

Cần cẩn thận khi nói về exactly-once. Kafka có exactly-once semantics trong một số pipeline nội bộ Kafka (transactional producer, read-process-write), nhưng khi pipeline đi ra ngoài — gọi payment API, ghi vào database khác — không có gì đảm bảo toàn hệ thống exactly-once. Cách nói đúng: "exactly-once trong phạm vi X", và consumer vẫn phải idempotent.

Hệ quả thiết kế: mặc định giả định **at-least-once + consumer idempotent**. Chi tiết cách làm idempotency nằm ở `backend/distributed-systems.md`.

## 6. Message ordering: cần thứ tự ở phạm vi nào?

Global ordering rất đắt và hiếm khi cần. Thường chỉ cần **ordering theo entity**: mọi event của order 123 phải đúng thứ tự, còn order 123 và 456 không cần liên quan.

Cách làm: partition key = entity id (`orderId`, `userId`, `paymentId`), để mọi event của cùng entity vào cùng partition/shard.

Nhưng partition ordering chưa đủ. Producer retry có thể tạo duplicate, nhiều nguồn có thể phát event, và consumer có thể nhận lại event cũ. Vì vậy nên mang thêm **aggregate version**:

```json
{ "orderId": "123", "version": 7 }
```

Consumer đang giữ version 7 mà nhận version 6 → biết là stale, bỏ qua. Nhận version 9 khi đang ở 7 → biết thiếu version 8, có thể chờ/retry, fetch state hiện tại, đưa vào DLQ hoặc chạy reconciliation tùy yêu cầu.

Trade-off: một partition cho toàn hệ thống đảm bảo thứ tự tuyệt đối nhưng biến pipeline thành một luồng tuần tự, mất khả năng scale. Chi tiết cho Kinesis/DMS ở `backend/kinesis-ordering.md`.

## 7. Giao tiếp đồng bộ hay bất đồng bộ?

**Đồng bộ** (HTTP/gRPC): phù hợp khi caller cần kết quả ngay để quyết định. Ưu: dễ suy luận, có kết quả tức thì. Nhược: coupling runtime, latency cộng dồn, lỗi lan truyền.

**Bất đồng bộ** (message/event): phù hợp khi không cần kết quả ngay, cần buffer traffic, hoặc nhiều consumer độc lập. Ưu: giảm coupling, chịu lỗi tốt hơn, buffer được spike. Nhược: eventual consistency, duplicate, ordering, debug khó.

Lỗi thiết kế thường gặp là biến mọi thứ thành async rồi phải dựng lại cơ chế chờ kết quả (poll, webhook, callback). Hệ thống thực tế thường hybrid: luồng chính đồng bộ, side effect async.

## 8. Cascading failure và các lớp bảo vệ

Kịch bản: `Client → A → B → C → D`, D chết. C timeout, B timeout, A timeout, cuối cùng toàn hệ thống chết dù chỉ một service hỏng.

Các lớp cần có:

- **Timeout**: không bao giờ chờ vô hạn, nếu không connection/thread bị giữ.
- **Bounded retry**: giới hạn số lần và tổng thời gian.
- **Exponential backoff + jitter**: tránh dồn tải.
- **Circuit breaker**: khi dependency chết, fail nhanh thay vì chờ timeout.
- **Bulkhead**: cô lập tài nguyên — payment pool 20, shipping pool 20, để payment treo không kéo shipping chết theo.
- **Backpressure / rate limiting**: chặn tải vượt khả năng.
- **Async khi có thể**: queue buffer thay vì giữ request.

Điểm cần nêu: resilience không phải một thư viện, mà là nhiều lớp, và mỗi lớp có cái giá (thêm độ trễ, thêm phức tạp).

## 9. Retry nguy hiểm ở đâu?

Ba vấn đề lớn:

**Duplicate side effect.** Payment xử lý thành công nhưng response bị mất; client thấy timeout và retry, khách bị trừ tiền hai lần. Cần idempotency key ở cả request và provider.

**Retry amplification.** `A → B → C → DB`, mỗi tầng retry 3 lần thì một request ở A tạo ra 27 lần chạm DB. 1.000 request thành 27.000 attempt, đúng lúc DB đang yếu nhất. Cần giới hạn retry ở một tầng (thường là tầng gần người dùng hoặc tầng có ngữ cảnh nghiệp vụ), các tầng khác fail nhanh.

**Retry đồng bộ.** Nhiều client cùng retry tại cùng thời điểm tạo ra các cơn sóng tải đều đặn. Jitter phân tán thời điểm.

Ngoài ra cần **deadline propagation**: nếu request của người dùng chỉ còn 400ms, không nên bắt đầu gọi downstream với timeout 2 giây. Deadline phải được truyền xuống và cắt bớt ở mỗi tầng.

## 10. Service discovery và API Gateway

**Service discovery**: instance thay đổi liên tục, không được hardcode IP. Trong Kubernetes thường dùng Service + DNS (`http://payment-service`); ngoài Kubernetes có Consul, Eureka hoặc cơ chế của cloud provider.

**API Gateway** xử lý các concern chung ở biên: authentication, routing, rate limiting, TLS termination, logging, request aggregation. Cái bẫy là biến gateway thành nơi chứa business logic — khi đó nó trở thành một monolith mới, và mọi thay đổi đều phải deploy gateway.

## 11. Versioning API và event: expand-and-contract

Trong microservices, nhiều version chạy đồng thời: Order v3/v4 gọi Payment v7/v8. Đổi response từ `amount: 100` sang `amount: { value, currency }` sẽ làm Order cũ chết ngay.

Nguyên tắc: **backward compatibility trước, version sau**. Cách làm phổ biến là expand-and-contract:

1. **Expand**: thêm field mới (`money: { value, currency }`) nhưng giữ field cũ.
2. **Migrate**: consumer chuyển sang dùng field mới.
3. **Contract**: khi không còn consumer cũ, xoá field cũ.

Với event còn khó hơn vì không biết hết consumer và thời điểm họ upgrade. Vì vậy event chỉ nên **thêm field**, tránh đổi tên, đổi kiểu, hoặc xoá. Versioning (`/api/v2/...`) không thay thế backward compatibility — nó chỉ là cách tách khi buộc phải breaking change.

## 12. Testing microservices: contract test

Không thể chỉ dựa vào E2E vì số lượng service và tổ hợp quá lớn. Cần:

- Unit test cho logic nghiệp vụ.
- Integration test cho ranh giới (DB, broker, HTTP adapter).
- **Consumer-driven contract test**: consumer định nghĩa contract nó cần (ví dụ Order cần Payment trả `paymentId`, `status`), provider verify trong CI. Công cụ: Pact.
- E2E cho vài luồng quan trọng nhất.

Giá trị của contract test: phát hiện breaking change ở provider trước khi deploy, thay vì để E2E chậm và flaky phát hiện muộn.

## 13. CAP theorem: hiểu đúng, đừng dán nhãn

CAP nói rằng khi có network partition, hệ thống phân tán phải chọn giữa consistency và availability. Điều cần hiểu là partition **sẽ** xảy ra, nên câu hỏi thực tế là: khi partition, bạn muốn từ chối ghi để giữ nhất quán, hay vẫn phục vụ và chấp nhận dữ liệu phân kỳ?

Không nên trả lời kiểu "MongoDB là AP, PostgreSQL là CP". Hành vi phụ thuộc cấu hình, mức replication, loại read/write consistency và use case. Ví dụ cùng một hệ thống có thể cấu hình read concern khác nhau cho từng thao tác.

## 14. Security giữa các service

Service nằm trong private network không có nghĩa là đáng tin. Cần:

- **Service identity**: mTLS hoặc token do platform cấp, để biết request đến từ service nào.
- **Authorization**: không chỉ "đã đăng nhập", mà "service này có quyền gọi endpoint này không".
- **Least privilege**: mỗi service chỉ có quyền tới tài nguyên của mình.
- **Secret management**: secret manager, xoay vòng, không nhúng trong image.
- **Propagate identity**: nếu cần kiểm tra quyền của người dùng cuối ở service downstream, phải truyền ngữ cảnh phù hợp (token có scope hẹp, không dùng lại token của user cho mọi service).

## 15. Event-driven: command khác event

- **Command**: yêu cầu ai đó làm gì (`CreatePayment`, `SendEmail`). Kỳ vọng có một nơi xử lý; thường dùng queue, có thể fail và retry.
- **Event**: điều đã xảy ra (`OrderCreated`, `PaymentCompleted`). Nhiều consumer độc lập, không kỳ vọng phản hồi.

Đặt tên event kiểu `DoSomething` là dấu hiệu đang dùng event bus như RPC trá hình. Hệ quả là coupling ngầm, khó mở rộng consumer, và khó suy luận về luồng.

**Eventual consistency và UX**: người dùng vừa đặt hàng nhưng lịch sử đơn chưa thấy đơn là hiện tượng bình thường của read model chưa cập nhật. Cách xử lý không phải giả vờ nhất quán, mà thiết kế UI phù hợp: hiển thị "đang xử lý", đọc từ nguồn chính cho màn hình vừa thao tác, hoặc chấp nhận trễ có thông báo.

## 16. Case study: thiết kế checkout bằng microservices

Bắt đầu từ yêu cầu, không từ danh sách service.

**Yêu cầu**: browse/search, add to cart, checkout, pay, track, cancel. Non-functional: 10k checkout/phút, không trừ tiền hai lần, search có thể trễ vài giây.

**Boundary**: Order, Inventory, Payment tách vì ownership và lifecycle tương đối độc lập; Catalog/Search tách vì nhu cầu đọc và scaling khác.

**Checkout flow**:

```text
POST /orders
  → tạo order status=PENDING + outbox event (cùng transaction)
  → OrderCreated
  → ReserveInventory
  → ChargePayment (Idempotency-Key = pay-order-123)
  → ConfirmOrder
```

**Failure**: hết hàng → order CANCELLED; payment fail → ReleaseInventory + CancelOrder. Compensation phải idempotent.

**Tồn kho**: atomic update `SET available = available - 1 WHERE product_id = ? AND available > 0`, kiểm tra affected rows, không read-then-write.

**Search**: read model riêng (Elasticsearch) cập nhật từ event của Catalog/Inventory/Review; chấp nhận eventual consistency.

**Notification/analytics**: async, không nằm trên critical path.

**Reliability**: timeout, bounded retry, circuit breaker cho sync; DLQ, retry topic, idempotent consumer, schema versioning cho async.

**Observability**: một `traceId` xuyên suốt, metric theo `checkout_success_rate`, `payment_failure_rate`, `consumer_lag`.

Chuỗi logic cần nắm — mỗi bước là hệ quả của bước trước, không phải pattern học thuộc:

```text
database per service
  → không có ACID xuyên service
  → saga + compensation
  → cần publish event tin cậy
  → outbox
  → at-least-once
  → duplicate
  → idempotency
  → cần ordering
  → partition key + version
  → lỗi phân tán
  → timeout, retry, breaker
  → debug khó
  → observability
```

Khi trình bày, nêu rõ giả định và trade-off ở từng bước, thay vì đọc tên pattern.

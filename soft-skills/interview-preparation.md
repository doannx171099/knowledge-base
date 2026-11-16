# Chuẩn bị Phỏng vấn Middle–Senior

## 1. Ở level này interviewer đánh giá điều gì?

Với stack Node.js/NestJS + SQL + React/Next.js + AWS, phỏng vấn Middle–Senior không còn hỏi "API này dùng để làm gì". Trọng tâm chuyển sang bốn thứ:

- Thiết kế hệ thống từ yêu cầu, không từ danh sách công nghệ.
- Phân tích trade-off và nói được cái giá của lựa chọn.
- Xử lý sự cố production có phương pháp.
- Ra quyết định kỹ thuật và giải thích được với người khác.

Các nhóm kiến thức cần phủ:

| Nhóm | Nội dung cốt lõi |
|---|---|
| Node.js | Event loop, concurrency, CPU-bound, memory, stream, pooling |
| NestJS | Phân lớp, DI, guard/pipe/interceptor, ranh giới module |
| SQL | Index, execution plan, transaction, isolation, pagination |
| Distributed systems | Retry, idempotency, outbox, saga, circuit breaker |
| Kiến trúc | Modular monolith, microservices, bounded context, ranh giới module |
| Messaging | Kafka vs RabbitMQ, Kinesis ordering, partition key, dedupe |
| System design | Requirements, data model, scale, failure mode |
| AWS | Lambda, SQS/SNS/EventBridge/Kinesis, IAM, CDK |
| Security | AuthN/AuthZ, JWT, CORS, XSS/CSRF, IAM least privilege |
| Frontend | React rendering, Next.js server/client, caching |
| Testing & vận hành | Test pyramid, kỹ thuật viết test, CI/CD, observability |

Kiến thức chi tiết của từng nhóm nằm ở các file tương ứng trong `backend/`, `database/`, `frontend/`, `miscellaneous/`.

## 2. Thứ tự ưu tiên ôn tập

Nếu thời gian có hạn, phân bổ theo thứ tự sau (theo mức độ ảnh hưởng tới quyết định tuyển):

| Ưu tiên | Chủ đề | File |
|---|---|---|
| 1 | System design | `backend/system-design.md` |
| 2 | SQL: index, transaction | `database/` |
| 3 | Node.js internals | `backend/nodejs-runtime.md` |
| 4 | Distributed systems | `backend/distributed-systems.md` |
| 5 | Kiến trúc: monolith hay microservices | `backend/modular-monolith.md`, `backend/microservices.md` |
| 6 | AWS architecture | `backend/aws-serverless.md` |
| 7 | NestJS architecture | `backend/nestjs-architecture.md` |
| 8 | React internals | `frontend/react-rendering.md` |
| 9 | Next.js rendering/caching | `frontend/nextjs-rendering-and-caching.md` |
| 10 | Security | `backend/security.md` |
| 11 | Messaging: Kafka/RabbitMQ, Kinesis ordering | `backend/kafka-vs-rabbitmq.md`, `backend/kinesis-ordering.md` |
| 12 | Testing, CI/CD, observability | `miscellaneous/writing-tests.md`, `miscellaneous/cicd-and-observability.md` |
| 13 | Algorithms | `miscellaneous/coding-interview.md` |

Điểm cần nhớ: đừng ôn theo kiểu "service X dùng để làm gì". Ôn theo scenario và theo chuỗi nhân quả.

## 3. Trả lời theo trade-off, không theo đáp án tuyệt đối

Cấu trúc trả lời đáng tin gồm ba phần: nêu phương án A kèm lợi ích và chi phí, nêu phương án B tương tự, rồi chốt — với yêu cầu cụ thể này thì chọn phương án nào và vì sao. Phần chốt là thứ phân biệt người có quan điểm với người chỉ liệt kê kiến thức.

Ví dụ câu "SQL hay NoSQL?":

- Domain có quan hệ rõ ràng và cần transaction (order, payment, inventory) → PostgreSQL.
- Truy cập key-value khối lượng lớn, schema linh hoạt, các aggregate độc lập → DynamoDB có thể hợp lý hơn.

Câu trả lời yếu là "SQL tốt hơn vì transaction mạnh", vì nó bỏ qua bối cảnh. Câu trả lời mạnh nêu điều kiện để lựa chọn đúng và cái giá phải trả.

Áp dụng tương tự cho: monolith vs microservices, SQS vs Kafka, cache ở đâu, SSR hay CSR, optimistic hay pessimistic locking.

## 4. Khác biệt Middle và Senior: bắt đầu từ vấn đề

Với câu "API đang chậm, bạn làm gì?":

- **Junior**: optimize code.
- **Middle**: kiểm tra query, thêm index, thêm Redis cache. Đây là câu trả lời đúng nhưng vội.
- **Senior**: hỏi trước — endpoint nào, P50 hay P99, traffic bao nhiêu, latency nằm ở application/DB/dependency, CPU và connection thế nào, có regression sau deploy không. Sau đó mới chọn biện pháp.

Tương tự với "dùng Kafka nhé": Senior hỏi cần Kafka vì yêu cầu nào — throughput, ordering, replay, retention? Nếu chỉ cần xử lý background email, SQS đơn giản hơn nhiều.

Khung tư duy cần thể hiện:

```text
Vấn đề -> ràng buộc -> số đo -> trade-off -> giải pháp
```

Không bắt đầu bằng giải pháp.

## 5. Mười ba câu phải trả lời thật chắc

1. Event loop của Node.js hoạt động thế nào?
2. Node.js xử lý concurrency ra sao khi JavaScript single-thread?
3. Xử lý workload CPU-bound trong Node thế nào?
4. Index và composite index hoạt động thế nào?
5. Isolation level và các concurrency anomaly.
6. Làm sao tránh overselling?
7. Thiết kế retry và idempotency.
8. Transactional Outbox giải quyết vấn đề gì?
9. SQS, SNS, EventBridge, Kinesis khác nhau ở đâu?
10. Lambda concurrency ảnh hưởng tới RDS thế nào?
11. React re-render và memoization.
12. Server/Client Component và chiến lược rendering của Next.js.
13. Thiết kế hệ thống order/payment có consistency và xử lý failure.

Mục tiêu là nói được 5–10 phút cho mỗi câu, có ví dụ và trade-off, không phải trả lời thuộc lòng.

## 6. Cách luyện và những câu đào sâu thường gặp

Cách luyện hiệu quả: tự nói thành tiếng, có người nghe hoặc ghi âm, cho từng câu trong danh sách trên. Sau đó tự hỏi tiếp:

- Why?
- What if traffic increases 10x?
- What happens if it fails here?
- What are the alternatives?
- Why not X?
- How would you monitor it?
- How would you migrate to this architecture?

Nếu trả lời được các câu đào sâu này một cách bình tĩnh, nền tảng đã sát mức Middle+/Senior.

Một bài luyện tổng hợp đáng làm: đưa ra một sự cố giả định và kể lại chuỗi xử lý.

```text
Traffic tăng 20 lần
-> DB timeout
-> Lambda retry
-> một số order bị tạo hai lần
-> payment đã thành công nhưng order vẫn pending

Bạn debug và thiết kế lại thế nào?
```

Trả lời tốt là nối được: concurrency → connection pool → timeout → retry → duplicate → idempotency → outbox → queue/DLQ → observability, và nêu rõ bước nào xử lý trước để giảm ảnh hưởng người dùng.

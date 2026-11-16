# Knowledge Base — Fullstack Middle–Senior

Tổng hợp kiến thức phục vụ ôn phỏng vấn và tra cứu khi làm việc, với stack Node.js/NestJS + SQL + React/Next.js + AWS. Nội dung được viết lại theo hướng giải thích và trade-off, không phải bản ghi câu trả lời.

## 1. Cấu trúc

```text
backend/          runtime, framework, kiến trúc, hạ tầng, bảo mật, system design
database/         index, truy vấn, transaction, concurrency
frontend/         React, Next.js
miscellaneous/    viết test, CI/CD, observability, coding interview
soft-skills/      chuẩn bị phỏng vấn, behavioral
```

## 2. Backend

| File | Nội dung |
|---|---|
| [nodejs-runtime.md](backend/nodejs-runtime.md) | Event loop, concurrency, nextTick/microtask, memory leak, worker/queue, chunking |
| [nestjs-architecture.md](backend/nestjs-architecture.md) | Phân lớp, request pipeline, business logic trong controller, DI, provider scope |
| [modular-monolith.md](backend/modular-monolith.md) | Module boundary, giao tiếp giữa module, transaction xuyên module, khi nào tách service |
| [microservices.md](backend/microservices.md) | Chia service, database per service, delivery semantics, ordering, resilience, versioning, case study checkout |
| [bounded-context.md](backend/bounded-context.md) | DDD bounded context, ubiquitous language, event storming, context map |
| [distributed-systems.md](backend/distributed-systems.md) | Idempotency, retry, circuit breaker, transactional outbox, saga |
| [kinesis-ordering.md](backend/kinesis-ordering.md) | Kinesis shard/partition key, entity vs transaction ordering, DMS, batch failure |
| [kafka-vs-rabbitmq.md](backend/kafka-vs-rabbitmq.md) | Log vs queue, consumer group, routing, replay, retry/DLQ, khi nào chọn cái nào |
| [aws-serverless.md](backend/aws-serverless.md) | Lambda cold start, Lambda + RDS, SQS/SNS/EventBridge/Kinesis, FIFO, DLQ, CDK |
| [security.md](backend/security.md) | IAM, JWT storage, CORS, AuthN/AuthZ, RBAC/ABAC, lỗ hổng web |
| [system-design.md](backend/system-design.md) | Quy trình thiết kế, hệ thống đặt hàng, debug latency, xử lý sự cố production |

## 3. Database

| File | Nội dung |
|---|---|
| [index-and-query-optimization.md](database/index-and-query-optimization.md) | Index, composite/left-most prefix, EXPLAIN ANALYZE, phân loại index, pagination |
| [transactions-and-concurrency.md](database/transactions-and-concurrency.md) | Isolation level, anomaly, atomic/pessimistic/optimistic locking, chống ghi trùng |
| [normalization.md](database/normalization.md) | 1NF–5NF/BCNF, functional dependency, denormalization và trade-off, chuẩn hóa tới đâu |

## 4. Frontend

| File | Nội dung |
|---|---|
| [react-rendering.md](frontend/react-rendering.md) | Re-render, memoization, useEffect, race condition, Context vs state store |
| [nextjs-rendering-and-caching.md](frontend/nextjs-rendering-and-caching.md) | Server/Client Component, CSR/SSR/SSG/ISR, hydration, các tầng cache |

## 5. Miscellaneous

| File | Nội dung |
|---|---|
| [writing-tests.md](miscellaneous/writing-tests.md) | Unit/Integration/E2E, test pyramid, kỹ thuật chọn test case, mock, test data |
| [cicd-and-observability.md](miscellaneous/cicd-and-observability.md) | Pipeline, chiến lược deploy, logs/metrics/traces |
| [coding-interview.md](miscellaneous/coding-interview.md) | Algorithm, backend/frontend coding, tiêu chí chấm |

## 6. Soft skills

| File | Nội dung |
|---|---|
| [interview-preparation.md](soft-skills/interview-preparation.md) | Phạm vi đánh giá, thứ tự ưu tiên ôn, trả lời theo trade-off, 13 câu trọng tâm |
| [behavioral-and-leadership.md](soft-skills/behavioral-and-leadership.md) | Bộ câu chuyện, STAR, câu hỏi về quyết định sai |

## 7. Quy ước

- Mỗi file có một tiêu đề `#` và các mục `##` đánh số lại từ 1.
- Mỗi mục chứa trọn một câu hỏi/chủ đề và phần giải thích tương ứng.
- Thuật ngữ chuyên môn giữ nguyên tiếng Anh; phần diễn giải dùng tiếng Việt.
- Tên file dùng tiếng Anh, kebab-case; nội dung không lặp giữa các file, chỉ tham chiếu chéo.

## 8. Lộ trình đọc gợi ý

- **Ôn phỏng vấn gấp**: `soft-skills/interview-preparation.md` → `backend/system-design.md` → `database/` → `backend/distributed-systems.md`.
- **Backend hằng ngày**: `backend/nodejs-runtime.md`, `backend/nestjs-architecture.md`, `database/index-and-query-optimization.md`.
- **Thiết kế hệ thống mới**: `backend/bounded-context.md` → `backend/modular-monolith.md` → `backend/microservices.md` → `backend/kafka-vs-rabbitmq.md`.
- **Vận hành và sự cố**: `backend/system-design.md`, `miscellaneous/cicd-and-observability.md`.

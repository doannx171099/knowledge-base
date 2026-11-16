# Modular Monolith

## 1. Modular Monolith khác monolith truyền thống ở đâu?

Monolith truyền thống thường tổ chức theo tầng kỹ thuật:

```text
controllers/
services/
repositories/
entities/
```

Mọi domain dùng chung service, entity, repository. Sau vài năm, code không còn ranh giới nào và trở thành **Big Ball of Mud**: sửa một chỗ không biết cái gì sẽ hỏng.

Modular Monolith vẫn là **một application, một deployment unit**, nhưng bên trong chia theo business capability, mỗi module có domain, application và infrastructure riêng:

```text
Application
├── order/
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── api/
├── payment/
└── inventory/
```

Điểm khác biệt không nằm ở số process mà ở chỗ **module có boundary và public API rõ ràng**: module khác chỉ được gọi qua contract, không được chạm vào repository, entity hay service nội bộ.

Cách diễn đạt gọn khi phỏng vấn: Modular Monolith là kiến trúc monolith trong đó hệ thống được chia thành các business module độc lập về logic, với boundary và public API rõ ràng, hạn chế truy cập trực tiếp vào implementation và dữ liệu của nhau.

## 2. So sánh với microservices

| Tiêu chí | Modular Monolith | Microservices |
|---|---|---|
| Deployment | Một application | Nhiều service độc lập |
| Process | Thường cùng process | Process riêng |
| Giao tiếp | In-process | HTTP/gRPC/message |
| Database | Có thể chung cluster, tách schema | Thường database riêng |
| Transaction | ACID dễ dàng | Distributed transaction khó |
| Scaling | Scale toàn application | Scale từng service |
| Failure isolation | Thấp hơn | Cao hơn |
| Chi phí vận hành | Thấp | Cao |

Điểm cần nhấn mạnh: Modular Monolith **không phải "microservices nhưng deploy chung"**. Boundary của nó được enforce ở **code level**, còn microservices enforce thêm ở **network, process, deployment và database**. Vì vậy modular monolith dễ vi phạm boundary hơn nếu không có kỷ luật và công cụ kiểm tra.

## 3. Xác định module boundary thế nào?

Hai tiêu chí kinh điển:

- **High cohesion**: những thứ cùng một capability nằm cùng module. Ví dụ module payment chứa create payment, refund, payment status, transaction.
- **Low coupling**: module không phụ thuộc chéo vào nhiều module khác. Khi dependency tạo thành mạng nhện (`Order → Payment`, `Payment → Order`, `Order → Inventory`, `Inventory → Order`), boundary đã sai.

Cách chia đúng là theo **business capability**, không theo technical layer và không theo bảng dữ liệu:

```text
order/  payment/  inventory/  shipping/
```

thay vì

```text
controllers/  services/  repositories/
```

Chia theo bảng (`Product`, `User`, `Order`) cũng thường sai, vì một "Product" ngoài đời có thể thuộc catalog (mô tả, ảnh), pricing (giá, khuyến mãi) và inventory (tồn kho) — ba module với model và rule khác nhau. Chi tiết cách tìm boundary nằm ở `backend/bounded-context.md`.

## 4. Các module giao tiếp với nhau thế nào?

Hai kiểu, và hệ thống thực tế thường dùng cả hai:

**Đồng bộ** — gọi trực tiếp qua interface:

```text
Order → Inventory.reserve()
```

Ưu: đơn giản, có kết quả ngay, dễ đặt transaction. Nhược: coupling runtime, dễ tạo chuỗi phụ thuộc dài.

**Bất đồng bộ** — phát event qua event bus nội bộ:

```text
OrderPlaced → Inventory, Payment
```

Ưu: giảm coupling, dễ tách thành service sau này. Nhược: eventual consistency, debug khó hơn, khó đảm bảo thứ tự.

Nguyên tắc: dùng đồng bộ khi cần kết quả để ra quyết định ngay trong request; dùng event cho side effect (email, analytics, audit, thông báo) hoặc khi muốn chuẩn bị ranh giới cho việc tách service.

## 5. Domain Event và Integration Event khác nhau ở đâu?

- **Domain event**: dùng nội bộ trong module/application, mô tả điều gì đã xảy ra trong domain (`OrderPlaced`, `PaymentCompleted`). Có thể mang theo entity hoặc cấu trúc nội bộ.
- **Integration event**: contract công khai giữa các module hoặc với hệ thống ngoài (`PaymentCompletedIntegrationEvent`). Phải ổn định, có version, chỉ chứa dữ liệu cần thiết.

Sai lầm phổ biến là expose thẳng domain object làm event contract; khi domain thay đổi, mọi consumer vỡ theo. Integration event nên được thiết kế như một API: additive change, có version, không đổi kiểu dữ liệu tùy tiện.

## 6. Database: dùng chung được không, có được JOIN giữa module?

Dùng chung database **được**, nhưng phải có **data ownership**:

```text
order module    owns orders, order_items
payment module  owns payments, payment_transactions
inventory       owns stock, reservations
```

Ngay cả khi cùng một PostgreSQL, có thể tách schema (`order`, `payment`, `inventory`). Điều quan trọng là module A không `SELECT` thẳng bảng của module B — vì như vậy module B không thể đổi schema mà không phá A.

JOIN giữa bảng của hai module: về kỹ thuật là được, về kiến trúc nên hạn chế. JOIN tạo coupling ở tầng database, khiến việc tách module thành service sau này gần như phải viết lại. Thay thế tùy use case:

- Gọi API của module kia khi cần dữ liệu mới nhất.
- Nghe event và giữ read model / materialized view cho truy vấn đọc.
- Chấp nhận duplicate dữ liệu có chủ đích (xem mục 7).

## 7. Duplicate dữ liệu có phải lúc nào cũng xấu?

Không. Có hai loại duplicate rất khác nhau:

- **Duplicate để tối ưu truy vấn** (read model, cache, materialized view): chấp nhận được nếu xác định rõ nguồn sự thật và cơ chế cập nhật.
- **Snapshot dữ liệu nghiệp vụ**: cần thiết. Order lưu `shippingAddress` tại thời điểm đặt hàng thay vì chỉ `addressId`, vì nếu người dùng đổi địa chỉ sau ba tháng, đơn cũ không được thay đổi theo. Đây là snapshot có chủ đích, không phải dữ liệu thừa.

Điều bắt buộc là trả lời được: **đâu là source of truth cho từng trường**, và nếu lệch thì bên nào thắng.

## 8. Transaction xuyên module: lợi thế và cái bẫy

Đây là lợi thế lớn nhất của modular monolith. Trong cùng một database, có thể:

```sql
BEGIN;
INSERT INTO orders ...;
UPDATE inventory SET stock = stock - 1 ...;
INSERT INTO payments ...;
COMMIT;
```

và nhận ACID mà microservices phải đánh đổi bằng saga, outbox, compensation.

Nhưng có cái bẫy: transaction đúng nhưng kiến trúc sai nếu code gọi thẳng repository của module khác:

```typescript
@Transactional()
createOrder() {
  this.orderRepository.save(...);
  this.inventoryRepository.reserve(...);   // sai boundary
  this.paymentRepository.create(...);
}
```

Cách đúng là vẫn đi qua public interface của từng module, dù tất cả nằm trong một transaction:

```typescript
@Transactional()
createOrder() {
  this.order.create();
  this.inventory.reserve();
  this.payment.create();
}
```

Trade-off cần nêu: **strong consistency đổi lấy module autonomy**. Nếu một luồng cần transaction trải qua quá nhiều module, đó có thể là dấu hiệu boundary bị cắt sai — không phải lý do để gộp mọi thứ vào một chỗ.

## 9. Chiều phụ thuộc, circular dependency và shared kernel

Chiều phụ thuộc nên là:

```text
api → application → domain
infrastructure → domain (implement interface)
```

Domain không import framework, database, message broker hay SDK cloud. Nếu entity cần gọi service, đó là dấu hiệu logic đang đặt sai chỗ.

Circular dependency giữa hai module (`Order → Payment`, `Payment → Order`) nên được phá bằng event: `Order` phát `OrderPlaced`, `Payment` phát `PaymentCompleted`, thay vì hai service gọi nhau. Trong NestJS, `forwardRef()` chỉ vá lỗi DI, không sửa được kiến trúc.

**Shared kernel** là vùng nguy hiểm. Ban đầu là `shared/utils`, vài năm sau thành `shared/everything`, kéo modular monolith trở lại thành big ball of mud. Chỉ nên chứa thứ thực sự generic: `Money`, `DomainEvent`, `Result`, `Clock`, abstraction cho ID. Không đặt `OrderEntity`, `PaymentRepository`, `CustomerService` trong shared.

## 10. Làm sao enforce boundary, không chỉ dựa vào quy ước?

Nói "developer không được import module khác" là chưa đủ. Có thể enforce bằng công cụ:

- **Node.js**: tách package/workspace cho từng module, dùng ESLint `import/no-restricted-paths` hoặc Nx module boundaries để chặn import chéo.
- **.NET**: tách project/assembly, dùng NetArchTest.
- **Java**: package visibility, ArchUnit, Java Module System, multi-module Maven/Gradle.

Cấu trúc build:

```text
application
├── order-module
├── payment-module
├── inventory-module
└── bootstrap
```

Khi boundary được kiểm tra ở compile/build time, vi phạm bị chặn ngay thay vì phát hiện sau nhiều tháng review.

## 11. Testing, deployment và failure isolation

**Testing**: có thể chia unit → module integration → application integration → E2E. Lợi thế lớn là integration test không cần dựng container cho từng service, Kafka cluster hay service discovery; chỉ cần database và các dependency thật sự liên quan.

**Deployment**: vẫn là một artifact (jar hoặc Docker image). Hạn chế là scale toàn bộ application: nếu order cần scale gấp 10 lần còn payment thì không, bạn vẫn phải nhân bản cả app.

**Failure isolation**: một memory leak ở payment có thể làm sập cả application, kéo order và inventory theo. Đây là trade-off thật so với microservices, cần nêu khi so sánh thay vì chỉ nói monolith "đơn giản hơn".

## 12. Khi nào chọn Modular Monolith, khi nào tách Microservices?

Chọn modular monolith khi domain đủ phức tạp để cần boundary nhưng chưa có nhu cầu vận hành độc lập: team khoảng 5-10 người, sản phẩm còn thay đổi nhanh, traffic vừa phải, deployment một lần/ngày vẫn ổn.

Chọn microservices khi có lý do cụ thể, không phải "hệ thống lớn":

- Một module cần scale độc lập (search chiếm phần lớn traffic và cần CPU/Elasticsearch riêng).
- Một module cần nhịp release khác hẳn.
- Nhiều team sở hữu các phần khác nhau, cần ranh giới tổ chức rõ.
- Cần cô lập lỗi giữa các phần.
- Yêu cầu công nghệ khác nhau cho từng phần.

Không nhất thiết tách toàn bộ: có thể giữ modular monolith cho order/payment/customer và tách riêng search thành một service. Đây thường là bước chuyển tiếp hợp lý hơn cả "big bang rewrite".

## 13. Scenario phỏng vấn: thiết kế checkout trong Modular Monolith

Yêu cầu: user đặt hàng, cần reserve inventory và thanh toán.

Trình bày theo hai phương án và trade-off:

**Đồng bộ** — phù hợp khi cần strong consistency:

```text
CreateOrder
  → Inventory.reserve()
  → Payment.authorize()
  → Order.confirm()
```

Có thể dùng một transaction nếu cả ba module chung database, nhưng vẫn gọi qua interface.

**Bất đồng bộ** — phù hợp khi muốn giảm coupling và chuẩn bị tách service:

```text
OrderCreated → Inventory, Payment
InventoryReserved / PaymentFailed → Order
```

Kèm theo: eventual consistency, retry, idempotency, và cách hiển thị trạng thái trung gian cho người dùng.

Câu trả lời được đánh giá cao là nêu rõ: "Nếu yêu cầu nghiệp vụ là strong consistency thì tôi ưu tiên orchestration đồng bộ; nếu ưu tiên autonomy và chuẩn bị tách service thì dùng event + eventual consistency", thay vì chọn một cách máy móc.

## 14. Mười câu hỏi thường gặp về Modular Monolith

1. Modular Monolith khác monolith truyền thống ở đâu?
2. Nó khác microservices ở điểm nào?
3. Xác định module boundary bằng cách nào?
4. Hai module nên giao tiếp ra sao?
5. Module có được truy cập database của module khác không?
6. Làm sao tránh circular dependency?
7. Transaction trải qua nhiều module xử lý thế nào?
8. Domain event khác integration event thế nào?
9. Làm sao enforce boundary ở mức code?
10. Khi nào quyết định tách một module thành microservice?

# Bounded Context (DDD)

## 1. Bounded Context là gì?

Bounded Context là **ranh giới mà bên trong đó domain model, thuật ngữ và business rule có ý nghĩa nhất quán**. Nó là boundary về ngữ nghĩa, không phải boundary vật lý hay deployment.

Hệ quả: không có một model chung cho toàn hệ thống. Cùng một khái niệm ngoài đời có thể có model khác nhau ở từng context, miễn là trong mỗi context nó nhất quán.

Nguyên tắc nhận diện: **nếu cùng một khái niệm có ý nghĩa hoặc quy tắc khác nhau ở hai khu vực nghiệp vụ, rất có thể chúng thuộc hai bounded context khác nhau.**

Cần phân biệt với vài khái niệm hay bị gộp:

- **Subdomain**: phân vùng của business (core/supporting/generic), là vấn đề của bài toán.
- **Bounded Context**: ranh giới của model và ngôn ngữ, là vấn đề của giải pháp.
- **Module**: đơn vị tổ chức code trong một context.
- **Microservice**: đơn vị deploy; một bounded context có thể là một module, một service, hoặc nhiều service.

## 2. Bắt đầu từ business capability

Với hệ thống thương mại điện tử, các capability tự nhiên:

```text
Catalog, Pricing, Inventory, Ordering, Payment, Shipping
```

Mỗi capability là ứng viên cho một context, với model riêng:

```text
Catalog   → Product, Category, ProductDescription
Pricing   → ProductPrice, Discount, Promotion
Inventory → StockItem, Warehouse, Reservation
Ordering  → Order, OrderItem, Customer
Payment   → Payment, Refund, Transaction
```

Điều quan trọng: đây là các context nói về cùng một sản phẩm ngoài đời nhưng không chia sẻ model.

## 3. Một entity ngoài đời, nhiều model

Ví dụ `Product`:

- **Catalog** quan tâm: tên, mô tả, ảnh, category, SEO.
- **Inventory** quan tâm: SKU, kho, số lượng khả dụng, số lượng đã giữ.
- **Pricing** quan tâm: giá gốc, giảm giá, khuyến mãi.

Cố gộp thành một class `Product` chứa tất cả (giá, tồn kho, kho, nhà cung cấp, đánh giá, vận chuyển) thường tạo ra **God Entity**: mọi thay đổi nghiệp vụ đều chạm vào nó, mọi team đều phải sửa, và không ai hiểu hết invariant của nó.

## 4. Tìm khác biệt về ngôn ngữ

DDD gọi đây là **Ubiquitous Language**: mỗi context có ngôn ngữ riêng, dùng nhất quán trong code, tài liệu và giao tiếp với chuyên gia nghiệp vụ.

Ví dụ từ `Customer`:

- Sales: lead source, segment, account manager.
- Billing: billing address, tax number, payment terms.
- Support: ticket, SLA, support tier.

Ngoài đời là cùng một người, nhưng business nói về họ theo cách khác. Tách thành `SalesCustomer`, `BillingAccount`, `SupportCustomer` thường đúng hơn là ép về một model chung.

## 5. Tìm khác biệt về business rule và invariant

Đây là tiêu chí mạnh hơn cả entity. Ví dụ ba bộ rule hoàn toàn khác nhau:

- Ordering: order chỉ confirm khi có ít nhất một item, địa chỉ hợp lệ, tổng tiền > 0.
- Inventory: chỉ reserve khi `availableQuantity >= requestedQuantity`.
- Payment: không capture nếu authorization chưa thành công.

Ba nhóm invariant khác nhau → ba context.

## 6. Tìm transaction boundary

Câu hỏi hữu ích: **rule nào bắt buộc phải nhất quán tức thời với nhau?**

Trong Order, `total = sum(quantity * price)` là invariant cần giữ cùng nhau. Nhưng Order, Stock, Payment, Shipment không nhất thiết phải nằm trong cùng transaction; có thể dùng event, saga hoặc process manager.

Nếu bạn phải dựng distributed transaction cho nhiều capability, có khả năng boundary đang sai — hoặc consistency requirement chưa được xác định rõ.

## 7. Ownership của dữ liệu và của quyết định

**Ai quyết định sự thật của dữ liệu này?** Inventory sở hữu available/reserved stock; Ordering có thể hỏi "còn hàng không" nhưng không được tự đọc bảng stock rồi tự diễn giải.

**Ai có quyền ra quyết định nghiệp vụ?** Ví dụ "khách này được giảm 20% hay không" thuộc Pricing. Nếu Ordering tự viết `if (customer.vip) discount = 20`, rule sẽ bị nhân bản ở checkout, admin, báo giá — và lệch nhau theo thời gian.

## 8. Tốc độ thay đổi và organizational boundary

Hai phần thay đổi độc lập, vì lý do khác nhau, thường nên tách: Pricing thay đổi liên tục (campaign, voucher, dynamic pricing) trong khi Inventory thay đổi theo kho, nhập hàng, phân bổ. Hai team phát triển độc lập là tín hiệu boundary tốt (Conway's Law), dù không có quy tắc "mỗi team = một context".

## 9. Event Storming: cách tìm boundary thực tế nhất

Liệt kê các event theo trình tự nghiệp vụ:

```text
OrderPlaced → StockReserved → PaymentAuthorized → OrderConfirmed → ShipmentCreated → ShipmentDelivered
```

Rồi nhóm theo cụm nghiệp vụ:

- Ordering: `OrderCreated`, `OrderConfirmed`, `OrderCancelled`.
- Inventory: `StockReserved`, `StockReservationFailed`, `StockReleased`.
- Payment: `PaymentAuthorized`, `PaymentCaptured`, `PaymentFailed`, `PaymentRefunded`.
- Shipping: `ShipmentCreated`, `ShipmentDispatched`, `ShipmentDelivered`.

Các cụm này thường trở thành bounded context. Event Storming hiệu quả vì nó dựa trên ngôn ngữ nghiệp vụ thay vì cấu trúc dữ liệu hiện có.

## 10. Không chia theo CRUD, không chia theo technical layer

Chia theo bảng dữ liệu:

```text
User Service, Product Service, Order Service
```

chỉ vì database có ba bảng — đây là entity decomposition, chưa phải domain decomposition. Một `Product` gồm thông tin, giá, tồn kho, khuyến mãi, đánh giá thực chất thuộc bốn context khác nhau.

Chia theo technical layer (`RepositoryContext`, `NotificationContext`, `DatabaseContext`) còn sai hơn: đó là concern kỹ thuật, không phải business capability.

## 11. Bounded Context không đồng nghĩa với Microservice

Đây là câu hỏi phỏng vấn rất hay. Một bounded context có thể được triển khai như:

- Một module trong modular monolith.
- Một hoặc vài microservice.
- Một service dùng chung cho nhiều context nhỏ (generic subdomain).

Ví dụ modular monolith với các package `catalog/`, `pricing/`, `ordering/`, `inventory/`, `payment/` vẫn giữ domain boundary rõ, dù chỉ một deployment unit. Sau này khi cần scale độc lập mới tách payment hoặc inventory ra.

## 12. Giao tiếp qua contract và Context Map

Giữa các context, giao tiếp qua contract, không qua repository hay entity nội bộ:

```typescript
interface InventoryGateway {
  reserveStock(productId: ProductId, quantity: number): Promise<ReservationResult>;
}
```

Hoặc qua command (`ReserveStock`) và event (`StockReserved`, `StockReservationFailed`).

**Context Map** mô tả quan hệ giữa các context: Customer/Supplier, Conformist, Anti-Corruption Layer, Shared Kernel, Open Host Service, Published Language, Partnership.

Ví dụ khi tích hợp payment provider bên ngoài, đặt một **Anti-Corruption Layer**:

```typescript
interface PaymentGateway { authorize(); capture(); refund(); }
class StripePaymentGateway implements PaymentGateway { ... }
```

Nhờ vậy domain không phụ thuộc trực tiếp vào model của Stripe, và có thể đổi provider mà không lan thay đổi ra toàn hệ thống.

## 13. Checklist xác định Bounded Context

| Câu hỏi | Nếu "có" |
|---|---|
| Cùng thuật ngữ nhưng nghĩa khác nhau? | Có thể tách |
| Business rule khác nhau? | Có thể tách |
| Invariant khác nhau? | Có thể tách |
| Dữ liệu có owner khác nhau? | Có thể tách |
| Người ra quyết định khác nhau? | Có thể tách |
| Hai phần thay đổi độc lập? | Có thể tách |
| Team ownership khác nhau? | Có thể tách |
| Chấp nhận eventual consistency giữa chúng? | Khả năng cao nên tách |

Câu hỏi tổng hợp mạnh nhất: **nếu tôi đổi business rule ở A, tôi có thường xuyên phải đổi model của B không?** Nếu gần như không → hai context. Nếu luôn phải đổi cùng nhau → có thể chúng thực chất là một.

## 14. Điều cần nhớ khi phỏng vấn

Đừng trả lời "bounded context là chia hệ thống thành các service". Câu trả lời đúng:

> Bounded Context là boundary về ngữ nghĩa và domain model. Bên trong boundary, ubiquitous language và business rule nhất quán. Boundary thường được tìm dựa trên business capability, khác biệt về ngôn ngữ/model, ownership của rule và dữ liệu, consistency requirement và tốc độ thay đổi. Bounded Context là logical boundary; microservice là deployment boundary. Một context có thể là module trong modular monolith hoặc một microservice.

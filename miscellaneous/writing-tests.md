# Viết Test

## 1. Unit, Integration và E2E khác nhau ở đâu?

Ba loại khác nhau chủ yếu ở **phạm vi**, **mức độ dùng dependency thật**, tốc độ và độ tin cậy.

| Tiêu chí | Unit | Integration | E2E |
|---|---|---|---|
| Kiểm tra | Một đơn vị logic | Nhiều component + boundary | Toàn bộ luồng qua public interface |
| Ví dụ | `OrderService.calculateTotal()` | Service + PostgreSQL thật | `POST /orders` → auth → controller → service → DB |
| Database | Mock | Thật (container) | Thật |
| HTTP server | Không | Có thể | Có |
| Tốc độ | Rất nhanh | Trung bình | Chậm |
| Số lượng nên có | Nhiều nhất | Vừa | Ít |

**Unit test** kiểm tra logic trong cô lập, dependency được mock. Giá trị: chạy hàng nghìn case nhanh, khoanh vùng lỗi chính xác. Rủi ro: mock sai khiến test pass nhưng production fail.

**Integration test** kiểm tra các component làm việc với nhau: repository + database thật, service + Redis, producer + broker. Đây là nơi bắt được lỗi mà unit test che mất: schema `VARCHAR(50)` trong khi application nhận chuỗi 100 ký tự, foreign key, unique constraint, transaction rollback, mapping ORM, serialization.

**E2E test** chạy một luồng hoàn chỉnh giống người dùng thật, qua public interface. Giá trị cao nhất nhưng chậm, dễ flaky và debug khó.

Cách phân biệt dễ nhớ:

- Unit: "nếu logic của tôi sai thì sao?"
- Integration: "nếu các component không khớp nhau thì sao?"
- E2E: "nếu hệ thống không chạy được từ góc nhìn người dùng thì sao?"

## 2. Hai ranh giới dễ nhầm

**Integration vs E2E.** Không phải "integration test database, E2E test API". Khác biệt là **boundary**: integration kiểm tra một nhóm component tương tác (ví dụ repository với DB thật), E2E kiểm tra toàn bộ luồng qua public interface. Gọi `orderService.createOrder()` với DB thật vẫn là integration; gửi `POST /orders` qua HTTP đi hết hệ thống thường là E2E.

**E2E có được mock không?** Có. Với hệ thống thanh toán thật, không ai muốn mỗi lần CI tạo một giao dịch Stripe thật. E2E hợp lý vẫn mock boundary bên ngoài hệ thống của mình (payment provider, email), miễn là toàn bộ hệ thống bên trong được chạy thật. E2E không đồng nghĩa "không mock bất cứ thứ gì".

Với external service ở mức integration, có thể dùng mock HTTP server (WireMock) để kiểm tra đúng format request, timeout handling, mapping response — những thứ mock ở mức class không kiểm tra được.

## 3. Chiến lược: test pyramid

```text
        E2E
      Integration
    Unit
```

Ý tưởng: nhiều unit, vừa integration, ít E2E, vì chi phí tăng dần theo tốc độ, độ flaky và độ khó debug. Không có tỷ lệ bắt buộc kiểu 70/20/10; tỷ lệ phụ thuộc kiến trúc và mức rủi ro.

Điều quan trọng hơn số lượng là **đặt test ở tầng rẻ nhất có thể bắt được lỗi**:

- Business rule nhiều nhánh → unit.
- Câu SQL, constraint, transaction → integration.
- Luồng tạo doanh thu (đăng ký, checkout, refund) → E2E, số lượng nhỏ.

Unit test với repository mock chỉ chứng minh service gọi đúng hàm; nó không chứng minh câu SQL đúng. Với query phức tạp, một integration test có giá trị hơn mười unit test mock.

## 4. Test behavior, không test implementation

Đây là nguyên tắc quan trọng nhất.

```typescript
// yếu: phụ thuộc implementation
expect(repository.findById).toHaveBeenCalledTimes(1);
expect(repository.save).toHaveBeenCalledTimes(1);
```

Test này sẽ fail khi refactor sang `repository.cancelById()` dù behavior không đổi. Nó đang kiểm tra "code viết thế nào", không phải "hệ thống làm gì".

Nên assert outcome nghiệp vụ:

```typescript
const order = await service.cancelOrder('order-1');
expect(order.status).toBe(OrderStatus.CANCELLED);
```

Kiểm tra side effect quan sát được (đã ghi DB, đã phát event, đã trả lỗi đúng) thay vì đếm số lần gọi nội bộ. Ngoại lệ hợp lý: khi việc "chỉ gọi một lần" chính là yêu cầu (không charge hai lần, không gửi email trùng).

## 5. Cấu trúc một test tốt

**AAA (Arrange – Act – Assert)** hoặc **Given – When – Then**: cùng một ý. Chuẩn bị state, thực hiện hành động, kiểm tra kết quả. Không trộn ba phần với nhau.

**Một test một behavior.** Không nhất thiết chỉ một `expect()`, nhưng các assertion phải cùng phục vụ một behavior. Ba assertion `status = PAID`, `paidAt != null`, `remainingBalance = 0` vẫn là một behavior. Nhưng `should create order, charge payment, send email and cancel` là quá nhiều — khi fail không biết phần nào hỏng.

**Tên test thể hiện business behavior.** Pattern `should [kết quả] when [điều kiện]`:

```text
should reject creating an order when user already has a pending order
should apply 20% discount for premium customers
should expire token when expiration time has passed
```

Khi CI fail, đọc tên test là hiểu vấn đề, không cần mở code.

## 6. Chọn test case bằng kỹ thuật, không bằng cảm giác

**Happy path + edge + failure**: đừng chỉ test đường thành công. Ví dụ `calculateDiscount`: order = 0, discount 0%, 100%, >100%, giá âm, làm tròn.

**Boundary Value Analysis**: với ngưỡng `1,000,000–4,999,999 → 5%`, test `999,999`, `1,000,000`, `1,000,001`, `4,999,999`, `5,000,000`, `5,000,001`. Hiệu quả hơn nhiều so với test vài giá trị ngẫu nhiên ở giữa khoảng.

**Equivalence Partitioning**: input cùng behavior thì chỉ cần một đại diện. Với `age < 18`, `18–59`, `>= 60`, test `10`, `30`, `70` cộng các giá trị biên.

**State transition**: với order `CREATED → PAID → SHIPPED → DELIVERED` và `CREATED → CANCELLED`, `PAID → REFUNDED`, phải test cả transition hợp lệ và **không hợp lệ** (`SHIPPED → CANCELLED` phải bị chặn). Kỹ thuật này rất quan trọng với order, payment, subscription, workflow, ticket, vòng đời tài khoản.

**Decision table**: khi kết quả phụ thuộc tổ hợp nhiều điều kiện (loại khách, số tiền, kênh thanh toán), liệt kê tổ hợp và expected result thành bảng, tránh bỏ sót nhánh.

**Pairwise**: khi có nhiều tham số, test mọi cặp giá trị thay vì tích Descartes; giảm số case mạnh mà vẫn bắt được lỗi tương tác phổ biến.

**Property-based testing**: thay vì chọn input, khai báo invariant luôn đúng (ví dụ "tổng tiền không âm", "encode rồi decode trả lại dữ liệu gốc") và để công cụ sinh hàng trăm input. Hữu ích cho parser, tính toán, serialize.

## 7. Mock đúng cách

- **Không mock thứ đang test.** Đang test `OrderService` mà spy chính `calculateTotal` của nó rồi kết luận "works" là vô nghĩa.
- **Chỉ mock boundary kiến trúc**: database, HTTP API, message broker, filesystem, clock, random. Pure logic (policy, factory, mapper) nên dùng thật; mock quá nhiều dẫn tới test chỉ chứng minh mock hoạt động.
- **Mock phải realistic.** API thật trả `Promise<Order | null>`, mock trả `undefined` là sai. Response thật có wrapper `{ data: { status } }`, mock không có wrapper — unit pass, integration fail.
- **Đừng mock để né vấn đề thiết kế.** Nếu phải mock 7 thứ cho một hàm, có thể hàm đang làm quá nhiều.

## 8. Deterministic và độc lập

Test phải cho cùng kết quả ở mọi lần chạy, mọi thứ tự.

- **Kiểm soát thời gian**: bơm `Clock` vào service thay vì gọi `new Date()` trực tiếp. Test được "1 giây trước hết hạn", "đúng lúc hết hạn", "1 giây sau".
- **Không phụ thuộc thứ tự chạy**: chạy riêng một test bằng `-t "should cancel order"` phải pass. Nếu phải chạy test khác trước, đó là vấn đề thiết kế.
- **Không dùng shared mutable state**: `beforeAll` tạo object rồi nhiều test cùng mutate sẽ khiến chúng phụ thuộc nhau; dùng `beforeEach`.
- **Không dùng `sleep`** để chờ async; chờ theo điều kiện/tín hiệu hoặc dùng fake timer.
- **Cleanup đúng**: transaction rollback, truncate, hoặc dữ liệu unique theo test.

## 9. Test data phải thể hiện ý định

Một object 8 trường với giá trị `type: 2`, `level: 4` khiến người đọc phải suy luận. Dùng **builder/Object Mother**:

```typescript
const user = UserBuilder.aUser().premium().build();
const order = OrderBuilder.anOrder().shipped().build();
```

Chỉ rõ phần dữ liệu mà test quan tâm, phần còn lại dùng default hợp lý. Tránh magic number rải rác; đặt tên cho ngưỡng (`FREE_SHIPPING_THRESHOLD`) để test đọc như mô tả nghiệp vụ.

## 10. Test những ranh giới dễ sinh lỗi production

- **Idempotency**: gọi cùng thao tác hai lần, kết quả và side effect không đổi.
- **Duplicate message / retry**: consumer nhận cùng event hai lần chỉ xử lý một lần; retry không tạo bản ghi trùng.
- **Contract test** (microservices): consumer định nghĩa contract, provider verify trong CI (Pact), phát hiện breaking change trước khi deploy.
- **Concurrency**: khi có race condition nghiệp vụ (hai người mua món cuối), test bằng nhiều request đồng thời và assert invariant (không âm kho, chỉ một thành công).
- **Error contract**: kiểm tra status code và body lỗi, không chỉ happy path.

## 11. Coverage và mutation testing

Coverage là chỉ số **đã chạy qua dòng code**, không phải chỉ số **đã kiểm tra đúng**. 100% coverage vẫn có thể không có assertion nào. Đừng chase 100%; tập trung vào nhánh quan trọng: business rule, error path, boundary.

Branch coverage hữu ích hơn line coverage vì nó buộc mỗi nhánh `if/else` được đi qua ít nhất một lần.

**Mutation testing** (Stryker, PIT) tự sửa code (đổi `>` thành `>=`, đảo điều kiện) rồi chạy test; nếu test vẫn pass, test chưa đủ mạnh. Đây là cách đo chất lượng test tốt hơn coverage, nhưng chạy chậm nên thường chỉ dùng cho module quan trọng.

## 12. FIRST và checklist review test

**FIRST**:

- **Fast**: unit test chạy nhanh để chạy liên tục.
- **Independent**: không phụ thuộc test khác.
- **Repeatable**: cùng kết quả ở mọi môi trường.
- **Self-validating**: pass/fail rõ ràng, không cần đọc log.
- **Timely**: viết cùng lúc với code, không để cuối.

Checklist khi review test:

- Test có mô tả behavior nghiệp vụ, không phải implementation?
- Có case biên và case lỗi, hay chỉ happy path?
- Mock có thật sự cần, và có realistic?
- Test có chạy độc lập và lặp lại được?
- Assert có đủ để bắt regression, nhưng không thừa?
- Tên test có đọc hiểu khi CI fail?

Test code cũng là production code: nó được đọc, sửa và debug nhiều lần. Đầu tư vào tên, cấu trúc và dữ liệu test sẽ trả lại nhiều hơn số test tăng thêm.

# Transaction và Concurrency

## 1. Isolation level giải quyết vấn đề gì?

Isolation kiểm soát việc transaction này **nhìn thấy** thay đổi của transaction khác tới mức nào. Đây là trade-off giữa tính đúng đắn và mức độ song song.

Bốn mức phổ biến (theo SQL standard) và các anomaly tương ứng:

| Mức | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| Read Uncommitted | có thể | có thể | có thể |
| Read Committed | không | có thể | có thể |
| Repeatable Read | không | không | có thể (tuỳ DB) |
| Serializable | không | không | không |

Các anomaly cần hiểu bằng ví dụ:

- **Dirty read**: B đọc dữ liệu A chưa commit; A rollback. B vừa đọc một trạng thái chưa từng tồn tại.
- **Non-repeatable read**: trong cùng transaction, A đọc `price = 100`, B cập nhật thành 200 và commit, A đọc lại thấy 200. Hai lần đọc cùng một dòng cho hai kết quả.
- **Phantom read**: A đếm được 10 order `status = 'NEW'`; B chèn thêm một order mới; A chạy lại cùng query thấy 11. Không phải dòng cũ đổi, mà là **tập kết quả** đổi.
- **Lost update**: hai transaction cùng đọc `stock = 1`, cùng ghi `stock = 0`. Một cập nhật bị ghi đè.
- **Write skew**: hai transaction đọc cùng một tập dữ liệu, mỗi bên sửa một dòng khác nhau dựa trên điều kiện chung, kết quả vi phạm invariant. Ví dụ hai bác sĩ cùng đọc "còn 2 người trực", mỗi người xin nghỉ một người, cả hai commit → còn 0 người. `Repeatable Read` không chặn được write skew; `Serializable` thì có.

Lưu ý thực tế: PostgreSQL dùng MVCC; `Read Committed` là mặc định và mỗi statement thấy snapshot mới, nên non-repeatable read vẫn xảy ra. `Repeatable Read` trong PostgreSQL mạnh hơn standard (chặn cả phantom) nhưng có thể ném lỗi serialization. MySQL InnoDB mặc định `Repeatable Read` và dùng next-key lock để chặn phantom.

**Isolation không đồng nghĩa với locking.** Isolation là guarantee mà database cung cấp; còn MVCC, row lock, gap lock, predicate lock, optimistic concurrency control là các **cơ chế implementation** để đạt guarantee đó. PostgreSQL chủ yếu dùng MVCC nên `SELECT` thường không cần lock dòng; dữ liệu được giữ nhiều version và mỗi transaction đọc một snapshot phù hợp.

Cách trả lời đúng: không học thuộc bảng, mà nêu **yêu cầu nghiệp vụ** quyết định mức isolation, và cái giá là throughput cùng khả năng phải retry khi gặp lỗi serialization.

## 2. Hai request cùng mua sản phẩm cuối cùng: chống overselling thế nào?

Kịch bản:

```text
stock = 1
A đọc stock = 1
B đọc stock = 1
A ghi stock = 0
B ghi stock = 0
```

Kết quả: bán được 2 sản phẩm dù chỉ có 1. Đây là lost update, xảy ra ở mọi hệ thống có read-modify-write mà không có bảo vệ.

### Cách 1 — Atomic update có điều kiện

```sql
UPDATE products
SET stock = stock - 1
WHERE id = ?
  AND stock > 0;
```

Kiểm tra số dòng ảnh hưởng: 1 là thành công, 0 là hết hàng. Database đảm bảo tính nguyên tử của một câu lệnh nên hai request đồng thời không thể cùng trừ về âm. Không cần đọc trước, không cần lock thủ công.

Cần hiểu đúng: **atomic update không phải "không có lock"**. Ở tầng application không cần quản lý lock, nhưng database vẫn dùng row lock hoặc MVCC bên trong để đảm bảo câu lệnh không bị xen ngang. Cách diễn đạt chính xác là "application không cần explicit lock".

Pattern này dùng được cho mọi counter: `views = views + 1`, `balance = balance - 80 WHERE balance >= 80`, `UPDATE ... WHERE available > 0` khi reserve kho. Điều kiện nằm trong câu lệnh nên không có khoảng hở giữa lúc kiểm tra và lúc ghi.

### Cách 2 — Pessimistic locking

```sql
BEGIN;
SELECT * FROM products WHERE id = ? FOR UPDATE;
UPDATE products SET stock = stock - 1 WHERE id = ?;
COMMIT;
```

Request thứ hai phải chờ lock. Với TypeORM/NestJS, dùng `manager.findOne(Product, { where: { id }, lock: { mode: 'pessimistic_write' } })` bên trong transaction.

Phù hợp khi: conflict xác suất cao, consistency quan trọng, transaction ngắn. Ví dụ banking, đặt ghế, reserve kho, ghi sổ.

Nhược điểm: giảm throughput khi contention cao, và có thể deadlock. Ví dụ T1 lock A rồi chờ B, T2 lock B rồi chờ A; database phát hiện và abort một transaction. Giảm thiểu bằng cách lock theo thứ tự nhất quán và giữ transaction ngắn.

### Cách 3 — Optimistic locking

```sql
UPDATE products
SET stock = stock - 1,
    version = version + 1
WHERE id = ?
  AND version = ?;
```

Nếu 0 dòng bị ảnh hưởng, nghĩa là có người sửa trước; đọc lại và retry hoặc trả 409 Conflict.

Điểm cực kỳ quan trọng: optimistic locking **không ngăn conflict**, nó chỉ **phát hiện conflict sau khi đã xảy ra**. Vì vậy phù hợp với nơi conflict hiếm: user profile, CMS, product information, cấu hình — nơi hai admin cùng sửa một bản ghi là ngoại lệ.

Với hot counter hoặc flash sale, optimistic locking là lựa chọn tồi: 100 request cùng lúc thì 1 thành công, 99 conflict và phải retry; retry tiếp tục tạo conflict. Atomic update xử lý tuần tự 100 → 99 → 98 mà không cần retry.

### So sánh trực tiếp

| | Pessimistic | Optimistic | Atomic update |
|---|---|---|---|
| Lock tường minh ở app | Có | Không | Không |
| Blocking | Có | Không | Chỉ trong câu lệnh |
| Xử lý conflict | Ngăn từ đầu | Phát hiện sau | Loại bỏ race |
| Cần version column | Không | Thường có | Không |
| Retry | Ít | Thường cần | Thường không |
| Deadlock | Có thể | Ít | Thấp |
| Counter / stock decrement | Tốt | Kém khi contention cao | Rất tốt |
| Logic nhiều bước phụ thuộc | Rất tốt | Có thể | Không đủ |

### Có thể kết hợp

Ba kỹ thuật không loại trừ nhau. Ví dụ giữ lock order bằng pessimistic, còn trừ kho bằng atomic update trong cùng transaction; hoặc dùng câu lệnh atomic có kèm điều kiện `version = ?`, vừa là một statement vừa implement optimistic locking.

### Lớp chặn cuối

Ngoài database, thêm constraint `CHECK (stock >= 0)` để dữ liệu không thể sai kể cả khi code có bug. Với hệ thống lớn hơn, thêm idempotency key để một request bị retry không trừ kho hai lần.

## 3. Chọn isolation và cơ chế lock theo yêu cầu nào?

Câu hỏi cần trả lời trước khi chọn:

- Có chấp nhận đọc dữ liệu hơi cũ không? (dashboard, danh sách sản phẩm: có)
- Có cần chính xác tuyệt đối trên một dòng? (số dư, tồn kho: có)
- Có cần nhất quán trên một tập dòng? (báo cáo tài chính, đối soát: có)
- Tranh chấp cao hay thấp? (flash sale: rất cao)

Thứ tự quyết định thực dụng:

```text
Biểu diễn được invariant trong một câu SQL?
  → có: atomic update
  → không: conflict có thường xuyên không?
        → thấp: optimistic locking
        → cao: pessimistic locking / transaction
```

Hướng dẫn theo tình huống:

- **Counter, stock decrement, số dư có điều kiện**: atomic update kèm điều kiện, kiểm tra affected rows.
- **Hai người sửa cùng một document, conflict hiếm**: optimistic locking, trả 409 hoặc retry.
- **Booking cần nhiều bước kiểm tra (credit, voucher, kho)**: transaction + pessimistic lock trên các dòng liên quan, lock theo thứ tự nhất quán.
- **Mặc định Read Committed**: đủ cho phần lớn CRUD, concurrency cao.
- **Cần nhất quán trong transaction dài** (tính tiền, đối soát, invariant nhiều dòng): `Repeatable Read` hoặc `Serializable`, chấp nhận retry khi gặp serialization failure. Code phải retry an toàn — không có side effect bên ngoài (gửi email, gọi API) trước khi commit.

Hai nguyên tắc bổ sung:

- **Đừng giữ transaction mở trong khi gọi dịch vụ bên ngoài.** Transaction dài giữ lock, tăng nguy cơ deadlock, và nếu external call chậm thì toàn bộ hàng đợi chờ theo. Tách thành: ghi trạng thái pending trong transaction ngắn, gọi ngoài, cập nhật kết quả bằng transaction khác — kèm idempotency.
- **Đừng chọn isolation cao hơn nhu cầu.** `Serializable` ở mọi nơi biến tranh chấp bình thường thành lỗi serialization và retry, giảm throughput mà không giải quyết vấn đề nghiệp vụ nào.

## 4. Làm sao đảm bảo hai người cùng tạo một bản ghi thì chỉ một thành công?

Để database quyết định, không để application kiểm tra.

```sql
CREATE TABLE processing_records (
  id         BIGSERIAL PRIMARY KEY,
  order_id   BIGINT NOT NULL,
  user_id    BIGINT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE processing_records
  ADD CONSTRAINT uq_processing_order UNIQUE (order_id);
```

Hai request `INSERT` cùng `order_id` chạy đồng thời: một thành công, một nhận lỗi vi phạm unique. Application chỉ cần bắt lỗi và trả thông báo phù hợp ("đơn hàng đã được xử lý bởi người khác").

Vì sao **không** dùng kiểm tra trước:

```sql
-- sai khi có concurrency
IF NOT EXISTS (SELECT 1 FROM processing_records WHERE order_id = 1001) THEN
  INSERT INTO processing_records (order_id, user_id) VALUES (1001, 10);
END IF;
```

A kiểm tra thấy chưa có, B cũng kiểm tra thấy chưa có, cả hai insert. Không có gì ngăn được vì giữa `SELECT` và `INSERT` là một khoảng hở. Chỉ unique constraint (hoặc lock đúng cách) mới đóng được khoảng hở đó.

Các biến thể:

| Yêu cầu | Giải pháp |
|---|---|
| Một bản ghi cho mỗi `order_id` | `UNIQUE (order_id)` |
| Một bản ghi cho mỗi cặp `order_id` + `action_type` | `UNIQUE (order_id, action_type)` |
| Một người xử lý một đối tượng tại một thời điểm | Unique constraint hoặc cơ chế claim/lock có thời hạn |
| Chỉ một thao tác ghi thành công trên cùng một đối tượng | Transaction + lock trên dòng đối tượng |

Với PostgreSQL, có thể dùng `INSERT ... ON CONFLICT (order_id) DO NOTHING` và kiểm tra số dòng để phân biệt "tạo mới" và "đã tồn tại", tránh phải bắt exception ở tầng application. Lưu ý `ON CONFLICT` chỉ xử lý được unique/exclusion constraint đã khai báo, không thay thế được kiểm tra nghiệp vụ phức tạp.

Nguyên tắc chung: **invariant phải được database enforce**, còn application chỉ phản ứng với kết quả. Kiểm tra ở frontend hay backend chỉ là UX, không phải bảo vệ.

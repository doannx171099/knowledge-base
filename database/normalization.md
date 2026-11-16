# Chuẩn hóa Dữ liệu Quan hệ

## 1. Normalization giải quyết vấn đề gì?

Khi cùng một fact được lưu ở nhiều dòng, database có thể rơi vào trạng thái không nhất quán. Ba loại anomaly kinh điển, xét bảng `orders` chứa cả thông tin khách hàng:

```sql
orders(order_id, customer_name, customer_address, product, quantity)
```

- **Insert anomaly**: không thể thêm khách hàng chưa có order, vì thông tin khách chỉ tồn tại như một phần của dòng order.
- **Update anomaly**: khách đổi địa chỉ phải update mọi dòng order của khách đó; sót một dòng là dữ liệu mâu thuẫn.
- **Delete anomaly**: xoá order cuối cùng của khách vô tình xoá luôn thông tin khách.

Normalization là quá trình phân rã schema theo các **functional dependency** để mỗi fact chỉ được lưu ở đúng một nơi. Hai tiêu chí của một phân rã tốt:

- **Lossless join**: join các bảng sau phân rã phải tái tạo đúng dữ liệu gốc, không sinh thêm dòng giả và không mất dòng.
- **Dependency preservation**: các ràng buộc của bảng gốc vẫn kiểm tra được trên các bảng mới (không phải join để validate). Tiêu chí này không phải lúc nào cũng giữ được — BCNF là ví dụ.

Normalization không phải mục tiêu tự thân. Nó là cách để **ràng buộc dữ liệu bằng cấu trúc** thay vì bằng code application, và để tránh phải đồng bộ nhiều bản sao.

## 2. Functional dependency, superkey và candidate key

**Functional dependency** `X → Y` nghĩa là: hai dòng có cùng giá trị `X` thì buộc phải có cùng giá trị `Y`. Đây là ràng buộc của dữ liệu, không phải của một query cụ thể.

```text
order_id → customer_id, created_at
(customer_id, sku) → price        -- nếu mỗi khách chỉ có một mức giá cho một sku
```

Các khái niệm đi kèm:

- **Superkey**: tập cột xác định duy nhất một dòng.
- **Candidate key**: superkey tối thiểu (bỏ bất kỳ cột nào cũng không còn là superkey). Một bảng có thể có nhiều candidate key; một cái được chọn làm **primary key**.
- **Prime attribute**: cột thuộc ít nhất một candidate key. Các cột còn lại là non-prime.

Các dạng chuẩn được định nghĩa trên các dependency này, nên bước đầu tiên khi thiết kế luôn là viết ra các FD thực sự đúng trong domain — không phải các FD "trông có vẻ đúng" theo dữ liệu mẫu.

## 3. First Normal Form (1NF)

Bảng đạt 1NF khi mỗi ô chứa **một giá trị nguyên tử** của domain, không có repeating group, và mỗi dòng xác định được bằng key.

Vi phạm điển hình:

```sql
-- Sai: nhiều giá trị trong một cột
orders(order_id, product_ids)   -- '101,102,103'

-- Sai: repeating group
customers(customer_id, phone_1, phone_2, phone_3)
```

Cái giá của việc vi phạm 1NF không chỉ là "xấu về lý thuyết":

- Không join được, không đặt foreign key, không index hiệu quả cho từng phần tử.
- Mọi truy vấn phải parse chuỗi trong application; `WHERE product_ids LIKE '%102%'` còn match nhầm `1020`.
- Xoá một phần tử phải đọc, sửa, ghi lại cả ô — dễ mất dữ liệu khi hai request cùng sửa.

Cách sửa: tách bảng con.

```sql
order_items(order_id, product_id, quantity, PRIMARY KEY (order_id, product_id))
customer_phones(customer_id, phone, PRIMARY KEY (customer_id, phone))
```

Lưu ý: kiểu `JSONB`/array trong PostgreSQL **không** tự động vi phạm 1NF. Nó hợp lý khi cả khối dữ liệu được đọc/ghi như một value trọn vẹn (cấu hình, payload snapshot). Nó là vi phạm khi application phải tách phần tử ra để query, join, hoặc ràng buộc tính duy nhất.

## 4. Second Normal Form (2NF)

Bảng đạt 2NF khi đạt 1NF và **không có partial dependency**: không có non-prime attribute phụ thuộc vào một phần của composite key.

2NF chỉ có ý nghĩa với bảng có composite key. Xét:

```sql
order_items(order_id, product_id, product_name, quantity)
-- PRIMARY KEY (order_id, product_id)
-- product_id → product_name  (chỉ phụ thuộc một phần key)
```

`product_name` lặp lại ở mọi dòng bán cùng sản phẩm. Hệ quả: đổi tên sản phẩm phải update toàn bộ lịch sử order — và mọi dòng cũ sẽ hiển thị tên mới, có thể sai về mặt nghiệp vụ.

Cách sửa: tách phần phụ thuộc một phần ra bảng riêng.

```sql
products(product_id PK, name, ...)
order_items(order_id, product_id, quantity, unit_price_at_order)
```

Chú ý phân biệt: `unit_price` ở `order_items` **không** phải partial dependency cần loại bỏ, vì giá tại thời điểm bán là một fact khác với giá hiện tại của sản phẩm. Nếu gộp `unit_price` vào `products` thì mất lịch sử giá.

## 5. Third Normal Form (3NF)

Bảng đạt 3NF khi đạt 2NF và **không có transitive dependency**: non-prime attribute không phụ thuộc vào một non-prime attribute khác.

```sql
orders(order_id PK, customer_id, customer_name, customer_city)
-- order_id → customer_id → customer_name    (transitive)
```

Định nghĩa hình thức: với mọi FD không tầm thường `X → A`, `X` là superkey **hoặc** `A` là prime attribute. Điều kiện thứ hai là lý do 3NF "dễ đạt" hơn BCNF: nó cho phép redundancy trên các candidate key.

Cách sửa:

```sql
customers(customer_id PK, name, city)
orders(order_id PK, customer_id, created_at, ...)
```

3NF là mức thực dụng cho phần lớn OLTP: nó loại bỏ hầu hết update anomaly mà vẫn giữ được dependency preservation và join ở mức chấp nhận được.

## 6. Boyce–Codd Normal Form (BCNF)

BCNF yêu cầu mạnh hơn: với mọi FD không tầm thường `X → Y`, **`X` phải là superkey**. Không có ngoại lệ cho prime attribute.

Ví dụ kinh điển:

```text
schedule(student, course, teacher)
(student, course) → teacher
teacher → course
```

Candidate key là `(student, course)` và `(student, teacher)`. `teacher → course` có vế trái không phải superkey → vi phạm BCNF (nhưng vẫn đạt 3NF vì `course` là prime attribute).

Phân rã:

```text
teacher_course(teacher, course)     -- teacher → course
student_teacher(student, teacher)
```

Phân rã này là lossless nhưng **không** giữ được FD `(student, course) → teacher`: muốn kiểm tra "mỗi cặp student–course chỉ có một teacher" phải join hai bảng. Đây là trade-off cố hữu của BCNF.

Trong thực tế, schema OLTP nhắm tới 3NF, và chỉ đẩy lên BCNF khi phát hiện redundancy thật sự gây lỗi cập nhật.

## 7. 4NF và 5NF

Hai dạng chuẩn cao hơn xử lý các dependency không phải functional:

- **4NF**: loại bỏ **multivalued dependency** — hai tập giá trị độc lập bị nhồi chung một bảng.

  ```text
  employee_skills_languages(employee, skill, language)
  ```

  Nếu `skill` và `language` không liên quan gì tới nhau, bảng này buộc phải ghi tích Descartes: thêm một skill phải nhân bản mọi language. Tách thành `employee_skills` và `employee_languages`.

- **5NF** (project-join normal form): loại bỏ **join dependency** — bảng biểu diễn được bằng join của các projection nhỏ hơn mà không mất thông tin. Hiếm gặp trong schema nghiệp vụ thông thường; giá trị thực tế là để nhận diện các bảng "ba chiều" đang trộn nhiều quan hệ độc lập.

Không cần thuộc lòng thuật toán phân rã cho phỏng vấn senior; cần nhận ra dấu hiệu: một bảng mà mọi tổ hợp giá trị đều phải tồn tại, hoặc bảng chỉ tồn tại để trả lời một query cụ thể.

## 8. Normalization có làm chậm hệ thống không?

Có, nhưng thường ít hơn lo ngại. Chuẩn hóa đổi chi phí:

- **Được**: ít dữ liệu trùng → cache đệm được nhiều hơn, ít I/O hơn trên bảng lớn; update chỉ chạm một dòng; constraint do database đảm bảo thay vì code.
- **Mất**: query cần nhiều join hơn, mỗi join là một lần tra index và một roundtrip trong execution plan.

Với index đúng, join trên khóa ngoại là phép tra cứu `O(log n)` và thường rẻ hơn việc đọc nhiều bản sao dữ liệu lớn. Vấn đề thật sự thường không phải số join mà là:

- Join thiếu index, hoặc join trên cột có kiểu/encoding lệch.
- **N+1 query** ở tầng application — nguyên nhân phổ biến hơn nhiều so với schema chuẩn hóa.
- Query trả về quá nhiều cột/dòng, không phải số bảng.

Nguyên tắc: đo bằng execution plan trước khi kết luận. Chi tiết đọc plan và composite index ở [index-and-query-optimization.md](index-and-query-optimization.md).

## 9. Denormalization có kiểm soát

Denormalization là chủ động lưu trùng dữ liệu để đổi lấy tốc độ đọc. Nó hợp lý khi:

- **Aggregate nóng**: `like_count`, `average_rating`, số dư ví. Đếm lại từ bảng chi tiết ở mỗi lần đọc là quá đắt.
- **Snapshot nghiệp vụ**: `unit_price_at_order`, `shipping_address_at_order`. Đây không hẳn là denormalization — đó là fact lịch sử, và việc lưu riêng là bắt buộc để đúng nghiệp vụ.
- **Read model / reporting**: star schema, materialized view, bảng tổng hợp chạy theo batch. OLAP cố tình phi chuẩn hóa (fact + dimension) vì join lớn trên hàng tỷ dòng tốn kém hơn nhiều so với scan một bảng rộng.
- **Document database**: embed dữ liệu con khi chúng luôn được đọc/ghi cùng parent và không cần query độc lập. Đây là denormalization theo thiết kế, không phải ngoại lệ.

Cái giá và cách kiểm soát:

- Update anomaly quay lại → phải có **một writer duy nhất** cho mỗi bản sao, các đường ghi khác đi qua nó.
- Bản sao có thể drift → cần job đối soát định kỳ hoặc rebuild từ nguồn sự thật.
- Query phức tạp hơn khi phải biết bản nào là nguồn sự thật → ghi rõ invariant trong comment/migration, không để người sau đoán.
- Counter cập nhật đồng thời là điểm tranh chấp → cân nhắc atomic increment hoặc ghi log rồi tổng hợp, xem [transactions-and-concurrency.md](transactions-and-concurrency.md).

Quy tắc thực dụng: **chuẩn hóa trước, denormalize sau khi có số đo**, và mỗi lần denormalize phải trả lời được "ai ghi bản sao này, ai đối soát nó".

## 10. Chuẩn hóa tới đâu trong thiết kế thực tế?

- Bắt đầu ở 3NF cho OLTP. Đây là điểm cân bằng giữa anomaly và chi phí join, đồng thời giữ được dependency preservation.
- Viết ra FD trước khi vẽ bảng. Nếu không nêu được "cột này phụ thuộc vào cái gì và tại sao", rất dễ tạo bảng trộn nhiều entity.
- Dùng constraint thay cho code: `PRIMARY KEY`, `UNIQUE`, `FOREIGN KEY`, `CHECK`. Mỗi business rule ràng buộc được ở database là một lớp bug bị loại bỏ.
- Không lưu danh sách dạng CSV. Nếu cần nhiều giá trị, đó là quan hệ nhiều-nhiều và cần bảng nối.
- Tách bảng theo **entity và vòng đời**, không theo màn hình. Bảng sinh ra để phục vụ một trang UI thường là read model, không phải schema chuẩn.
- Khi access pattern đòi hỏi, thêm read model/materialized view bên cạnh schema chuẩn hóa thay vì bóp méo bảng gốc.
- Rà lại schema khi thêm feature: một cột mới lặp dữ liệu đã có ở bảng khác là dấu hiệu cần dừng lại và hỏi "nguồn sự thật ở đâu".

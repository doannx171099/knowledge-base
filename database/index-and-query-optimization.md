# Index và Tối ưu Truy vấn

## 1. Index hoạt động thế nào và tại sao không phải lúc nào cũng tốt?

Index là cấu trúc dữ liệu phụ giúp tìm dòng nhanh hơn mà không phải đọc toàn bảng. Với B-tree (mặc định của PostgreSQL/MySQL), index lưu giá trị đã sắp xếp kèm con trỏ tới dòng, cho phép tìm kiếm theo `O(log n)` thay vì quét toàn bộ.

```sql
SELECT * FROM users WHERE email = 'a@example.com';
```

Không có index: full table scan, đọc và so sánh từng dòng. Có index trên `email`: đi từ gốc cây tới lá, lấy vị trí dòng. Với bảng vài triệu dòng, khác biệt là hàng trăm mili-giây so với vài mili-giây.

Index không miễn phí. Mỗi lệnh `INSERT`, `UPDATE`, `DELETE` đều phải cập nhật tất cả index liên quan, nên bảng có 15 index sẽ ghi chậm và tốn I/O hơn đáng kể. Index cũng chiếm disk và bộ nhớ cache — cache bị chia sẻ cho nhiều index nghĩa là mỗi index được giữ ít hơn trong RAM.

Các khái niệm cần phân biệt:

- **Selectivity**: mức độ phân biệt của cột. Cột `status` có 3 giá trị (selectivity thấp) thường không đáng index; cột `email` gần như unique (selectivity cao) rất đáng index.
- **Cardinality**: số giá trị khác nhau trong cột. Cardinality thấp → index ít hiệu quả cho truy vấn trả về nhiều dòng.
- **Covering index**: index chứa đủ cột mà query cần, nên không phải quay lại bảng (index-only scan). Tăng tốc tốt nhưng index to hơn.
- **Index scan vs sequential scan**: với truy vấn trả về phần lớn bảng, planner thường chọn sequential scan vì đọc tuần tự rẻ hơn đọc index rồi nhảy ngẫu nhiên.

Vì vậy quy tắc thực tế không phải "query chậm thì thêm index", mà là: đọc execution plan, xác định nút tốn kém nhất, rồi quyết định thêm/sửa/bỏ index. Index thừa cũng là một dạng nợ kỹ thuật, cần rà soát định kỳ.

## 2. Composite index và nguyên tắc left-most prefix

Composite index trên nhiều cột được sắp xếp theo thứ tự khai báo:

```sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at);
```

Dữ liệu trong index được tổ chức như một danh bạ: nhóm theo `user_id` trước, trong mỗi nhóm mới sắp theo `created_at`.

Truy vấn tận dụng tốt:

```sql
WHERE user_id = ? AND created_at > ?
```

Truy vấn tận dụng được một phần:

```sql
WHERE user_id = ?
```

Truy vấn **không** tận dụng được theo cùng cách:

```sql
WHERE created_at > ?
```

Lý do: muốn lọc theo `created_at` cho mọi user, database phải duyệt qua tất cả các nhóm `user_id`, tức là không có một dải liên tục theo `created_at` để nhảy tới. Đây là **left-most prefix rule**: index được dùng từ cột trái nhất sang phải, dừng lại khi gặp điều kiện không còn là equality/range liên tục.

Hệ quả thực hành:

- Đặt cột dùng cho equality trước, cột dùng cho range sau. Index `(user_id, created_at)` đúng cho "order của user X trong khoảng thời gian"; index `(created_at, user_id)` lại đúng cho "mọi order trong khoảng thời gian".
- Một index `(a, b, c)` phục vụ được truy vấn trên `(a)`, `(a,b)`, `(a,b,c)`, nên không cần tạo thêm index `(a)`.
- Range trên cột ở giữa làm các cột phía sau gần như không dùng được để lọc.
- Với `ORDER BY`, thứ tự và chiều sắp xếp của index có thể loại bỏ bước sort — đáng cân nhắc với truy vấn phân trang lớn.

## 3. Đọc execution plan: EXPLAIN và EXPLAIN ANALYZE

`EXPLAIN` cho biết **kế hoạch dự kiến**; `EXPLAIN ANALYZE` chạy thật và trả về **số liệu thực tế**, bao gồm thời gian từng node. Khi điều tra, luôn dùng `EXPLAIN ANALYZE` (cẩn thận với câu lệnh ghi, dùng trong transaction rồi rollback).

Các thông tin cần đọc:

- **Node type**: `Seq Scan` (quét toàn bảng), `Index Scan` (đi index rồi lấy dòng), `Index Only Scan` (đủ dữ liệu trong index), `Bitmap Heap Scan`.
- **Join strategy**: `Nested Loop` tốt khi một phía rất nhỏ; `Hash Join` tốt cho tập lớn không có index; `Merge Join` khi hai phía đã sắp xếp.
- **rows (estimated) vs actual rows**: lệch lớn là dấu hiệu statistic cũ hoặc điều kiện khó ước lượng. Planner chọn sai vì ước lượng sai — đây là nguyên nhân phổ biến của "tự nhiên chậm".
- **Filter/Remove rows**: số dòng bị loại sau khi đọc. Loại nhiều nghĩa là index chưa đúng hoặc thiếu.
- **Sort / External Merge**: sort trên disk thay vì memory.
- **Buffers**: `shared hit` (đọc từ cache) so với `read` (đọc từ disk).

Ví dụ dấu hiệu điển hình: `Seq Scan` trên bảng 5 triệu dòng với `rows removed by filter` rất lớn → thiếu index hoặc biểu thức không dùng được index (ví dụ `WHERE lower(email) = ...` mà không có functional index).

Sau khi thêm index, cần chạy lại plan để xác nhận planner thực sự dùng nó. Index tồn tại không có nghĩa là được dùng: kiểu dữ liệu lệch, điều kiện không sargable, hoặc planner đánh giá sequential scan rẻ hơn đều dẫn tới bỏ qua index.

## 4. Phân trang bằng OFFSET có vấn đề gì?

```sql
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 1000000;
```

Database không nhảy thẳng tới dòng thứ một triệu; nó phải tạo ra và bỏ qua từng dòng trước đó. Chi phí tăng tuyến tính theo offset:

```text
OFFSET 20        nhanh
OFFSET 100000    chậm hơn rõ rệt
OFFSET 10000000  rất đắt
```

Ngoài hiệu năng, còn vấn đề **tính nhất quán**: nếu có dòng mới được chèn giữa hai lần lấy trang, trang sau có thể lặp lại hoặc bỏ sót item, vì offset đếm theo vị trí chứ không theo dữ liệu.

Keyset (cursor) pagination sửa cả hai:

```sql
SELECT * FROM orders
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Ưu điểm: mỗi trang là một lần nhảy trực tiếp vào index, chi phí không phụ thuộc độ sâu; và không bị lặp/sót khi dữ liệu thay đổi.

Điều kiện để dùng đúng:

- `ORDER BY` phải có **tie-breaker duy nhất**, thường là `(created_at, id)`. Chỉ dùng `created_at` sẽ sai khi nhiều bản ghi cùng thời điểm.
- Index phải khớp thứ tự sắp xếp, nếu không sẽ có sort.
- Không nhảy tới trang bất kỳ. Với UI cần "trang 57", keyset không phù hợp; khi đó giới hạn độ sâu phân trang hoặc chấp nhận offset có kiểm soát.

Với UI dạng infinite scroll hoặc "xem thêm", keyset là lựa chọn mặc định. Với bảng dữ liệu cần nhảy trang tùy ý, cần cân nhắc thêm giới hạn và cache tổng số.

## 5. Phân loại index theo những góc nhìn nào?

Câu hỏi "có những loại index nào" không nên trả lời bằng một danh sách phẳng. Index được phân loại theo nhiều trục độc lập:

| Góc nhìn | Các loại |
|---|---|
| Cấu trúc dữ liệu | B-Tree/B+Tree, Hash, Bitmap, GIN, GiST, R-Tree |
| Quan hệ với dữ liệu vật lý | Clustered, Non-clustered/Secondary |
| Số cột | Single-column, Composite |
| Ràng buộc | Unique, non-unique |
| Tập dữ liệu được index | Toàn bộ, Partial/Filtered |
| Nội dung index | Thường, Covering/Included columns |
| Biểu thức | Function-based / Expression |
| Dữ liệu đặc biệt | Full-text, Spatial |

Các nhóm này **không loại trừ nhau**. `CREATE UNIQUE INDEX ... ON users(LOWER(email))` đồng thời là B-Tree, unique, single-column và expression index.

Chi tiết từng loại đáng nhớ:

- **B-Tree/B+Tree**: mặc định của PostgreSQL, MySQL InnoDB. Hỗ trợ equality, range, `BETWEEN`, `ORDER BY`, `MIN/MAX`; tìm kiếm `O(log n)`. Dùng được cho hầu hết trường hợp.
- **Hash**: chỉ tốt cho equality (`WHERE email = ?`), không hỗ trợ range hay sort vì không giữ thứ tự.
- **Clustered**: quyết định thứ tự lưu vật lý của row. Trong InnoDB, primary key chính là clustered index và leaf node chứa toàn bộ row, nên tìm theo PK không cần lookup thêm.
- **Secondary**: index độc lập. Trong InnoDB, secondary index chứa **primary key** của row, nên query theo secondary phải đi thêm một bước (bookmark lookup) vào clustered index. Hệ quả thực tế: primary key quá lớn (ví dụ UUID dạng chuỗi dài) làm mọi secondary index phình theo.
- **Unique index**: vừa tăng tốc tìm kiếm vừa enforce uniqueness; là công cụ đúng để chống ghi trùng (xem `database/transactions-and-concurrency.md`).
- **Composite**: nhiều cột, chịu left-most prefix; hữu ích khi thứ tự cột khớp cả điều kiện lọc và `ORDER BY`.
- **Covering index**: index chứa đủ cột query cần nên không phải quay lại bảng. PostgreSQL hỗ trợ `INCLUDE (status, created_at)` để thêm cột không dùng cho tìm kiếm.
- **Partial/Filtered**: chỉ index một phần row (`WHERE deleted_at IS NULL`). Index nhỏ hơn, cache tốt hơn, nhưng chỉ dùng được khi query có cùng điều kiện.
- **Expression index**: index trên `LOWER(email)` hoặc `DATE(created_at)` để query dùng hàm vẫn tận dụng được index. Query phải viết đúng biểu thức đã index.
- **Bitmap**: biểu diễn từng giá trị bằng bitmap, hiệu quả với cột cardinality thấp (`status`, `gender`) và phép AND/OR trên nhiều điều kiện; phổ biến ở hệ analytics hơn OLTP.
- **Full-text**: inverted index cho tìm kiếm từ; thay cho `LIKE '%...%'` trên bảng lớn.
- **Spatial**: cho `POINT`, `POLYGON`, `GEOMETRY`, dùng R-Tree/GiST tùy database.
- **GIN/GiST (PostgreSQL)**: GIN phù hợp `JSONB`, array, full-text (`attributes @> '{"color":"red"}'`); GiST phù hợp geometry, range, nearest-neighbor.

Với phỏng vấn, cách trả lời tốt là phân loại theo trục rồi nêu khi nào dùng, thay vì kể tên. Ví dụ: "B-Tree cho equality và range; hash chỉ equality; partial index khi phần lớn query chỉ chạm một tập con; covering index khi cần tránh lookup; GIN cho JSONB và full-text".

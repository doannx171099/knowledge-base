# Security

## 1. IAM: role, policy và nguyên tắc least privilege

IAM là lớp kiểm soát ai/cái gì được làm gì trên tài nguyên AWS. Cần phân biệt:

- **User**: danh tính dài hạn gắn với credential (access key). Trong hệ thống hiện đại nên hạn chế tối đa, vì key dài hạn rất khó thu hồi và dễ lộ.
- **Role**: danh tính được assume tạm thời, trả về credential có thời hạn. Lambda, EC2, CI/CD đều nên dùng role.
- **Policy**: tài liệu JSON mô tả quyền. Gắn vào role/user (identity policy) hoặc vào tài nguyên (resource policy, ví dụ bucket policy).
- **AssumeRole**: cơ chế lấy credential tạm thời của một role, có thể kèm điều kiện (MFA, IP, thời gian).

Nguyên tắc least privilege: mỗi role chỉ có đúng quyền cần thiết, trên đúng tài nguyên cần thiết.

Ví dụ Lambda chỉ cần đọc một bucket:

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/uploads/*"
}
```

Không nên dùng `"Action": "*", "Resource": "*"` vì tiện — đó là cách biến một lỗ hổng nhỏ thành quyền kiểm soát toàn bộ tài khoản.

Điểm thực hành đáng nói trong phỏng vấn: quyền nên gắn với **danh tính công việc** (một Lambda cụ thể, một pipeline cụ thể), không gắn với con người; và nên kiểm tra quyền thực tế bằng IAM Access Analyzer hoặc CloudTrail thay vì đoán.

## 2. JWT nên lưu ở localStorage hay HttpOnly cookie?

Không có lựa chọn an toàn tuyệt đối; mỗi cách đổi một rủi ro này lấy một rủi ro khác.

**localStorage**: JavaScript đọc được, nên nếu có XSS thì token bị lấy ngay và attacker dùng được từ bất kỳ máy nào. Ưu điểm là dễ triển khai với API cross-domain, không lo CSRF.

**HttpOnly cookie**: JavaScript không đọc được, nên XSS không lấy trực tiếp được token. Nhưng browser tự động gửi cookie theo request, nên phải chống CSRF: `SameSite=Lax/Strict`, `Secure`, kiểm tra `Origin`, và với thao tác nhạy cảm có thể dùng CSRF token.

Cần hiểu rõ mô hình tấn công thay vì học thuộc kết luận:

- XSS trong ứng dụng dùng localStorage → chiếm token, chiếm phiên.
- XSS trong ứng dụng dùng HttpOnly cookie → không đọc được token, nhưng vẫn có thể thực hiện request thay người dùng ngay trong phiên (XSS luôn nguy hiểm, chỉ là mức độ khác nhau).
- CSRF chỉ ảnh hưởng tới cơ chế credentials tự động gửi kèm (cookie), không ảnh hưởng tới header `Authorization`.

Hướng thiết kế thường được chọn: access token ngắn hạn, refresh token trong cookie HttpOnly + Secure + SameSite, xoay refresh token mỗi lần dùng, và kiểm soát CSRF ở các endpoint dùng cookie. Như vậy giảm cả thời gian sống của token bị lộ và phạm vi ảnh hưởng.

## 3. CORS có phải cơ chế bảo vệ API không?

Không. CORS là **chính sách bảo vệ người dùng của trình duyệt**, không phải cơ chế xác thực hay phân quyền cho API.

CORS quy định JavaScript trên origin A có được đọc response từ origin B hay không. Nó không chặn được:

- `curl`, Postman, script server-side gọi thẳng API.
- Request gửi từ một backend khác.
- Việc tài nguyên bị truy cập nếu kẻ tấn công đã có credential hợp lệ.

Vì vậy cấu hình `Access-Control-Allow-Origin: https://myapp.com` không có nghĩa API được bảo vệ. Nếu API không yêu cầu authentication, bất kỳ ai biết URL đều gọi được.

Điều CORS thật sự làm: ngăn trang web độc hại đọc dữ liệu từ API của bạn bằng cookie của người dùng (kết hợp với SameSite), và kiểm soát preflight cho các request "không đơn giản". Nó là một lớp phòng thủ theo chiều sâu, không phải hàng rào chính.

Sai lầm phổ biến khác: bật `Access-Control-Allow-Origin: *` cùng `Allow-Credentials: true` (không hợp lệ theo spec) hoặc phản chiếu mọi origin từ header `Origin` — cách sau vô hiệu hoá tác dụng bảo vệ của CORS.

## 4. Authentication và Authorization khác nhau thế nào? RBAC và ABAC?

**Authentication**: bạn là ai. Kiểm tra danh tính qua password, session, JWT, OIDC.
**Authorization**: bạn được làm gì. Kiểm tra quyền trên tài nguyên cụ thể.

Hai việc này thường bị gộp trong code thành `if (user.role === 'admin')` rải rác khắp nơi, dẫn tới quy tắc phân tán, khó audit, và dễ bỏ sót.

**RBAC** (role-based): quyền gắn với role. Đơn giản, phù hợp khi quyền ít và ổn định.

**ABAC** (attribute-based): quyết định dựa trên thuộc tính của subject, resource và context. Ví dụ:

```text
Manager được duyệt invoice nếu
  invoice.department == manager.department
  AND amount < manager.approvalLimit
```

Role thuần không diễn đạt được điều kiện như vậy mà không tạo ra vô số role tổ hợp.

Hướng triển khai thực tế: tập trung kiểm tra quyền vào một lớp (guard, policy service) thay vì rải trong controller; định nghĩa policy theo tài nguyên (`can(user, 'approve', invoice)`); và log các quyết định từ chối để audit. Với hệ thống nhiều tenant, kiểm tra tenant phải nằm trong cùng lớp này, không dựa vào việc client gửi đúng `tenantId`.

## 5. Làm sao thu hồi một JWT đã bị lộ?

JWT stateless được thiết kế để verify mà không cần tra cứu server, nên **không có cơ chế thu hồi tự nhiên**. Token còn hạn thì còn dùng được, trừ khi hệ thống thêm state.

Các cách xử lý, thường kết hợp:

- **Access token ngắn hạn** (5-15 phút): giới hạn cửa sổ rủi ro. Đây là biện pháp chính.
- **Refresh token dài hạn, lưu server-side**: thu hồi bằng cách xoá record; refresh token dùng một lần và xoay mỗi lần refresh để phát hiện bị đánh cắp.
- **Denylist/revocation list**: lưu `jti` của token bị thu hồi tới khi hết hạn; mỗi request kiểm tra. Đánh đổi là mất tính stateless và thêm một lần tra cứu.
- **Token version theo user**: tăng version trong DB khi logout toàn bộ thiết bị; token cũ có version thấp bị từ chối.

Với hệ thống yêu cầu thu hồi tức thời (ngân hàng, quản trị), nên dùng session server-side hoặc đặt access token rất ngắn hạn. Không nên trả lời "JWT không thu hồi được nên dùng session" mà không nêu trade-off: session dễ thu hồi nhưng cần store và tra cứu mỗi request.

## 6. Những lỗ hổng web mà fullstack cần nắm

Danh sách tối thiểu, kèm cách phòng:

- **XSS**: chèn script qua dữ liệu người dùng. Phòng bằng escape output theo context, CSP, tránh `innerHTML`, validate input. Với React, `dangerouslySetInnerHTML` là điểm cần kiểm soát.
- **CSRF**: lợi dụng cookie tự động gửi. Phòng bằng SameSite, CSRF token, kiểm tra Origin.
- **SQL Injection**: nối chuỗi vào query. Phòng bằng parameterized query/ORM đúng cách; lưu ý `ORDER BY ${column}` cũng là injection nếu không whitelist.
- **SSRF**: server bị dụ gọi URL nội bộ. Phòng bằng whitelist domain, chặn dải IP nội bộ, không trả nguyên response lỗi.
- **CORS sai cấu hình**: wildcard kèm credentials, phản chiếu mọi origin.
- **Clickjacking**: nhúng site trong iframe. Phòng bằng `X-Frame-Options`/`frame-ancestors`.
- **Open redirect**: tham số `next`/`redirect` không kiểm tra. Phòng bằng whitelist đường dẫn nội bộ.
- **Brute force / rate limiting**: giới hạn theo IP + tài khoản, có backoff, cảnh báo.

Cách trả lời tốt là gắn lỗ hổng với một quyết định thiết kế cụ thể, ví dụ: token lưu ở đâu quyết định XSS hay CSRF là rủi ro chính; API public hay nội bộ quyết định mức cần thiết của rate limit. Không nên chỉ đọc tên lỗ hổng.

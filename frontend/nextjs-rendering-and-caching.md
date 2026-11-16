# Next.js: Rendering và Caching

## 1. Server Component và Client Component khác nhau thế nào?

Server Component chạy trên server, có thể truy cập database, đọc secret, gọi backend trực tiếp; code của nó **không được gửi xuống client**. Client Component chạy ở cả hai phía (render server để tạo HTML ban đầu, sau đó hydrate ở browser) và cần thiết khi có state, effect, event handler hoặc API chỉ có trong browser.

```tsx
// Server Component (mặc định trong App Router)
export default async function ProductPage({ params }) {
  const product = await getProduct(params.id);
  return <ProductView product={product} />;
}
```

```tsx
'use client';
export function AddToCart({ productId }) {
  const [quantity, setQuantity] = useState(1);
  return <button onClick={() => addToCart(productId, quantity)}>Thêm</button>;
}
```

Nguyên tắc thiết kế: đẩy boundary `'use client'` xuống **càng sâu càng tốt**. Một button cần `onClick` không có nghĩa là cả trang phải thành Client Component. Cách làm tốt là Server Component lo phần lấy dữ liệu và render cấu trúc, Client Component chỉ là một đảo nhỏ tương tác.

Các thành phần khác trong mô hình App Router:

- **Server Action**: hàm chạy trên server, được gọi từ client như một function (qua network). Dùng cho mutation như submit form, cập nhật dữ liệu. Cần validate input và kiểm tra quyền như mọi endpoint, vì client hoàn toàn có thể gọi trực tiếp.
- **Route Handler**: HTTP endpoint trong `app/api/...`, dùng khi cần API cho bên ngoài, webhook, hoặc response không theo mô hình component.
- **Middleware**: chạy trước request, phù hợp với redirect, rewrite, kiểm tra cookie/session ở mức thô. Không phù hợp để truy vấn database nặng hoặc làm nơi đặt toàn bộ logic phân quyền.

Một lưu ý hay bị bỏ qua: Server Component render một lần cho mỗi request (hoặc theo cache), nên không có state tồn tại giữa các request. Không được dùng biến module-level để lưu dữ liệu người dùng.

## 2. Chọn CSR, SSR, SSG hay ISR thế nào?

Không chọn một chiến lược cho cả ứng dụng; chọn theo từng loại trang, dựa trên ba câu hỏi: dữ liệu có phụ thuộc người dùng không, có cần SEO không, dữ liệu thay đổi nhanh tới mức nào.

- **CSR** (render ở browser): phù hợp với ứng dụng tương tác cao, sau đăng nhập, không cần SEO — ví dụ admin panel, dashboard nội bộ. Ưu: đơn giản, tách biệt với server. Nhược: first paint chậm, cần loading state, phụ thuộc JS.
- **SSR** (render mỗi request): phù hợp với nội dung phụ thuộc người dùng hoặc cần dữ liệu mới nhất, mà vẫn cần SEO — ví dụ trang tài khoản công khai, kết quả tìm kiếm. Nhược: chi phí server mỗi request, TTFB cao hơn.
- **SSG** (build time): phù hợp với nội dung ít đổi — documentation, landing page, blog. Ưu: nhanh, đưa lên CDN dễ, rẻ. Nhược: nội dung cũ cho tới lần build sau.
- **ISR** (static + revalidate): phù hợp với nội dung "gần tĩnh" — catalog sản phẩm, bài viết cập nhật định kỳ. Trang được cache và làm mới sau khoảng thời gian hoặc theo yêu cầu.

Ví dụ phân tầng cho một ứng dụng thương mại điện tử:

```text
Marketing, docs     -> SSG
Product catalog     -> ISR (revalidate vài phút)
Product detail      -> SSR nếu giá/tồn kho cần chính xác, ngược lại ISR
Checkout, giỏ hàng  -> CSR / dynamic, không cache
Dashboard user      -> SSR + Client Component cho phần tương tác
```

Điều interviewer muốn nghe là **lý do chọn**, không phải định nghĩa. Câu trả lời tốt nêu rõ dữ liệu nào được phép cũ, cũ bao lâu, và ai chịu chi phí render.

## 3. Hydration mismatch là gì và sửa thế nào?

Server render ra HTML tại thời điểm T1. Client nhận HTML đó và render lại để gắn event handler (hydrate). Nếu output của hai lần khác nhau, React báo hydration mismatch và có thể phải render lại toàn bộ phần đó, gây nhấp nháy và mất hiệu năng.

Nguyên nhân phổ biến:

```tsx
return <div>{Date.now()}</div>;         // thời điểm khác nhau
return <div>{Math.random()}</div>;      // giá trị khác nhau
return <div>{window.innerWidth}</div>;  // server không có window
```

Các trường hợp khác: dùng `localStorage`/cookie khác nhau giữa hai phía, format ngày phụ thuộc timezone của server so với browser, render có điều kiện dựa trên `typeof window !== 'undefined'`, hoặc dữ liệu thay đổi giữa lúc render server và lúc hydrate.

Cách xử lý theo bản chất:

- **Giá trị phụ thuộc thời gian/ngẫu nhiên**: render giá trị xác định trên server, hoặc chỉ hiển thị sau khi mount (`useEffect` + state ban đầu giống server).
- **Giá trị chỉ có ở browser**: đưa vào Client Component và render sau mount, chấp nhận một khung trống ban đầu.
- **Khác biệt do timezone/locale**: format cố định ở server hoặc truyền locale xuống một cách tường minh.
- **Dữ liệu thay đổi giữa hai lần render**: đảm bảo cùng nguồn dữ liệu cho cả hai phía, hoặc dùng dynamic rendering.

Có những API cho phép bỏ qua cảnh báo mismatch, nhưng đó là che triệu chứng: HTML server và client vẫn khác nhau, chỉ là React không phàn nàn. Nên sửa nguyên nhân.

## 4. Vì sao update database rồi mà UI vẫn hiển thị dữ liệu cũ?

Vì dữ liệu có thể bị cache ở nhiều tầng độc lập, và chỉ một tầng được làm mới thì các tầng còn lại vẫn trả bản cũ:

| Tầng | Cache gì | Cách làm mới |
|---|---|---|
| Browser | response HTTP, asset | `Cache-Control`, hard reload |
| CDN / edge | HTML, RSC payload, API response | revalidate, purge, TTL |
| Next.js data cache | kết quả `fetch`, render tĩnh | `revalidatePath`/`revalidateTag`, TTL |
| Client data layer | React Query/SWR cache | invalidate query |
| Application cache | Redis, in-memory | xoá key, version key |
| Database | buffer cache, materialized view | refresh view, không cần xoá cache |

Thứ tự điều tra khi gặp hiện tượng này: dữ liệu trong DB đã đúng chưa → API trả đúng chưa → response có header cache không → client có đang dùng cache cũ không.

Với Next.js App Router, điểm dễ sai là mặc định `fetch` có thể được cache và route có thể được static hoá. Một trang "đáng lẽ động" nhưng bị build thành static sẽ không bao giờ thấy dữ liệu mới cho tới khi revalidate. Cách xử lý đúng là khai báo rõ hành vi cache ở nơi lấy dữ liệu (`cache: 'no-store'` cho dữ liệu động), và gọi revalidate sau mutation.

Với React Query, lỗi thường là mutation thành công nhưng không invalidate query liên quan, nên UI vẫn hiển thị dữ liệu cũ cho tới khi refetch tự động. Cần invalidate theo key ngay trong `onSuccess` của mutation.

Điểm cần nêu khi phỏng vấn: cache không phải lỗi, nó là lựa chọn có chủ đích. Vấn đề chỉ xảy ra khi **thời gian sống của cache không khớp với yêu cầu về độ tươi của dữ liệu**. Vì vậy câu trả lời đúng là xác định yêu cầu độ tươi cho từng loại dữ liệu, rồi đặt TTL/invalidation tương ứng.

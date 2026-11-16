# React: Rendering và Tối ưu

## 1. Khi nào một component re-render?

Các nguyên nhân chính:

- State của chính nó thay đổi.
- Component cha re-render, kéo theo con (trừ khi con được memo và props không đổi).
- Context mà nó consume có giá trị mới.
- Store bên ngoài (Redux, Zustand, React Query) báo thay đổi mà nó subscribe.

Điều quan trọng nhất cần nói rõ: **re-render không đồng nghĩa với DOM update**. React gọi lại function component để tạo một cây element mới, sau đó reconciliation so sánh với cây trước và chỉ thay đổi DOM khi thật sự khác. Với component nhẹ, việc gọi lại function gần như miễn phí.

Hiểu lầm phổ biến: "component render lại là vấn đề hiệu năng". Thực tế chỉ đúng khi việc render tốn kém (tính toán nặng, danh sách lớn, tạo nhiều element) hoặc khiến DOM thay đổi thật. Tối ưu chỉ nên làm khi có số đo: React DevTools Profiler cho biết component nào render bao nhiêu lần, mất bao lâu.

Cũng cần biết React gom (batch) nhiều lần cập nhật state trong cùng một event handler thành một lần render, và trong React 18 điều này áp dụng cả cho promise, `setTimeout` và native event handler. Nhờ vậy nhiều `setState` liên tiếp không tạo ra nhiều render.

## 2. `useMemo` và `useCallback` có nên dùng càng nhiều càng tốt?

Không.

```javascript
const total = useMemo(() => a + b, [a, b]);
```

Với phép cộng hai số, `useMemo` **đắt hơn** phép tính nó tránh. Mỗi lần render vẫn phải so sánh dependency, giữ giá trị cũ trong bộ nhớ, và thêm code phải đọc.

`useMemo` có ý nghĩa khi:

- Tính toán thật sự tốn kém (xử lý danh sách lớn, format dữ liệu phức tạp).
- Cần giữ nguyên tham chiếu để tránh effect chạy lại hoặc con memoized render lại.
- Giá trị được dùng làm dependency ở nơi khác.

`useCallback` chỉ có tác dụng khi **identity của function quan trọng**: truyền xuống component đã bọc `React.memo`, hoặc dùng làm dependency của `useEffect`. Nếu con không memoized, function mới mỗi lần render không gây hại gì, và `useCallback` chỉ thêm nhiễu.

Một điểm hay bị bỏ qua: memoization chỉ chặn được re-render khi props **shallow-equal**. Truyền một object literal hoặc array literal xuống con memoized sẽ tạo tham chiếu mới mỗi lần, làm `React.memo` vô dụng. Khi đó cách sửa là ổn định cấu trúc props hoặc chuyển sang truyền primitive, không phải thêm `useMemo` khắp nơi.

## 3. `useEffect` thường bị dùng sai ở đâu?

Sai lầm phổ biến nhất là dùng effect để **tính ra state dẫn xuất**:

```javascript
const [fullName, setFullName] = useState('');

useEffect(() => {
  setFullName(`${firstName} ${lastName}`);
}, [firstName, lastName]);
```

Đoạn này tạo thêm state, thêm một lần render, và một nguồn dữ liệu có thể lệch nhau. Cách đúng là tính trực tiếp khi render:

```javascript
const fullName = `${firstName} ${lastName}`;
```

Các trường hợp lạm dụng khác: dùng effect để phản ứng với một event (nên xử lý ngay trong handler), dùng effect để reset state khi prop đổi (có thể dùng `key` để remount), hoặc chuỗi effect nối tiếp nhau để biến đổi dữ liệu (nên gộp thành một phép tính).

`useEffect` đúng chỗ khi cần **đồng bộ React với hệ thống bên ngoài**: gọi API, mở WebSocket, đăng ký event listener của browser, làm việc với DOM API, ghi analytics, subscription.

Quy tắc thực dụng: nếu giá trị có thể tính được từ state/props hiện có, không cần effect. Nếu logic xảy ra để đáp lại một hành động của người dùng, đặt trong handler. Chỉ dùng effect khi có thứ nằm ngoài React cần được đồng bộ.

Đừng quên cleanup: mọi subscription, timer, fetch trong effect đều cần hàm cleanup để tránh leak và cập nhật state sau khi component unmount.

## 4. Race condition khi làm autocomplete xảy ra thế nào?

Người dùng gõ `c` → `ca` → `cat`, tương ứng ba request. Response có thể về sai thứ tự:

```text
cat -> 100ms
ca  -> 200ms
c   -> 500ms
```

Nếu mỗi response đều ghi vào state, kết quả cuối là của `c` — trong khi ô input đang hiển thị `cat`. Người dùng thấy gợi ý không khớp với từ đã gõ.

Đây là race condition do thời gian phản hồi không xác định, không phải do logic filter sai.

Ba lớp xử lý, nên kết hợp:

```javascript
useEffect(() => {
  const controller = new AbortController();

  fetch(`/search?q=${encodeURIComponent(query)}`, { signal: controller.signal })
    .then(res => res.json())
    .then(setResults)
    .catch(err => { if (err.name !== 'AbortError') throw err; });

  return () => controller.abort();
}, [query]);
```

- **AbortController**: huỷ request cũ khi query đổi. Giảm tải server và tránh response cũ ghi đè.
- **Kiểm tra tính hợp lệ**: bỏ qua response nếu query tại thời điểm trả về không còn là query hiện tại (dùng request id hoặc so sánh giá trị).
- **Debounce**: chỉ gửi sau khi người dùng ngừng gõ 200-300ms, giảm số request.

Lưu ý abort không đảm bảo server dừng xử lý, và không phải mọi thư viện đều trả lỗi `AbortError` giống nhau. Với data-fetching library (React Query, SWR), cơ chế cancel/stale handling đã có sẵn — nên dùng thay vì tự viết lại.

## 5. Context có thay thế Redux/Zustand không?

Không nên đặt hai thứ này ngang hàng. Chúng giải quyết hai vấn đề khác nhau.

**Context** là cơ chế truyền giá trị qua cây component mà không phải prop drilling. Nó không phải state manager: không có selector, không có devtools, không có middleware, và khi value thay đổi thì **mọi consumer** đều re-render (trừ khi tách context nhỏ hoặc bọc memo hợp lý).

Context phù hợp với dữ liệu ít thay đổi và phạm vi rộng: theme, locale, thông tin user đã đăng nhập, cấu hình feature.

**State management library** phù hợp khi state toàn cục phức tạp: nhiều màn hình cùng đọc/ghi, cần selector để tránh re-render thừa, cần devtools để trace, cần middleware cho side effect.

Điểm quan trọng hơn cả việc chọn thư viện là **phân loại state**:

- **Server state** (dữ liệu từ API): nên dùng lớp data-fetching/cache như React Query hoặc SWR, vì nó cần cache, revalidate, retry, dedupe, stale handling. Đưa dữ liệu server vào Redux thường dẫn tới viết lại toàn bộ những thứ đó bằng tay.
- **Client state** (UI state, filter, form draft): local state hoặc một store nhỏ.
- **URL state** (filter, phân trang, tab): nên nằm trên URL để chia sẻ và back/forward hoạt động.

Ứng dụng nhỏ thường chỉ cần `useState` + Context cho vài giá trị toàn cục. Thêm store chỉ nên xuất phát từ nhu cầu cụ thể, không phải từ việc "app sẽ lớn".

## 6. Các khái niệm nền cần nắm chắc

- **Virtual DOM và reconciliation**: React giữ cây element mô tả UI, so sánh cây mới với cây cũ theo kiểu và `key`, rồi áp dụng thay đổi tối thiểu. `key` sai (dùng index cho danh sách có sắp xếp/thêm/xoá) gây mất state và render sai.
- **State batching**: nhiều cập nhật trong cùng một event được gộp thành một render.
- **Controlled vs uncontrolled**: controlled lấy giá trị từ state (dễ validate, nhiều render hơn); uncontrolled để DOM giữ giá trị, đọc bằng `ref` (phù hợp form lớn hoặc tích hợp thư viện ngoài).
- **`useRef`**: giữ giá trị giữa các lần render mà không gây render; dùng cho timer id, DOM node, giá trị "previous".
- **Composition**: ưu tiên `children` và component nhỏ thay vì hệ thống props phức tạp hay inheritance.
- **Error Boundary**: bắt lỗi khi render cây con, hiển thị fallback. Không bắt được lỗi trong event handler hay async — những chỗ đó phải try/catch.
- **Suspense**: cho phép "chờ" dữ liệu hoặc code (lazy) với fallback khai báo; là nền của các tính năng streaming ở Next.js.

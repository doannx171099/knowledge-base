# Coding Interview

## 1. Phần algorithm

Với fullstack Middle–Senior, hiếm khi yêu cầu LeetCode Hard, nhưng cần giải được bài medium trong thời gian hợp lý và nói được độ phức tạp.

Các nhóm cần chắc:

- Array, string, hash map (đếm, nhóm, tìm kiếm).
- Stack, queue, deque.
- Tree, graph, BFS/DFS.
- Binary search (kể cả biến thể tìm biên).
- Sliding window, two pointers.
- Heap / top-K.

Cần nói được độ phức tạp và giải thích đánh đổi bộ nhớ: `O(1)`, `O(log n)`, `O(n)`, `O(n log n)`, `O(n²)`. Không cần tối ưu ngay từ đầu, nhưng phải nhận ra khi giải pháp hiện tại chưa đủ và nêu hướng cải thiện.

Điều được chấm nhiều hơn code chạy đúng: bạn có hỏi làm rõ input (rỗng, trùng, âm, kích thước tối đa) trước khi viết không, có nêu brute force rồi cải tiến không, có tự phát hiện edge case không.

## 2. Phần backend coding

Thường là bài nhỏ nhưng đòi hỏi đúng bản chất production:

- **Rate limiter**: token bucket hoặc sliding window; nêu trường hợp nhiều instance (cần store chia sẻ như Redis) và race condition khi tăng biến đếm.
- **Retry**: exponential backoff + jitter, phân loại lỗi, giới hạn số lần, idempotency.
- **Task queue**: enqueue, worker, retry, DLQ, xử lý trùng.
- **Pagination**: offset vs cursor, tie-breaker, kiểm tra index.
- **Transaction**: chống overselling bằng atomic update, xử lý retry khi gặp xung đột.
- **Idempotent endpoint**: nhận idempotency key, lưu kết quả, trả lại kết quả cũ khi gọi lần hai.

Điểm khác biệt giữa Middle và Senior thường nằm ở việc **có nghĩ tới failure hay không**: nếu process chết giữa chừng, nếu hai request đồng thời, nếu downstream chậm, nếu message đến hai lần.

## 3. Phần frontend coding

Các bài phổ biến:

- Autocomplete/search có debounce, huỷ request cũ, xử lý loading và empty state.
- Infinite scroll (IntersectionObserver, tránh gọi trùng, giữ vị trí scroll).
- Data table có sort/filter/pagination.
- Modal, dropdown có đóng khi click ngoài và khi nhấn Escape.
- Form validation (inline error, submit state, tránh submit hai lần).

Điều interviewer quan tâm không phải UI đẹp mà là các trạng thái phụ:

- **Loading / empty / error**: ba trạng thái này thường bị bỏ quên.
- **Race condition**: response về sai thứ tự (xem lại phần autocomplete ở `frontend/react-rendering.md`).
- **Cleanup**: clear timer, abort request, remove listener, tránh setState sau unmount.
- **Accessibility**: label, `aria-*`, keyboard navigation, focus management trong modal.
- **Performance**: tránh render thừa, danh sách dài cần virtualization, không block main thread.

Một câu hỏi nối tiếp hay gặp: "nếu danh sách có 100.000 item thì sao?" — câu trả lời đúng thường là virtualization + phân trang phía server, không phải tối ưu vòng lặp.

## 4. Interviewer chấm gì ở bài coding

Ngoài tính đúng đắn, các tín hiệu được đánh giá:

- **Làm rõ yêu cầu trước khi code**: input, output, ràng buộc, trường hợp biên.
- **Giao tiếp trong lúc làm**: nói ra giả định và đánh đổi, không im lặng 20 phút.
- **Cấu trúc code**: tách hàm hợp lý, đặt tên rõ, không lặp.
- **Xử lý lỗi**: input sai, network fail, huỷ giữa chừng.
- **Tự kiểm tra**: tự chạy thử bằng ví dụ nhỏ, tự tìm edge case trước khi được nhắc.

Một sai lầm phổ biến: cố viết giải pháp tối ưu ngay, bị kẹt, và hết giờ mà không có gì chạy được. Cách an toàn là nêu hướng đơn giản, viết cho chạy, rồi cải tiến — vừa cho thấy tư duy, vừa có kết quả.

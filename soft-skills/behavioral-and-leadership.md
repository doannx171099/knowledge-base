# Behavioral và Leadership

## 1. Vì sao phần behavioral nặng hơn ở level Senior?

Ở Middle, người ta chấm bạn có làm được việc không. Ở Senior, người ta chấm bạn có **ra quyết định tốt và kéo được người khác đi cùng** không. Kỹ thuật thuần không còn đủ để phân biệt, vì nhiều ứng viên cùng giải được bài kỹ thuật.

Những gì interviewer tìm:

- Bạn đã từng chịu trách nhiệm cho một quyết định kỹ thuật và bảo vệ được nó chưa.
- Bạn xử lý bất đồng kỹ thuật thế nào, có leo thang đúng lúc không.
- Bạn có nhận ra sai và sửa chưa.
- Bạn có giúp người khác tốt lên không.

Câu trả lời kỹ thuật hoàn hảo nhưng mọi câu chuyện đều "tôi làm một mình, mọi thứ suôn sẻ" thường bị đánh giá thấp ở level này.

## 2. Bộ câu chuyện cần chuẩn bị

Chuẩn bị 6–8 câu chuyện thật, mỗi câu 2–3 phút, phủ các loại sau:

- Một sự cố production bạn trực tiếp xử lý.
- Một quyết định kiến trúc bạn đề xuất và thuyết phục được người khác.
- Một lần bất đồng kỹ thuật với đồng nghiệp hoặc leader.
- Một vấn đề hiệu năng bạn đo, tìm ra nguyên nhân và sửa.
- Một lần refactor hệ thống cũ có rủi ro.
- Một dự án hoặc quyết định thất bại.
- Một lần mentoring hoặc unblock người khác.
- Một tình huống phải đánh đổi dưới áp lực deadline.

Mỗi câu chuyện nên có số liệu: thời gian, quy mô, mức cải thiện, số người ảnh hưởng. "Tối ưu query giúp API nhanh hơn" yếu hơn "giảm P95 của endpoint danh sách order từ 1.2s xuống 180ms bằng composite index và keyset pagination".

Tránh dùng cùng một câu chuyện cho nhiều câu hỏi, và tránh câu chuyện trong đó bạn là người duy nhất đúng còn tất cả người khác sai.

## 3. Cấu trúc STAR, cộng thêm phần học được gì

STAR:

- **Situation**: bối cảnh, ràng buộc.
- **Task**: bạn chịu trách nhiệm gì.
- **Action**: bạn đã làm gì — tập trung vào quyết định và lý do.
- **Result**: kết quả, có số liệu.

Điểm cộng đáng kể là thêm **What I learned** hoặc **What I would do differently**. Phần này cho thấy bạn có khả năng nhìn lại và điều chỉnh, thứ mà interviewer đánh giá cao ở Senior.

Ví dụ cấu trúc trả lời cho câu "kể về một sự cố production":

```text
S: traffic tăng đột biến, DB bắt đầu timeout, đơn hàng bị tạo trùng
T: tôi là người xử lý chính, cần giảm ảnh hưởng trong 30 phút
A: giới hạn concurrency, xác định connection storm qua metric,
   rollback thay đổi gần nhất, thêm unique constraint chặn trùng
R: error rate về bình thường sau 20 phút, thiệt hại ghi nhận được 12 đơn trùng
L: sau đó thêm alert trên DB connection và quy trình deploy có canary
```

Khi kể, nêu cả phần bạn chưa biết lúc đó và cách bạn thu hẹp phạm vi. Ứng viên kể mọi thứ đều rõ ràng ngay từ đầu thường không đáng tin.

## 4. Câu hỏi về một quyết định sai

Đây là câu phân biệt rõ nhất. Người chưa chuẩn bị thường né bằng cách kể một "thất bại" nhỏ, hoặc đổ lỗi cho hoàn cảnh.

Câu trả lời tốt đi theo chuỗi:

```text
quyết định -> giả định đằng sau -> giả định sai ở đâu
-> hậu quả -> cách phát hiện -> cách sửa -> thay đổi thế nào về sau
```

Ví dụ: chọn cache Redis cho một endpoint đọc để giảm tải DB, giả định dữ liệu ít thay đổi; thực tế dữ liệu cập nhật liên tục khiến người dùng thấy trạng thái cũ và gây tranh chấp với bộ phận vận hành. Phát hiện qua báo cáo, sửa bằng cách giảm TTL, thêm invalidation sau mutation, và rút ra nguyên tắc: xác định yêu cầu độ tươi của dữ liệu trước khi chọn cache.

Điều interviewer muốn thấy không phải "tôi chưa từng sai", mà là bạn có mô hình đúng về việc ra quyết định dưới điều kiện không chắc chắn, và có cơ chế phát hiện sai sớm.

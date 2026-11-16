# AGENTS.md

Knowledge base ôn phỏng vấn Fullstack Middle–Senior (Node.js/NestJS + SQL + React/Next.js + AWS). Nội dung là Markdown tiếng Việt, thuật ngữ chuyên môn giữ tiếng Anh.

Danh mục đầy đủ từng file: [README.md](README.md).

## 1. Định tuyến theo loại kiến thức

| Chủ đề | Thư mục |
|---|---|
| Runtime, framework, kiến trúc, AWS, bảo mật, system design | `backend/` |
| Index, truy vấn, transaction, concurrency | `database/` |
| React, Next.js | `frontend/` |
| Viết test, CI/CD, observability, coding interview | `miscellaneous/` |
| Chuẩn bị phỏng vấn, behavioral | `soft-skills/` |

## 2. Quy ước file

- Tên file tiếng Anh, kebab-case; một file một chủ đề hẹp.
- Đúng một `#` làm tiêu đề, khớp nội dung file.
- Mỗi file đánh số `## 1.`, `## 2.`, … liên tục từ 1; đánh lại khi thêm hoặc bớt mục.
- Mỗi mục chứa trọn một câu hỏi/chủ đề và phần giải thích của nó; không tách câu hỏi sang mục khác.
- Viết giải thích và trade-off, không chép nguyên văn câu trả lời phỏng vấn.

## 3. Quy ước commit

- Commit message viết bằng tiếng Anh.

## 4. Thêm hoặc sửa kiến thức

1. Xác định thư mục theo bảng định tuyến.
2. Tìm chủ đề trong các file hiện có (grep tiêu đề `##`) trước khi tạo mục mới.
3. Chủ đề đã có → sửa ngay tại mục đó; giữ một nguồn sự thật cho mỗi ý, nội dung trùng thì gộp và để lại tham chiếu tương đối tới file chứa bản đầy đủ.
4. Chủ đề mới → thêm `## N+1.` ở cuối file phù hợp; chỉ tạo file mới khi chủ đề đủ lớn để đứng riêng và không trùng file nào.
5. Thêm, xoá hoặc đổi tên file → cập nhật `README.md` và mọi tham chiếu chéo.

## 5. Điều kiện hoàn tất

- Mỗi file đúng một `#`, các `##` đánh số liên tục không thiếu số.
- Mọi đường dẫn `.md` trong nội dung trỏ tới file tồn tại.
- Không còn nội dung trùng lặp giữa các file; mỗi chủ đề chỉ có một bản đầy đủ.
- Code fence cân bằng; `README.md` liệt kê đủ file hiện có.

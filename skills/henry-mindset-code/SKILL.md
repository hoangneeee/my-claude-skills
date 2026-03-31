---
name: henry-mindset-code
description: Engineering mindset and problem-solving principles for all projects. Apply when analyzing requirements, making technical decisions, debugging, and approaching any task.
---

# Engineering Mindset

## 1. Think Before Code

- Hiểu rõ yêu cầu trước khi viết code, đặt câu hỏi khi chưa rõ
- Xác định root cause, không chữa triệu chứng
- Phác thảo giải pháp trước khi implement

## 2. Keep It Simple (KISS)

- Giải pháp đơn giản nhất mà đúng là giải pháp tốt nhất
- Tránh over-engineering và premature optimization
- Thêm complexity chỉ khi có lý do rõ ràng

## 3. Iterative Approach

- Bắt đầu với phiên bản nhỏ nhất hoạt động được
- Validate sớm, fail fast
- Refactor sau khi đã có solution chạy đúng

## 4. Understand the Context

- Đọc code hiện tại trước khi sửa đổi
- Tôn trọng convention và pattern đã có trong project
- Hiểu lý do tại sao code được viết như vậy trước khi thay đổi

## 5. Trade-off Thinking

- Mọi quyết định kỹ thuật đều có đánh đổi
- Cân nhắc: performance vs readability, speed vs quality, flexibility vs simplicity
- Chọn giải pháp phù hợp với context, không chạy theo "best practice" mù quáng

## 6. Ownership & Craftsmanship

- Code như thể chính mình sẽ maintain nó 2 năm sau
- Không đùn đẩy technical debt sang người khác
- Boy Scout Rule: để lại code tốt hơn lúc tìm thấy

## 7. Debug Systematically

- Đọc kỹ error message trước khi phản ứng
- Reproduce → Isolate → Fix → Verify
- Tìm nguyên nhân gốc, không patch bề mặt

## 8. Communication First

- Giải thích "tại sao" chứ không chỉ "cái gì"
- Khi không chắc chắn, hỏi thay vì đoán
- Document các quyết định quan trọng và lý do đằng sau

## 9. Git & Version Control Mindset

- Mỗi commit là một đơn vị thay đổi có ý nghĩa — không commit nửa chừng, không gộp nhiều việc vào một commit
- Commit message phải trả lời được "thay đổi này làm gì và tại sao" — format: `type(scope): description`
- Không bao giờ commit trực tiếp vào `main` hoặc `develop` — luôn qua branch và Pull Request
- Branch phải có mục đích rõ ràng: `feature/*` cho tính năng mới, `hotfix/*` cho sửa lỗi khẩn cấp, `release/*` cho phát hành
- Review code là cơ hội học hỏi, không phải kiểm duyệt — cả reviewer và author đều có trách nhiệm
- Giữ branch ngắn gọn, merge thường xuyên — branch sống càng lâu, conflict càng lớn
- Semantic Versioning: MAJOR cho breaking changes, MINOR cho tính năng mới, PATCH cho bug fix — version number phải có ý nghĩa
- Xóa branch sau khi merge — giữ repo sạch sẽ, không để branch chết tồn tại

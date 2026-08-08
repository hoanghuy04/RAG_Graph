# 12. GenerationSynthesisNode (Tổng Hợp Kết Quả & Trích Dẫn Minh Bạch)

## 1. Mạch Hoạt Động
`GenerationSynthesisNode` là node tổng hợp cuối cùng (**Fan-in Aggregation Node**). Nhiệm vụ chính của node bao gồm:
1. Gom toàn bộ ngữ cảnh (`filtered_context_chunks`, `calculation_result`, hoặc `calendar_data`) từ các nhánh xử lý trước đó.
2. Chọn và lắp ghép các prompt thành phần từ bộ `prompt_template` (`common` và `main`) dựa trên luồng nghiệp vụ.
3. Gọi LLM chính sinh câu trả lời hành chính chính xác, lịch sự và **gắn chỉ số trích dẫn minh bạch (`Citations`)** dưới câu trả lời.

---

## 2. Bảng Lắp Ghép Prompt Theo Nhánh Nghiệp Vụ

Node 12 sẽ chọn template chính từ thư mục `main/` dựa vào loại dữ liệu/luồng đi của Graph:

| Nhánh đi vào | Template chính chọn | Các khối Common đi kèm |
| :--- | :--- | :--- |
| **Single Advisory** | `chat_single_intent.yaml` hoặc `chat_academic_default.yaml` | `header`, `metadata`, `security_access_control`, `academic_domain_rules`, `response_style`, `citation_rules`, `prepared_context` |
| **Multi-Intent / So sánh** | `chat_multi_intent_synthesis.yaml` | Cả 7 khối common ở trên + `{sub_queries_list}` |
| **Calculation** | `chat_calculation_result.yaml` | `header`, `metadata`, `security_access_control`, `response_style` (Không RAG) |
| **Calendar** | `chat_calendar_result.yaml` | `header`, `metadata`, `security_access_control`, `response_style` (Không RAG) |
| **Procedure** | `chat_procedure_steps.yaml` | Cả 7 khối common giống Single Advisory |

---

## 3. Quy Tắc Trích Dẫn Nguồn (Citation Protocol)

- Mỗi thông tin điều khoản quy chế, con số GPA, định mức học phí đưa ra trong câu trả lời bắt buộc phải được gắn chỉ số trích dẫn dạng `[1]`, `[2]`.
- Ở cuối câu trả lời, hiển thị danh mục **"Nguồn Trích Dẫn Quy Định"**:
  - Format: `[X] <Tên Văn Bản> - <Phiên Bản/Năm> (<Mục/Điều/Khoản>)`
  - Ví dụ: `[1] Quy chế Đào tạo Đại học 2023 - v2.1 (Mục 3, Điều 12)`

---

## 4. Input / Output Schema & State

### Input State
- `user_query`: Câu hỏi gốc của người dùng.
- `filtered_context_chunks`: Top 3-5 chunks đã qua re-rank (nếu có).
- `academic_security_context`: Thông tin vai trò/đơn vị của user.
- `calculation_result` / `calendar_data`: Dữ liệu số/lịch (nếu đi nhánh Tool).

### Final Output Response
```markdown
Dựa trên Quy chế Đào tạo và Định mức học phí của Nhà trường, xin gửi đến bạn thông tin chi tiết về việc đăng ký học song hành:

1. **Điều kiện học vụ [1]:**
   - Sinh viên đã hoàn thành tối thiểu 30 tín chỉ của chương trình thứ nhất.
   - Điểm trung bình tích lũy (GPA) đạt từ **2.50 trở lên** tính đến thời điểm xét duyệt.

2. **Định mức học phí chênh lệch [2][3]:**
   - Ngành Công nghệ thông tin: **520.000 VNĐ / tín chỉ**.
   - Ngành Quản trị Dịch vụ Du lịch & Lữ hành: **480.000 VNĐ / tín chỉ**.

---
📌 **Tài liệu trích dẫn:**
- [1] *Quy chế Đào tạo Đại học Chính quy* (Ban hành theo QĐ 45/2023 - Điều 12)
- [2] *Định mức Học phí Ngành CNTT* (Biểu phí năm học 2024-2025)
- [3] *Định mức Học phí Ngành Du lịch* (Biểu phí năm học 2024-2025)
```

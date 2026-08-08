# 05a. DirectLLMNode (Trả Lời Trực Tiếp - No RAG)

## 1. Mạch Hoạt Động & Vai Trò Trong Graph
`DirectLLMNode` là một **LLM Agent Node** hoạt động ở **Tier 1 (Direct LLM)**. Node này xử lý các câu hỏi mang tính chất kiến thức phổ thông, giáo dục cơ bản mà không cần phải truy xuất dữ liệu quy chế nội bộ của Nhà trường.

Node gọi LLM chính với template đặc thù để trả lời nhanh, lịch sự và định hướng người dùng quay lại chủ đề học vụ nếu cần thiết.

---

## 2. Prompt Template Sử Dụng
- **File prompt chính:** [main/chat_direct_llm.yaml](file:///e:/Project/self/Flow/KLTN/prompt_template/main/chat_direct_llm.yaml)
- **Các thành phần common đi kèm:**
  - `common/header.yaml`
  - `common/academic_metadata.yaml`
  - `common/security_access_control.yaml`
  - `common/response_style.yaml`

---

## 3. Quy Tắc Trả Lời Trực Tiếp
- Trả lời ngắn gọn, chính xác và lịch sự.
- Không tự suy diễn các chính sách nội bộ của Nhà trường.
- Nếu câu hỏi vừa có yếu tố kiến thức phổ thông VỪA liên quan đến học vụ, hãy trả lời phần phổ thông và gợi ý người dùng đặt câu hỏi học vụ cụ thể để tra cứu RAG.

---

## 4. Input / Output Schema & State Update

### Input State
```json
{
  "user_query": "Ngôn ngữ lập trình Python là gì?",
  "academic_security_context": {
    "user_id": "SV20246677",
    "role": "SINH_VIEN",
    "organization_scopes": ["KHOA_CNTT", "GLOBAL"]
  }
}
```

### Output Response
Node này trực tiếp sinh ra câu trả lời và kết thúc luồng (End Node):

```markdown
Python là một ngôn ngữ lập trình bậc cao, thông dịch, hướng đối tượng và đa mục đích. Nó nổi tiếng với cú pháp rõ ràng, dễ đọc và dễ học...

💡 Nếu bạn cần hỏi về các môn học liên quan đến lập trình Python trong chương trình đào tạo ngành CNTT của trường, hãy gửi câu hỏi cho tôi nhé!
```

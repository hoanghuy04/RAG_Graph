# 03. MessageClassificationNode (Phân Loại Ý Định Người Dùng)

## 1. Mạch Hoạt Động & Vai Trò Trong Graph
`MessageClassificationNode` là một **LLM Agent Node** sử dụng mô hình LLM nhỏ/nhanh để phân tích tin nhắn người dùng (kèm lịch sử hội thoại ngắn) và phân loại ý định vào một trong **9 loại intent** chính của hệ thống.

Kết quả phân loại này sẽ quyết định luồng xử lý tối ưu (Tier) ở bước tiếp theo để tối ưu hóa chi phí và hiệu suất (Fast Path, Direct LLM, Tool Call, hoặc RAG).

---

## 2. Prompt Template Sử Dụng
- **File prompt:** [agents/message_classification.yaml](file:///e:/Project/self/Flow/KLTN/prompt_template/agents/message_classification.yaml)

---

## 3. Phân Loại 9 Intent Types

| Intent Type | Mô tả | Ví dụ |
| :--- | :--- | :--- |
| `social_chat` | Xã giao, cảm ơn, đồng ý | *"Cảm ơn bot nhiều"* |
| `general_knowledge` | Kiến thức chung phổ thông, không cần RAG quy chế | *"Python là gì?"* |
| `academic_advisory` | Hỏi quy chế, chính sách học vụ chung | *"Điều kiện nhận học bổng là gì?"* |
| `academic_comparison` | So sánh 2+ thực thể học vụ | *"Học phí ngành CNTT vs Kế toán"* |
| `academic_calculation` | Yêu cầu tính toán điểm/học phí | *"Tính GPA kỳ này cho em"* |
| `academic_procedure` | Quy trình, thủ tục hành chính | *"Làm sao bảo lưu học kỳ?"* |
| `academic_calendar` | Lịch trình học vụ, hạn chót | *"Khi nào đóng học phí học kỳ 2?"* |
| `academic_document` | Hướng dẫn xin cấp giấy tờ | *"Xin giấy xác nhận sinh viên ở đâu?"* |
| `off_topic` | Ngoài phạm vi học vụ của trường | *"Công thức nấu phở"* |

---

## 4. Input / Output Schema & State Update

### Input State
```json
{
  "user_query": "Học phí ngành Công nghệ thông tin là bao nhiêu vậy ạ?",
  "conversation_history": []
}
```

### Output State Update
Node này ghi nhận kết quả phân loại vào trường `intent_classification` của State:

```json
{
  "intent_classification": {
    "primary_intent": "academic_advisory",
    "secondary_intents": [],
    "confidence": 0.98,
    "routing_mode": "SINGLE"
  }
}
```

*Lưu ý về `routing_mode`:*
- `SINGLE`: Đối với câu hỏi đơn.
- `MULTI`: Đối với câu hỏi phức hợp cần bẻ câu hỏi con hoặc so sánh thực thể.
- `null`: Đối với social_chat, general_knowledge, off_topic.

---

## 5. Graph Routing Logic
Chuyển tiếp vô điều kiện sang Node định tuyến logic:
```python
return IntentRoutingNode()
```

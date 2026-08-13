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
| `academic_procedure` | Thủ tục làm **thay đổi trạng thái học vụ** (không sinh giấy tờ) | *"Làm sao bảo lưu học kỳ?"*, *"Muốn rút môn thì làm sao?"* |
| `academic_calendar` | Lịch trình học vụ, hạn chót | *"Khi nào đóng học phí học kỳ 2?"* |
| `academic_document` | Thủ tục để **nhận về một văn bản/giấy tờ** | *"Xin giấy xác nhận sinh viên ở đâu?"*, *"Lấy bảng điểm thế nào?"* |
| `off_topic` | Ngoài phạm vi học vụ của trường | *"Công thức nấu phở"* |

> ⚠️ **Không có nhãn `greeting` trong taxonomy.** Câu chào ở **lượt đầu tiên** đã được `GreetingDetectionNode` (01) chặn bằng regex và trả template ngay. Câu chào ở **lượt thứ 2 trở đi** được phân loại thành `social_chat` (Decision Rule 7 trong prompt), vì đích xử lý của hai trường hợp là như nhau — không cần thêm intent thứ 10.

> ⚠️ **Ranh giới `academic_procedure` ↔ `academic_document`** được quyết định bằng một câu hỏi duy nhất: *"Kết thúc thủ tục, sinh viên cầm về một tờ giấy hay chỉ đổi trạng thái học vụ?"* Trường hợp vừa đổi trạng thái vừa sinh giấy tờ (VD: đăng ký xét tốt nghiệp) → `academic_procedure`, và đặt `academic_document` vào `secondary_intents`.

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
- `SINGLE`: Câu hỏi học vụ đơn cần tra cứu tài liệu (`academic_advisory`, `academic_procedure`, `academic_document`).
- `MULTI`: Câu hỏi phức hợp cần bẻ câu hỏi con hoặc so sánh thực thể.
- `null`: Các intent **không đi qua RAG** — `social_chat`, `general_knowledge`, `off_topic`, `academic_calculation`, `academic_calendar`.

*Node này **không** phát ra mode `PROCEDURE` / `DOCUMENT`.* Hai mode đó do `IntentRoutingNode` suy ra từ `primary_intent` khi gọi `QueryTransformationNode` (xem node 04).

---

## 5. Graph Routing Logic
Chuyển tiếp vô điều kiện sang Node định tuyến logic:
```python
return IntentRoutingNode()
```

---

## 6. Điều Kiện Bỏ Qua Node (Bypass)
Node này **không chạy** khi State đang mang một yêu cầu làm rõ còn dang dở (`pending_clarification`) — khi đó lượt hiện tại là câu trả lời bổ sung tham số cho node trước, không phải một ý định mới. Xem chi tiết ở [02_security_context_extraction_node.md](02_security_context_extraction_node.md) mục 5.

Nếu bỏ qua guard này, câu trả lời ngắn của sinh viên (VD: *"2 tín chỉ ạ"*) sẽ bị phân loại nhầm thành `social_chat` và toàn bộ ngữ cảnh tính toán bị mất.

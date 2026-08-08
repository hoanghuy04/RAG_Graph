# Chi Tiết Cơ Chế Hoạt Động Của TicketFallbackNode

Tài liệu này phân tích chi tiết **TicketFallbackNode** - node kích hoạt kịch bản an toàn khi kho tài liệu chưa ghi nhận quy trình hoặc câu hỏi không đạt ngưỡng tin cậy.

---

## 1. Cấu Trúc Payload Button Tạo Ticket

Node trả về UI Action Payload để Frontend Client hiển thị Modal **"Tạo Ticket Hỗ Trợ Học Vụ"**:

```python
def create_ticket_fallback_response(state: AcademicChatState) -> Dict[str, Any]:
    return {
        "status": "FALLBACK_TICKET",
        "message": "Rất tiếc, hệ thống chưa tìm thấy hướng dẫn chính thức trong kho tài liệu quy chế hiện tại.",
        "ui_actions": [
            {
                "type": "OPEN_MODAL",
                "button_text": "Tạo Ticket Hỗ Trợ Học Vụ",
                "form_prefill": {
                    "student_id": state.academic_security_context.user_id,
                    "title": f"Hỏi về: {state.user_query[:50]}...",
                    "content": state.user_query,
                    "target_department": "PHONG_DAOTAO"
                }
            }
        ]
    }
```

---

## 2. Mã Nguồn Thực Thi Node

```python
class TicketFallbackNode(BaseNode):
    async def run(self, state: AcademicChatState) -> EndNode:
        fallback_data = create_ticket_fallback_response(state)
        state.final_response = fallback_data
        return EndNode()
```

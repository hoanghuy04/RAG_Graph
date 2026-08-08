# 13. TicketFallbackNode (Xử Lý Fallback & Tạo Ticket Hỗ Trợ)

## 1. Vai Trò & Nguyên Tắc Hoạt Động
`TicketFallbackNode` được kích hoạt khi hệ thống RAG không tìm thấy bất kỳ tri thức vượt ngưỡng tin cậy nào (`has_valid_context = False` từ `PostRetrievalRerankNode`), hoặc khi câu hỏi thuộc trường hợp ngoại lệ chưa được quy định trong VectorDB.

Node này hoạt động theo nguyên tắc:
1. **Không bịa đặt (Zero Hallucination):** Tuyệt đối không để LLM tự suy đoán quy chế khi thiếu thông tin.
2. **Hướng tới hành động (Action-Oriented):** Phản hồi lịch sự thừa nhận giới hạn thông tin, đồng thời trả về một Action Payload có cấu trúc để hiển thị **nút bấm tạo Ticket Hỗ Trợ**. Điều này cho phép sinh viên gửi thẳng câu hỏi tới Phòng Đào tạo hoặc Phòng Công tác Sinh viên (CTSV) để xử lý thủ công.

---

## 2. Input / Output Schema & Ticket Payload

### Input State
```json
{
  "user_query": "Trường có hỗ trợ gia hạn thời gian đóng học phí cho sinh viên gia đình vùng thiên tai không?",
  "has_valid_context": false,
  "academic_security_context": {
    "user_id": "SV20248899",
    "role": "SINH_VIEN",
    "organization_scopes": ["KHOA_CNTT", "GLOBAL"]
  }
}
```

### Output Response Structure (JSON Action Payload)
Node trả về nội dung tin nhắn fallback kèm cấu trúc cấu hình nút bấm:

```json
{
  "response_type": "TICKET_FALLBACK",
  "message_text": "Hiện tại hệ thống chưa tìm thấy quy định hoặc hướng dẫn cụ thể về trường hợp này trong kho tài liệu học vụ chính thức của Nhà trường.\n\nĐể được giải đáp chính xác, bạn có thể tạo một **Ticket hỗ trợ học vụ** gửi trực tiếp đến Phòng Đào tạo / Phòng Công tác Sinh viên.",
  "action_buttons": [
    {
      "label": "Tạo Ticket Hỗ Trợ Học Vụ",
      "action_type": "CREATE_TICKET_MODAL",
      "payload": {
        "prefill_subject": "Hỏi về hỗ trợ đóng học phí sinh viên vùng thiên tai",
        "prefill_department": "PHONG_CTSV",
        "user_id": "SV20248899",
        "original_query": "Trường có hỗ trợ gia hạn thời gian đóng học phí cho sinh viên gia đình vùng thiên tai không?"
      }
    }
  ]
}
```
---

## 3. UI Flow của Người Dùng
Khi nhận được payload này:
1. Giao diện Chatbot hiển thị thông báo `message_text`.
2. Phía dưới hiển thị nút bấm `"Tạo Ticket Hỗ Trợ Học Vụ"`.
3. Khi click vào nút, một pop-up modal hiện lên đã điền sẵn (prefill) Tiêu đề, Phòng CTSV phụ trách, MSSV, và câu hỏi gốc để người dùng gửi đi nhanh chóng.

# 05b. OffTopicRejectNode (Từ Chối Ngoài Phạm Vi)

## 1. Mạch Hoạt Động & Vai Trò Trong Graph
`OffTopicRejectNode` là node mang tính chất **Deterministic (Quyết định)**, không gọi LLM để tối ưu hóa chi phí. Node này được kích hoạt khi tin nhắn người dùng hoàn toàn không thuộc chủ đề học vụ hay kiến thức giáo dục phổ thông (`primary_intent = off_topic`).

Mục tiêu là từ chối khéo léo và hướng dẫn người dùng đặt câu hỏi nằm trong khả năng hỗ trợ của hệ thống.

---

## 2. Template Từ Chối Mẫu (Deterministic)
Hệ thống sử dụng template tĩnh dưới đây để trả về ngay lập tức cho người dùng:

> "Xin lỗi bạn, tôi chỉ có thể hỗ trợ các câu hỏi liên quan đến quy chế đào tạo, thủ tục hành chính, tính điểm, học phí và lịch học vụ của Nhà trường.
>
> **Ví dụ về những câu hỏi bạn có thể đặt:**
> - *'Điều kiện nhận học bổng khuyến khích học tập là gì?'*
> - *'Học phí học kỳ này của ngành CNTT tính thế nào?'*
> - *'Làm thế nào để xin bảo lưu kết quả học tập?'*
> - *'Hạn chót đóng học phí học kỳ này là khi nào?'*
>
> Hãy gửi cho tôi câu hỏi học vụ của bạn nhé!"

---

## 3. Input / Output Schema & State

### Input State
```json
{
  "user_query": "Dự báo thời tiết Hà Nội hôm nay thế nào?",
  "intent_classification": {
    "primary_intent": "off_topic",
    "confidence": 0.99
  }
}
```

### Graph Response (End Node)
Hệ thống trả trực tiếp chuỗi văn bản từ chối mà không qua LLM hay RAG pipeline.

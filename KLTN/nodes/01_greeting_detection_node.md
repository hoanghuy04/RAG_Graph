# 01. GreetingDetectionNode (Phát Hiện Câu Chào Hỏi)

## 1. Mạch Hoạt Động & Vai Trò Trong Graph
`GreetingDetectionNode` là điểm chạm đầu tiên khi tin nhắn người dùng được gửi tới hệ thống. Node này mang tính chất **Deterministic (Quyết định)**, dùng bộ lọc biểu thức chính quy (Regex) và logic chuỗi để:

1. Nhận diện các câu chào hỏi xã giao ngắn (VD: *"Chào bot"*, *"Xin chào"*, *"Hi"*, *"Hello"*).
2. Phản hồi tức thì bằng mẫu câu chào chuẩn mực kèm **Menu gợi ý các chủ đề học vụ nổi bật** mà không cần tốn chi phí gọi LLM hay truy xuất VectorDB.

---

## 2. Decision Logic & Edge Routing

```python
GREETING_PATTERNS = [
    r"^(xin\s+)?chào(\s+bot|\s+ad|\s+em|\s+thầy|\s+cô)?$",
    r"^(hi|hello|hey)(\s+bot|\s+there)?$",
    r"^cho\s+em\s+hỏi(\s+chút|\s+với)?$"
]

def check_greeting(user_query: str, history_len: int) -> bool:
    if history_len != 0:
        return False  # CHỈ kích hoạt ở lượt trò chuyện đầu tiên
    clean_query = user_query.strip().lower()
    return any(re.match(pattern, clean_query) for pattern in GREETING_PATTERNS)
```

### Graph Routing Edge
```python
if check_greeting(state.user_query, len(state.conversation_history)):
    return EndNode(result=GREETING_TEMPLATE_RESULT)
else:
    return SecurityContextExtractionNode()
```

### Câu chào ở lượt thứ 2 trở đi
Fast Path cố tình **không** bắt các lượt sau. Câu chào lúc đó đi tiếp vào luồng chính và được `MessageClassificationNode` (03) phân loại thành `social_chat` — đích xử lý tương đương template chào, nhưng không bỏ sót nội dung câu hỏi thật nếu người dùng viết *"Chào bot, cho em hỏi điều kiện học bổng"*.

Vì vậy taxonomy intent **không có nhãn `greeting`**: trường hợp đó đã được xử lý trọn vẹn tại đây (lượt đầu) và tại `social_chat` (lượt sau).

---

## 3. Template Phản Hồi Mẫu
> "Xin chào bạn! Tôi là Trợ Lý AI Học Vụ của Nhà trường. Bạn có thể hỏi tôi về:
> - 📜 **Quy chế Đào tạo & Điều kiện Học bổng**
> - 💰 **Định mức Học phí & Hạn nộp**
> - 🔄 **Điều kiện Học song hành & Rút học phần**
> - 📅 **Tra cứu Lịch thi & Bảng điểm cá nhân**
> 
> Bạn cần hỗ trợ thông tin gì hôm nay?"

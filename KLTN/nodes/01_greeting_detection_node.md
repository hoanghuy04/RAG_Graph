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
    if history_len > 1:
        return False  # Chỉ kích hoạt ở lượt trò chuyện đầu tiên
    clean_query = user_query.strip().lower()
    return any(re.match(pattern, clean_query) for pattern in GREETING_PATTERNS)
```

### Graph Routing Edge
```python
if check_greeting(state.user_query, len(state.conversation_history)):
    return EndNode(result=GREETING_TEMPLATE_RESULT)
else:
    return EarlyRejectNode()
```

---

## 3. Template Phản Hồi Mẫu
> "Xin chào bạn! Tôi là Trợ Lý AI Học Vụ của Nhà trường. Bạn có thể hỏi tôi về:
> - 📜 **Quy chế Đào tạo & Điều kiện Học bổng**
> - 💰 **Định mức Học phí & Hạn nộp**
> - 🔄 **Điều kiện Học song hành & Rút học phần**
> - 📅 **Tra cứu Lịch thi & Bảng điểm cá nhân**
> 
> Bạn cần hỗ trợ thông tin gì hôm nay?"

# 09. CalendarLookupNode (Tra Cứu Lịch Học Vụ)

## 1. Mạch Hoạt Động & Vai Trò Trong Graph
`CalendarLookupNode` hoạt động ở **Tier 3 (Calendar Lookup)**. Node này chịu trách nhiệm tra cứu các mốc thời gian chính thức (lịch thi, hạn đăng ký môn, hạn nộp học phí, lịch nghỉ lễ...) trực tiếp từ **Calendar Database (Cơ sở dữ liệu lịch)** bằng API, tránh việc truy xuất mập mờ qua RAG tài liệu PDF.

Nếu Calendar DB không trả về dữ liệu (ngoại lệ hoặc sự kiện không có sẵn), node sẽ tự động rẽ nhánh chuyển sang RAG VectorDB để tìm kiếm lịch được scan/đưa vào tài liệu hướng dẫn.

---

## 2. Quy Trình Xử Lý Chi Tiết

```
User Query ──► [Bước 1: Trích xuất Tham số Lịch] (Semester, Event Type)
                     │
                     └──► [Bước 2: Calendar DB API Query]
                               │
                               ├──► Có dữ liệu? ──► [Cập nhật State.calendar_data] ──► GenerationSynthesisNode
                               │
                               └──► Không có data? ──► [Fallback] ──► QueryTransformationNode(mode="SINGLE")
```

---

## 3. Các Loại Lịch Được Hỗ Trợ
- Lịch đăng ký môn học/học phần (Đợt 1, Đợt bổ sung, Hủy môn).
- Hạn nộp học phí từng học kỳ.
- Lịch thi cuối kỳ, giữa kỳ.
- Ngày nghỉ lễ, nghỉ Tết chính thức.

---

## 4. Input / Output Schema & State Update

### Input State
```json
{
  "user_query": "Hạn chót nộp học phí học kỳ 2 năm nay là ngày nào vậy bot?"
}
```

### Output State (Trường hợp truy vấn DB thành công)
```json
{
  "calendar_data": {
    "semester": "HK2/2024-2025",
    "event_type": "nop_hoc_phi",
    "events": [
      {
        "event_name": "Thu học phí học kỳ 2 (2024-2025)",
        "start_date": "2025-02-15",
        "end_date": "2025-03-10",
        "status": "ACTIVE"
      }
    ]
  }
}
```

---

## 5. Graph Routing Logic
```python
if state.calendar_data is not None:
    return GenerationSynthesisNode()  # Chuyển thẳng sang tổng hợp, bỏ qua RAG
else:
    # Nếu DB không có lịch này, thử dùng RAG tìm trong file PDF scan lịch
    return QueryTransformationNode(mode="SINGLE")
```

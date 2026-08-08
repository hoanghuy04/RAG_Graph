# Detail: CalendarLookupNode (Node 09)

## 1. Vai Trò & Đặc Điểm

`CalendarLookupNode` tra cứu **dữ liệu lịch học vụ có cấu trúc** từ Calendar Database — một nguồn dữ liệu riêng biệt với VectorDB. Lịch học vụ được lưu dạng structured data (ngày tháng, mốc sự kiện) thay vì văn bản quy chế, nên phải truy vấn theo phương pháp riêng.

---

## 2. Calendar DB Schema

```python
class AcademicEvent(BaseModel):
    event_id: str
    event_type: Literal[
        "TUITION_DEADLINE",       # Hạn đóng học phí
        "COURSE_REGISTRATION",    # Thời gian đăng ký môn
        "MIDTERM_EXAM",           # Thi giữa kỳ
        "FINAL_EXAM",             # Thi cuối kỳ
        "SCHOLARSHIP_APPLICATION",# Xét học bổng
        "ACADEMIC_WARNING",       # Thông báo cảnh báo học vụ
        "SCHOOL_HOLIDAY",         # Nghỉ lễ
        "ORIENTATION_WEEK",       # Tuần sinh hoạt công dân
        "GRADUATION_CEREMONY",    # Lễ tốt nghiệp
    ]
    academic_year: str          # "2024-2025"
    semester: int               # 1, 2, or 3 (hè)
    start_date: date
    end_date: date | None
    scope: str                  # "GLOBAL" or "KHOA_CNTT" etc.
    min_access_level: int
    description: str
```

---

## 3. Query Logic & Phân Quyền

```python
async def lookup_calendar(
    query_keywords: List[str],
    security_ctx: AcademicSecurityContext
) -> List[AcademicEvent]:
    # 1. Parse thời gian đề cập trong câu hỏi (nếu có)
    #    VD: "học kỳ 2" → semester=2, "năm nay" → current academic year
    
    # 2. Map từ khóa → event_type
    #    "đóng học phí" → TUITION_DEADLINE
    #    "đăng ký môn" → COURSE_REGISTRATION
    
    # 3. Truy vấn Calendar DB với filter phân quyền:
    events = calendar_db.query(
        event_types=mapped_types,
        academic_year=detected_year,
        semester=detected_semester,
        scope_filter=security_ctx.organization_scopes,
        access_level_filter=security_ctx.max_access_level
    )
    return events
```

---

## 4. Fallback sang VectorDB

Nếu Calendar DB không tìm thấy sự kiện (dữ liệu chưa nhập cho năm học hiện tại):
```python
return QueryTransformationNode(
    mode="SINGLE",
    prefill_query=f"Thời gian {user_query_summarized} học kỳ {semester} năm học {academic_year}"
)
```

# 02. SecurityContextExtractionNode (Trích Xuất Bối Cảnh Phân Quyền)

## 1. Mạch Hoạt Động & Vai Trò Trong Graph
`SecurityContextExtractionNode` là node mang tính chất **Deterministic (Quyết định)**. Node này chịu trách nhiệm giải mã mã JWT Token của người dùng gửi lên để trích xuất bối cảnh phân quyền học vụ.

Hệ thống chatbot học vụ yêu cầu phân cấp tài liệu nghiêm ngặt để đảm bảo sinh viên không tiếp cận được tài liệu của giảng viên hoặc tài liệu mật cấp cao hơn. Do đó, node này cung cấp dữ liệu nền tảng cho việc lọc bảo mật ở tầng truy xuất (Retrieval).

---

## 2. Input / Output Schema & Graph State

### Input State
```json
{
  "jwt_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### Output State
Sau khi giải mã token thành công, Graph State được cập nhật với đối tượng `academic_security_context`:

```json
{
  "academic_security_context": {
    "user_id": "SV20248899",
    "role": "SINH_VIEN",
    "organization_scopes": ["KHOA_CNTT", "GLOBAL"],
    "max_access_level": 2,
    "student_program": "Công nghệ thông tin",
    "faculty_code": null
  }
}
```

---

## 3. Cấu Trúc AcademicSecurityContext
Dữ liệu phân quyền được lưu dưới dạng Dataclass trong State:

```python
class AcademicSecurityContext(BaseModel):
    user_id: str
    role: Literal["SINH_VIEN", "GIANG_VIEN", "CAN_BO_PHONG_BAN", "ADMIN"]
    organization_scopes: List[str]   # Ví dụ: ["KHOA_CNTT", "GLOBAL"]
    max_access_level: int            # Cấp độ bảo mật cao nhất (1 đến 5)
    student_program: str | None      # Ngành học của sinh viên
    faculty_code: str | None         # Mã khoa của giảng viên
```

---

## 4. Graph Routing Logic
Mặc định node chuyển tiếp sang bước phân loại ý định:
```python
return MessageClassificationNode()
```

---

## 5. Clarification Guard (Nối Lại Ngữ Cảnh Đa Lượt)

Đây là điểm rẽ nhánh **duy nhất** của node này, và nó tồn tại để bảo vệ luồng hỏi-lại-tham số của `CalculationNode`.

**Vấn đề**: khi hệ thống hỏi *"Môn Vật lý của bạn mấy tín chỉ?"*, sinh viên trả lời *"2 tín chỉ ạ"*. Nếu chuỗi này đi vào `MessageClassificationNode`, nó là một câu ngắn không chứa câu hỏi → bị phân loại thành `social_chat` → trả về template xã giao, toàn bộ tham số đã thu thập ở lượt trước bị vứt bỏ.

**Giải pháp**: nếu State còn mang một yêu cầu làm rõ chưa hoàn tất, bỏ qua phân loại và nối thẳng về node đã đặt câu hỏi:

```python
MAX_CLARIFICATION_RETRY = 2

def route(state) -> str:
    pending = state.pending_clarification
    if pending and pending.retry_count < MAX_CLARIFICATION_RETRY:
        return pending.origin_node          # VD: "CalculationNode"

    if pending:                             # Đã hỏi quá số lần cho phép
        state.pending_clarification = None  # Dọn state, tránh kẹt vòng lặp
        logger.info("Clarification abandoned after %d tries", pending.retry_count)

    return "MessageClassificationNode"
```

**Điều kiện thoát bắt buộc**: sau `MAX_CLARIFICATION_RETRY` lần hỏi mà vẫn thiếu tham số, `pending_clarification` bị xóa và lượt hiện tại quay về luồng phân loại bình thường. Không có điều kiện này, sinh viên sẽ bị kẹt vĩnh viễn trong vòng hỏi tham số và không thể chuyển sang câu hỏi khác.

**Lưu ý bảo mật**: guard đặt **sau** khi `academic_security_context` đã được giải mã, không đặt trước. Mọi lượt — kể cả lượt trả lời bổ sung — đều phải mang bối cảnh phân quyền hợp lệ.

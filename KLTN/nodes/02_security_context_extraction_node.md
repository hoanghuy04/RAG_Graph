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
Node này luôn chuyển tiếp sang Node tiếp theo mà không rẽ nhánh dựa trên điều kiện:
```python
return MessageClassificationNode()
```

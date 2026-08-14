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
    user_id: str | None              # None với khách vãng lai
    role: Literal["KHACH", "SINH_VIEN", "GIANG_VIEN", "CAN_BO_PHONG_BAN", "ADMIN"]
    organization_scopes: List[str]   # Ví dụ: ["KHOA_CNTT", "GLOBAL"]
    max_access_level: int            # Cấp độ bảo mật cao nhất (1 đến 5)
    student_program: str | None      # Ngành học của sinh viên
    faculty_code: str | None         # Mã khoa của giảng viên
```

### 3.1 Khách vãng lai (không có JWT)

Cổng tra cứu học vụ mở cho cả thí sinh và phụ huynh chưa có tài khoản. Không có token thì node này **không lỗi** — nó dựng bối cảnh mặc định hẹp nhất:

```python
GUEST_CONTEXT = AcademicSecurityContext(
    user_id=None,
    role="KHACH",
    organization_scopes=["GLOBAL"],   # Chỉ văn bản công khai toàn trường
    max_access_level=1,               # Cấp thấp nhất
    student_program=None,
    faculty_code=None,
)
```

Hệ quả dây chuyền, và đây là lý do khách vãng lai là ca đáng thiết kế riêng chứ không phải một biến thể nhỏ:

| | Sinh viên có JWT | Khách vãng lai |
| :--- | :--- | :--- |
| Ngành, khoa, hệ đào tạo | Suy ra từ `organization_scopes` | **Không biết gì cả** |
| Số thuộc tính thường thiếu cho một câu hỏi quy chế | 0–1 | **3–4** |
| Phạm vi tài liệu | Theo khoa + cấp độ được cấp | Chỉ `GLOBAL`, cấp 1 |
| Dữ liệu cá nhân (điểm, tín chỉ tích lũy) | Tra được từ hệ thống | Không có, phải tự khai |

Cột phải chính là ca buộc `PendingClarification` phải mang **nhiều field**: một câu hỏi đơn giản như *"GPA 3.6 có được học bổng không?"* đụng ngay 4 thuộc tính chưa biết cùng lúc. Hỏi lẻ từng cái qua 4 lượt thì gần như chắc chắn mất người dùng.

**`role = "KHACH"` không mở thêm bất cứ quyền nào.** Khách tự khai "em là sinh viên K22 khoa CNTT" vẫn chỉ đọc được tài liệu `GLOBAL` cấp 1 — thông tin tự khai đi vào `confirmed_metadata` để chọn nhánh trả lời, không đi vào payload filter (xem mục 6 và [nodes/10](10_retrieval_filtering_node.md)).

---

## 4. Graph Routing Logic

Mặc định node chuyển tiếp sang bước phân loại ý định:

```python
return MessageClassificationNode()
```

Ngoại lệ duy nhất là Clarification Guard ở mục 5 — khi State còn một yêu cầu làm rõ treo từ lượt trước.

---

## 5. Clarification Guard (Nối Lại Ngữ Cảnh Đa Lượt)

Đây là điểm rẽ nhánh **duy nhất** của node này. Guard phục vụ **mọi** node có thể hỏi lại — `CalculationNode`/`AcademicComparisonNode` (Type A) lẫn `QueryTransformationNode` (Type B, do node 12 phát hiện) — không riêng nhánh tính toán.

**Vấn đề**: khi hệ thống hỏi _"Bạn đang học hệ đào tạo nào ạ?"_, sinh viên trả lời _"Chính quy ạ"_. Nếu chuỗi này đi vào `MessageClassificationNode`, nó là câu ngắn không chứa câu hỏi → bị phân loại thành `social_chat` → trả template xã giao, toàn bộ ngữ cảnh câu hỏi gốc bị vứt bỏ.

**Giải pháp**: nếu State còn mang một yêu cầu làm rõ chưa hoàn tất, giải mã câu trả lời, ghi vào `confirmed_metadata`, rồi nối thẳng về node đã đặt câu hỏi — bỏ qua phân loại.

```python
MAX_CLARIFICATION_RETRY = 2   # số lần hỏi lại tối đa cho CÙNG một bộ field

def route(state) -> str:
    pending = state.pending_clarification
    if pending is None:
        return "MessageClassificationNode"

    resolved, unresolved = resolve_form(state, pending)
    state.confirmed_metadata.update(resolved)

    if not unresolved:                       # Điền đủ mọi field
        state.pending_clarification = None
        return pending.origin_node

    if pending.retry_count < MAX_CLARIFICATION_RETRY:
        pending.retry_count += 1
        _keep_only(pending, unresolved)      # Chỉ hỏi lại phần còn thiếu
        return pending.origin_node

    state.pending_clarification = None   # Hết lượt thử → dọn state, tránh kẹt vòng lặp
    logger.info("Clarification abandoned after %d tries", pending.retry_count)
    return "MessageClassificationNode"
```

### 5.1 `resolve_form` — deterministic, không gọi LLM

```python
def resolve_form(state, pending) -> tuple[dict[str, str], list[str]]:
    """Trả về (field đã chốt, field còn thiếu)."""
    submitted = state.form_submission or {}   # UI gửi {field_id: value} khi bấm Gửi
    resolved: dict[str, str] = {}

    for field, options in zip(pending.missing_fields, pending.options):
        value = submitted.get(field)
        if value is None and len(pending.missing_fields) == 1:
            value = _match_free_text(state.user_query, options)   # Gõ tay thay vì bấm
        if value is not None and _is_valid(value, options):
            resolved[field] = value

    unresolved = [f for f in pending.missing_fields if f not in resolved]
    return resolved, unresolved


def _match_free_text(user_query: str, options: list[str] | None) -> str | None:
    if not options:
        return user_query.strip() or None     # Field mở: nhận nguyên văn
    normalized = strip_accents(user_query).lower()
    for option_id in options:
        if strip_accents(option_id).replace("_", " ") in normalized:
            return option_id
    return None
```

**Điền một phần vẫn được giữ.** Người dùng bỏ trống 1 trong 4 ô thì 3 giá trị đã điền vẫn vào `confirmed_metadata`, và lượt sau chỉ hỏi lại đúng ô còn thiếu (`_keep_only`). Vứt cả form vì thiếu một ô là cách nhanh nhất để mất người dùng.

**Chỉ đối chiếu văn bản tự do khi form có đúng 1 field.** Với form nhiều field, không có cách deterministic nào gán một câu văn xuôi vào đúng ô — buộc phải nhận qua `form_submission` từ UI. Đây là lý do form nhiều field luôn phải render nút Gửi thay vì để người dùng gõ trả lời tự nhiên.

### 5.2 Chỉ một ngưỡng, và nó chặn đúng một thứ

`MAX_CLARIFICATION_RETRY` chặn duy nhất tình huống bệnh lý: hỏi mãi **cùng một** field vì câu trả lời của sinh viên không khớp option nào. Đó là lỗi kỹ thuật (parse hỏng, sinh viên gõ lạc đề), không phải nhu cầu thông tin.

**Hệ thống KHÔNG giới hạn tổng số field được hỏi trong một phiên.** Hỏi lại vì văn bản chia nhánh theo một thuộc tính chưa biết là hành vi đúng, không phải thất bại cần bóp lại:

- Sau khi thêm **CÁCH 1** ở [`task_1.yaml`](../prompt_template/common/task_1.yaml), đường hỏi lại chỉ còn kích hoạt khi thật sự không dựng được bảng và sinh viên không tự biết thuộc tính. Những câu hỏi còn sót lại đều cần thiết.
- Số câu hỏi đã tự bị chặn: mỗi field chốt xong nằm lại trong `confirmed_metadata` và **không bao giờ được hỏi lần hai**. Trần thực tế bằng số thuộc tính phân nhánh khác nhau mà cuộc hội thoại chạm tới — hữu hạn tự nhiên.
- Một ngân sách cứng chỉ tạo failure mode mới: câu hỏi thứ N của sinh viên bị trả lời hớ hênh vì hệ thống đã hết quota, dù đó là câu hỏi hoàn toàn chính đáng về một chủ đề mới.

Thứ nên làm thay cho trần cứng là **đo**: log số lần clarification mỗi phiên và phân bố field được hỏi. Nếu số liệu cho thấy một field bị hỏi lặp bất thường, đó là tín hiệu văn bản cần gắn metadata tốt hơn ở tầng index, hoặc thuộc tính đó nên được đưa vào JWT — sửa gốc, không phải chặn ngọn.

### 5.3 Node này là single-writer của `confirmed_metadata`

Chỉ guard được ghi vào `confirmed_metadata`. Node 12 phát hiện field thiếu và **hỏi**, nhưng không bao giờ tự ghi giá trị — nó không có câu trả lời của sinh viên (câu trả lời đến ở lượt sau). Gán rời rạc ở nhiều nơi sẽ khiến giá trị bị ghi đè bằng `None` khi một node skip hoặc fail.

**Lưu ý bảo mật**: guard đặt **sau** khi `academic_security_context` đã giải mã, không đặt trước. Mọi lượt — kể cả lượt trả lời bổ sung — đều phải mang bối cảnh phân quyền hợp lệ. Xem thêm ranh giới JWT vs tự khai ở mục 6.

---

## 6. `confirmed_metadata` — Thuộc Tính Sinh Viên Tự Khai

```python
class PendingClarification(BaseModel):
    origin_node: str                     # Điểm QUAY LẠI, không phải điểm phát hiện
    pending_sub_query_id: str | None     # Sub-query nào còn treo (None nếu SINGLE)
    missing_fields: list[str]            # 1 hoặc nhiều field hỏi cùng lúc
    options: list[list[str] | None]      # Song song với missing_fields; None = ô nhập tự do
    retry_count: int = 0
```

**Bất biến bắt buộc**: `len(options) == len(missing_fields)`. Hai mảng song song, `options[i]` là tập lựa chọn của `missing_fields[i]`; `None` nghĩa là thuộc tính không có tập giá trị hữu hạn (điểm số, số tín chỉ) → UI render ô nhập tự do thay vì danh sách chọn.

Đây là dạng **nhiều field** chứ không phải một field lẻ, vì người dùng chưa đăng nhập thường thiếu 3-4 thuộc tính cùng lúc. Hỏi lẻ từng cái qua nhiều lượt thì phần lớn bỏ giữa chừng — xem mục 6.2.

`pending_sub_query_id` tồn tại cho luồng MULTI: một tin nhắn ghép có thể tách thành nhiều sub-query, trong đó vài cái đã trả lời trọn vẹn ở lượt này và chỉ một cái còn treo. Không có trường này, node 06 ở lượt sau hoặc chạy lại **toàn bộ** sub-query (tốn retrieval vô ích và sinh lại câu trả lời sinh viên đã đọc), hoặc mất luôn các câu trả lời đã có. Với `routing_mode = "SINGLE"` thì để `None`.

Schema **không có** `field_definition`. Trường này từng tồn tại để lưu nguyên văn câu hỏi LLM đã đặt, nhưng nó không phải một định nghĩa và cũng không còn tác dụng: đường resolve thành công thì giá trị đã nằm trong `confirmed_metadata`; đường resolve thất bại thì LLM đọc lại `<academic_context>` và tự sinh câu hỏi mới — chính xác hơn câu cũ vì văn bản có thể đã đổi sau lần retrieval mới.

### Hai nguồn thuộc tính, không được lẫn

|            | `academic_security_context`                              | `confirmed_metadata`              |
| :--------- | :------------------------------------------------------- | :-------------------------------- |
| Nguồn      | JWT do hệ thống nhà trường ký                            | Sinh viên tự khai trong hội thoại |
| Độ tin cậy | Thẩm quyền, đã xác minh                                  | **Chưa xác minh**                 |
| Dùng cho   | Pre-filter phân quyền (node 10)**và** chọn nhánh trả lời | **CHỈ** chọn nhánh trả lời        |
| Vòng đời   | Mỗi lượt decode lại từ token                             | Tích luỹ suốt phiên hội thoại     |

**Ranh giới cứng**: `confirmed_metadata` tuyệt đối **không** được đưa vào Qdrant payload filter ở node 10. Nếu lẫn, sinh viên chỉ cần khai _"em là học viên cao học"_ là đọc được văn bản ngoài quyền — đúng kiểu tấn công mà [`security_access_control.yaml`](../prompt_template/common/security_access_control.yaml) mục 2 đang cấm. Hai khối được render thành hai thẻ XML tên khác hẳn nhau trong prompt để LLM không nhầm nguồn.

### Vì sao phải tích luỹ qua nhiều lượt

Sinh viên chốt hệ đào tạo ở lượt 2. Đến lượt 8 họ hỏi _"Điều kiện xét học bổng thế nào?"_ — văn bản học bổng cũng chia nhánh theo hệ đào tạo. Không có state tích luỹ, hệ thống hỏi lại đúng câu đã hỏi ở lượt 2.

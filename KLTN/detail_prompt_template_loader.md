# Cơ Chế Tải & Lồng Ghép Prompt Động (Dynamic Prompt Interpolation Engine)

Tài liệu này chi tiết hóa cách thức hệ thống Chatbot Học vụ KLTN tổ chức, tải (loading) và lồng ghép động (interpolation) các prompt từ tệp cấu hình YAML thành một System Prompt hoàn chỉnh trước khi gửi tới LLM.

---

## 1. Nguyên Lý Thiết Kế 2 Pha

Hệ thống tách biệt hoàn toàn nội dung Prompt (tĩnh) ra khỏi logic điều phối (động) bằng cách chia làm 2 pha:

```
[YAML Templates] (Tệp tĩnh)
       │
       ▼  (Pha 1: Tải & Cache vào memory)
[PromptTemplates Dataclass]
       │
       ▼  (Pha 2: Lồng ghép động các lớp tham số từ Graph State)
[Final Flattened System Prompt] ──► Gửi tới LLM
```

---

## 2. Chi Tiết Pha 1: Tải Tệp Tin (Template Loading)

Tất cả các prompt được viết dưới dạng tệp cấu hình YAML trong thư mục [prompt_template/](file:///Users/admin/Desktop/Hoang_Huy/self/Flow/KLTN/prompt_template).
Hệ thống sử dụng một Module Loader để đọc các tệp này lên bộ nhớ đệm (caching) khi ứng dụng khởi chạy.

### Cấu trúc dữ liệu YAML
Các tệp YAML chứa các thông số mô tả và nội dung prompt được định nghĩa dưới trường `template` hoặc `content`:
```yaml
# Ví dụ về tệp common/header.yaml
description: "Giới thiệu AI Trợ lý học vụ"
content: |
  Bạn là Trợ Lý AI Học Vụ của trường Đại học. 
  Nhiệm vụ của bạn là tra cứu quy chế học vụ cho tài khoản: {user_id}.
```

### Cách thức hoạt động của Loader:
1. Đọc nội dung tệp YAML và parse thành Dict.
2. Trích xuất chuỗi thô từ trường `template` hoặc `content`.
3. Lưu các chuỗi thô này vào một Dataclass cache (`PromptTemplates`) dưới dạng chuỗi định dạng (Format String) của Python với các placeholder `{placeholder_name}` được giữ nguyên.

---

## 3. Chi Tiết Pha 2: Ráp & Lồng Ghép Động (Dynamic Interpolation)

Hàm `build_chat_system_prompt` thực thi nhiệm vụ lồng ghép các lớp dữ liệu theo mô hình từ trong ra ngoài (Nested Formatting):

### Bước 2.1: Format các khối con (Common Component Blocks)
Dữ liệu động từ Graph State (như metadata người dùng, tài liệu RAG truy xuất được, hay câu hỏi cần bổ sung) được đưa vào định dạng cho các khối con trong thư mục [common/](file:///Users/admin/Desktop/Hoang_Huy/self/Flow/KLTN/prompt_template/common):

* **Khối Metadata (`{metadata}`)**:
  ```python
  metadata_text = templates.academic_metadata.format(
      user_id=state.security_context.user_id,
      role=state.security_context.role,
      organization_scopes=state.security_context.organization_scopes,
      max_access_level=state.security_context.max_access_level,
  )
  ```
* **Khối Ngữ cảnh RAG (`{prepared_context}`)**:
  ```python
  prepared_context_text = templates.prepared_context.format(
      context_chunks=formatted_chunks_string
  )
  ```

### Bước 2.2: Ráp các khối con vào Khung Sườn Chính (Main Structural Templates)
Sau khi chuẩn bị xong tất cả các khối con dưới dạng chuỗi phẳng (flat string), hệ thống gom tất cả vào một dictionary `base_params`:

```python
base_params = {
    "header": templates.header,
    "academic_metadata": metadata_text,            # Khối con đã được điền thông tin ở Bước 2.1
    "security_access_control": templates.security_access_control,
    "academic_domain_rules": templates.academic_domain_rules,
    "response_style": templates.response_style,
    "citation_rules": templates.citation_rules,
    "prepared_context": prepared_context_text,    # Khối con đã được điền thông tin ở Bước 2.1
}
```

Hệ thống sẽ lựa chọn Khung sườn chính (Main Frame) tương ứng với luồng đi của Graph trong [main/](file:///Users/admin/Desktop/Hoang_Huy/self/Flow/KLTN/prompt_template/main) và gọi toán tử giải nén tham số `**base_params` để hoàn thiện:

```python
# Ví dụ: Ráp vào khung sườn luồng hỏi đáp mặc định
final_system_prompt = templates.chat_academic_default.format(**base_params)
```

---

## 4. Sơ Đồ Trực Quan Hóa Quá Trình Nesting Prompt

```
[Các tệp YAML trong common/] ──► Gọi .format() điền dữ liệu động từ Graph State
                                              │
┌─────────────────────────────────────────────┼─────────────────────────────────────────────┐
│ Dữ liệu động từ Graph State:                │ Các khối con sau khi được format:           │
│ - Thông tin phân quyền từ JWT                │ - {academic_metadata} (chứa MSSV, Role...)  │
│ - Văn bản quy chế trích xuất từ VectorDB    │ - {prepared_context} (chứa XML contexts)    │
│ - Câu hỏi định hướng/thủ tục                │ - {citation_rules}                          │
└─────────────────────────────────────────────┼─────────────────────────────────────────────┘
                                              │
                                              ▼
                                 Tạo base_params dictionary
                                              │
                                              ▼
                        templates.chat_academic_default.format(**base_params)
                                              │
                                              ▼
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│ SYSTEM PROMPT PHẲNG (FLATTENED STRING) GỬI TỚI LLM:                                       │
│                                                                                           │
│ # SYSTEM PROMPT: ACADEMIC RAG ASSISTANT                                                  │
│ Bạn là Trợ Lý AI Học Vụ Thông Minh... (header)                                            │
│                                                                                           │
│ ## Thông Tin Tài Khoản Người Dùng & Phân Quyền Access                                      │
│ - Mã người dùng: SV20248899                                                               │
│ - Vai trò (Role): SINH_VIEN                                                               │
│ - Đơn vị: ['KHOA_CNTT', 'GLOBAL'] (metadata)                                              │
│                                                                                           │
│ ## Tri Thức Học Vụ Trích Xuất Được                                                         │
│ <academic_context>                                                                        │
│   <chunk id="[1]">                                                                        │
│     <doc_name>Quy chế Đào tạo Đại học 2023</doc_name>                                     │
│     <content>Sinh viên được đăng ký học song hành khi tích lũy...</content>               │
│   </chunk>                                                                                │
│ </academic_context> (prepared_context)                                                    │
│                                                                                           │
│ ... (Các quy tắc bảo mật và phong cách phản hồi kế tiếp)                                     │
└───────────────────────────────────────────────────────────────────────────────────────────┘
```

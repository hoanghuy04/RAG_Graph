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

* **Khối Metadata (`{academic_metadata}`)**: gộp hai nguồn thuộc tính vào cùng một khối nhưng **hai thẻ XML tách biệt** — JWT đã xác minh, và thuộc tính sinh viên tự khai tích luỹ qua hội thoại:
  ```python
  metadata_text = templates.academic_metadata.format(
      user_id=state.security_context.user_id,
      role=state.security_context.role,
      organization_scopes=state.security_context.organization_scopes,
      max_access_level=state.security_context.max_access_level,
      confirmed_metadata=_render_declared_attributes(state.confirmed_metadata),
  )

  def _render_declared_attributes(confirmed: dict[str, str]) -> str:
      if not confirmed:
          return "Chưa có thuộc tính nào được sinh viên xác nhận."
      return "\n".join(f"    - {field}: {value}" for field, value in confirmed.items())
  ```
  Không bao giờ trả chuỗi rỗng — một thẻ XML trống dễ bị LLM đọc là lỗi nạp dữ liệu.
* **Khối Ngữ cảnh RAG (`{prepared_context}`)**:
  ```python
  prepared_context_text = templates.prepared_context.format(
      context_chunks=formatted_chunks_string
  )
  ```
* **Khối Hỏi Lại Metadata (`{task_2}`)** — chỉ có ở `chat_academic_advisory` và `chat_multi_intent_synthesis` (2 template duy nhất chạy qua flow Advisory hợp nhất, nơi có thể phát sinh Type B — xem [`missing_metadata_clarification_design.md`](missing_metadata_clarification_design.md)). `task_2.yaml` tự nó có 1 placeholder dữ liệu (`{missing_metadata_to_confirm}`) và nhúng lồng `{ask_user_choice_guide}`, nên phải format 2 lớp giống LISA:
  ```python
  # Nguồn duy nhất của missing_metadata_to_confirm: state.pending_clarification —
  # do CHÍNH node 12 ghi vào ở lượt hỏi trước đó (Type B tự phát hiện, không có
  # node extractor riêng nào feed vào đây như lisa). Nếu chưa từng hỏi gì, hoặc
  # lượt trước đã được giải quyết (node 06 merge câu trả lời + retrieval lại
  # thành công) thì pending_clarification = None → mặc định "Không có".
  if state.pending_clarification is not None:
      missing_metadata_text = json.dumps({
          "type": "ask_user_choice",
          "field": state.pending_clarification.missing_field,
          "options": (
              [{"id": o, "label": o} for o in state.pending_clarification.options]
              if state.pending_clarification.options else None
          ),
      }, ensure_ascii=False)
  else:
      missing_metadata_text = "Không có"

  task_2_text = templates.task_2.format(
      missing_metadata_to_confirm=missing_metadata_text,
      ask_user_choice_guide=templates.ask_user_choice_guide,   # chuỗi tĩnh, không có placeholder riêng
  )
  ```
  **Vì sao đọc lại từ `pending_clarification` thay vì luôn "Không có"?** Để xử lý đúng trường hợp retry: nếu ở lượt trước node 12 đã tự đặt field `training_type`, mà lượt sau node 06 merge câu trả lời của sinh viên vào query nhưng retrieval lại VẪN trả về đoạn văn bản chia nhánh (case hiếm, VD sinh viên trả lời mơ hồ khiến node 06 merge sai) — node 12 phải thấy lại đúng `training_type` đã hỏi lần trước để tiếp tục hỏi cho rõ, KHÔNG được tự đặt tên field khác (VD đổi thành `hinh_thuc_dao_tao`) vì sẽ làm mất liên kết `id` với lựa chọn UI đã hiển thị ở lượt trước.

  Thiếu bất kỳ placeholder nào trong hai cái trên là `KeyError` làm hỏng cả lượt hội thoại.

  **`{missing_metadata_to_confirm}` là "Không có" ở lượt đầu tiên của mọi câu hỏi** — và đó là đúng, không phải lỗi. Field Type B chỉ lộ ra khi LLM đọc `<academic_context>`, tức là **sau** khi prompt đã đóng băng; LLM không thể ghi ngược vào input của chính nó. Nó ghi nhận phát hiện bằng khối `ask_user_choice` trong **output**, rồi `collect_pending_clarification()` ([`nodes/12`](nodes/12_generation_synthesis_node.md#5-response-post-processing--thu-thập-pending_clarification-từ-output-llm)) parse ra state. Placeholder này chỉ khác "Không có" ở lượt kế tiếp, và chỉ khi sinh viên trả lời **không khớp** option nào — nếu trả lời rõ, guard đã chuyển giá trị sang `confirmed_metadata` và xoá `pending_clarification`.

  Không còn placeholder `{missing_field_definition}`: `PendingClarification` không lưu định nghĩa field, vì định nghĩa luôn nằm ở nhãn nhánh trong chính văn bản quy chế (xem [`nodes/12`](nodes/12_generation_synthesis_node.md)).

> **Khối Nhiệm vụ đối chiếu (`{task_1}`) là chuỗi tĩnh, không cần format ở bước này.** Toàn bộ tri thức học vụ đã nằm trong `<academic_context>` của `{prepared_context}`, nên `task_1` chỉ chứa quy tắc xử lý, không chứa dữ liệu động. Đây là điểm khác biệt so với LISA — hệ đó nạp tri thức theo từng cặp metadata được người dùng chốt dần qua hội thoại nên `task_1` phải có các block con (`confirmation_section`, `information_need_section`) và phải format 2 lớp. KLTN lấy bối cảnh phân quyền trực tiếp từ JWT và truy xuất một khối tri thức duy nhất, nên không có vòng chốt metadata nào để tách block.

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
    "task_1": templates.task_1,                   # Chuỗi tĩnh, không có placeholder
    "task_2": task_2_text,                        # CHỈ có ở chat_academic_advisory/chat_multi_intent_synthesis — Khối con đã format 2 lớp ở Bước 2.1
}
```

> `chat_direct_llm`, `chat_calculation_result`, `chat_ticket_fallback` không có `{task_2}` trong template nên **không cần** đưa key này vào `base_params` khi build cho các luồng đó — `.format(**base_params)` không lỗi nếu dict thừa key không dùng tới, nhưng nên loại bớt cho rõ ràng.

> **Cập nhật gộp flow (2026-08)**: `chat_single_intent.yaml`, `chat_calendar_result.yaml`, `chat_procedure_steps.yaml` đã bị gộp thành 1 file `chat_academic_advisory.yaml` (flow Advisory hợp nhất: advisory/procedure/document/calendar) — xem [`missing_metadata_clarification_design.md`](missing_metadata_clarification_design.md), [`nodes/06_query_transformation_node.md`](nodes/06_query_transformation_node.md).
>
> `{task_1}` chỉ có mặt ở các khung sườn đi qua RAG (`chat_academic_advisory`, `chat_multi_intent_synthesis`). Các khung không có `<academic_context>` — `chat_direct_llm`, `chat_calculation_result`, `chat_ticket_fallback` — **không** chèn khối này, vì quy tắc "chỉ trả lời dựa trên văn bản đối chiếu được" không áp dụng khi không có văn bản nào được truy xuất. `chat_academic_advisory` luôn có `<academic_context>` cho cả 4 loại câu hỏi nó xử lý (advisory/procedure/document/calendar) vì KLTN không có DB lịch riêng — Calendar chỉ là văn bản chung trong cùng VectorDB, nên `{task_1}` áp dụng thống nhất, không có ngoại lệ nào bỏ qua NHIỆM VỤ 1.

Hệ thống sẽ lựa chọn Khung sườn chính (Main Frame) tương ứng với luồng đi của Graph trong [main/](file:///Users/admin/Desktop/Hoang_Huy/self/Flow/KLTN/prompt_template/main) và gọi toán tử giải nén tham số `**base_params` để hoàn thiện:

```python
# Ví dụ: Ráp vào khung sườn flow Advisory hợp nhất (advisory/procedure/document/calendar)
final_system_prompt = templates.chat_academic_advisory.format(**base_params)
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
                        templates.chat_academic_advisory.format(**base_params)
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

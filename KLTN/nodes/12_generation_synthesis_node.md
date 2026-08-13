# 12. GenerationSynthesisNode (Tổng Hợp Kết Quả & Trích Dẫn Minh Bạch)

## 1. Mạch Hoạt Động
`GenerationSynthesisNode` là node tổng hợp cuối cùng (**Fan-in Aggregation Node**). Nhiệm vụ chính của node bao gồm:
1. Gom toàn bộ ngữ cảnh (`filtered_context_chunks` hoặc `calculation_result`) từ các nhánh xử lý trước đó.
2. Chọn và lắp ghép các prompt thành phần từ bộ `prompt_template` (`common` và `main`) dựa trên luồng nghiệp vụ.
3. Gọi LLM chính sinh câu trả lời hành chính chính xác, lịch sự và **gắn chỉ số trích dẫn minh bạch (`Citations`)** dưới câu trả lời.

---

## 2. Bảng Lắp Ghép Prompt Theo Nhánh Nghiệp Vụ

Node 12 sẽ chọn template chính từ thư mục `main/` dựa vào loại dữ liệu/luồng đi của Graph:

| Nhánh đi vào | Template chính chọn | Các khối Common đi kèm |
| :--- | :--- | :--- |
| **Advisory hợp nhất** (advisory / procedure / document / calendar) | `chat_academic_advisory.yaml` | `header`, `metadata`, `security_access_control`, `academic_domain_rules`, `response_style`, `citation_rules`, `prepared_context`, `task_1` |
| **Multi-Intent / So sánh** | `chat_multi_intent_synthesis.yaml` | Cả 8 khối common ở trên + `{sub_queries_list}` |
| **Calculation** | `chat_calculation_result.yaml` | `header`, `metadata`, `security_access_control`, `response_style` (Không RAG) |

> **Cập nhật gộp flow (2026-08)**: `chat_single_intent.yaml`, `chat_calendar_result.yaml`, `chat_procedure_steps.yaml` đã bị **xoá**, thay bằng **1 template duy nhất** `chat_academic_advisory.yaml`. `academic_calendar` không có DB/nguồn dữ liệu riêng — KLTN không quản lý lịch học cụ thể (hệ thống khác đảm nhiệm), chỉ trích xuất lịch tổng quan cả năm từ cùng VectorDB như mọi câu hỏi advisory khác, nên dùng chung `<academic_context>` và citation `[1][2]` như bình thường, không có nhánh `calendar_data` riêng. Rule định dạng riêng từng loại câu hỏi (bước tuần tự cho procedure, format ngày cho lịch...) nằm trong [`prompt_template/common/academic_domain_rules.yaml`](../prompt_template/common/academic_domain_rules.yaml).

---

## 3. Quy Tắc Trích Dẫn Nguồn (Citation Protocol)

- Mỗi thông tin điều khoản quy chế, con số GPA, định mức học phí đưa ra trong câu trả lời bắt buộc phải được gắn chỉ số trích dẫn dạng `[1]`, `[2]`.
- Ở cuối câu trả lời, hiển thị danh mục **"Nguồn Trích Dẫn Quy Định"**:
  - Format: `[X] <Tên Văn Bản> - <Phiên Bản/Năm> (<Mục/Điều/Khoản>)`
  - Ví dụ: `[1] Quy chế Đào tạo Đại học 2023 - v2.1 (Mục 3, Điều 12)`

---

## 4. Input / Output Schema & State

### Input State
- `user_query`: Câu hỏi gốc của người dùng.
- `filtered_context_chunks`: Top 3-5 chunks đã qua re-rank (nếu có) — dùng chung cho cả advisory/procedure/document/calendar.
- `academic_security_context`: Thông tin vai trò/đơn vị của user.
- `calculation_result`: Dữ liệu tính toán (nếu đi nhánh CalculationNode, không qua RAG).

### Final Output Response
```markdown
Dựa trên Quy chế Đào tạo và Định mức học phí của Nhà trường, xin gửi đến bạn thông tin chi tiết về việc đăng ký học song hành:

1. **Điều kiện học vụ [1]:**
   - Sinh viên đã hoàn thành tối thiểu 30 tín chỉ của chương trình thứ nhất.
   - Điểm trung bình tích lũy (GPA) đạt từ **2.50 trở lên** tính đến thời điểm xét duyệt.

2. **Định mức học phí chênh lệch [2][3]:**
   - Ngành Công nghệ thông tin: **520.000 VNĐ / tín chỉ**.
   - Ngành Quản trị Dịch vụ Du lịch & Lữ hành: **480.000 VNĐ / tín chỉ**.

---
📌 **Tài liệu trích dẫn:**
- [1] *Quy chế Đào tạo Đại học Chính quy* (Ban hành theo QĐ 45/2023 - Điều 12)
- [2] *Định mức Học phí Ngành CNTT* (Biểu phí năm học 2024-2025)
- [3] *Định mức Học phí Ngành Du lịch* (Biểu phí năm học 2024-2025)
```

---

## 5. Response Post-Processing — Thu Thập `pending_clarification` Từ Output LLM

`{task_2}` (xem [`missing_metadata_clarification_design.md`](../missing_metadata_clarification_design.md)) chỉ lo phần **NẠP VÀO** prompt — chiều ngược lại, **THU THẬP** `pending_clarification` từ chính câu trả lời LLM vừa sinh ra, là một bước **deterministic, không dùng LLM**, chạy ngay sau khi node 12 nhận response thô, trước khi trả kết quả về user:

```python
def collect_pending_clarification(raw_llm_response: str) -> PendingClarification | None:
    # 1. Trích khối ```json cuối cùng trong response (theo ask_user_choice_guide.yaml:
    #    "Chỉ MỘT block json duy nhất, đặt ở cuối câu trả lời")
    json_block = extract_trailing_json_block(raw_llm_response)
    if json_block is None:
        return None  # LLM trả lời trọn vẹn, không cần hỏi thêm → xoá pending_clarification cũ (nếu có)

    parsed = json.loads(json_block)
    if parsed.get("type") != "ask_user_choice":
        return None  # không phải JSON hỏi lại (VD lỡ có code block khác) → bỏ qua

    # 2. field_definition KHÔNG lấy từ 1 key riêng trong JSON — task_2.yaml không
    #    yêu cầu LLM tự mô tả field bằng JSON key riêng, mà lấy NGUYÊN VĂN câu hỏi
    #    tự nhiên LLM vừa viết (đoạn text ngay trước khối ```json) làm "định nghĩa".
    #    Lượt sau nạp lại đúng câu này vào {missing_field_definition} — LLM đọc lại
    #    chính câu mình từng hỏi, không cần một trường JSON tách biệt dễ lệch nghĩa.
    asked_question_text = extract_text_before_json_block(raw_llm_response)

    return PendingClarification(
        origin_node="QueryTransformationNode",  # luôn cố định — Type B luôn resume ở node 06, xem Mục 5 trong missing_metadata_clarification_design.md
        missing_field=parsed["field"],
        field_definition=asked_question_text,
        options=[o["id"] for o in parsed.get("options") or []] or None,
        retry_count=(
            state.pending_clarification.retry_count
            if state.pending_clarification and state.pending_clarification.missing_field == parsed["field"]
            else 0
        ),
    )

state.pending_clarification = collect_pending_clarification(raw_llm_response)
```

**3 điểm quan trọng**:
- `origin_node` **không phải LLM quyết định** — orchestrator gán cứng `"QueryTransformationNode"` mỗi khi parser bắt được `ask_user_choice` từ node 12, vì Type B luôn resume ở node 06 (retrieval lại với thuộc tính mới), không bao giờ resume thẳng ở node 12.
- `retry_count` **giữ nguyên** nếu field hỏi lại trùng với field đã hỏi lượt trước (cùng câu hỏi chưa được giải quyết) — việc **tăng** `retry_count` xảy ra ở bước routing của node 02 (Clarification Guard, [`nodes/02_security_context_extraction_node.md`](02_security_context_extraction_node.md)) khi quyết định route trở lại `origin_node`, không phải ở đây.
- Nếu response KHÔNG có khối `ask_user_choice` (LLM trả lời trọn vẹn — trường hợp bình thường, hoặc trường hợp hết `MAX_CLARIFICATION_RETRY` buộc phải trả lời an toàn theo rule "SO SÁNH PHƯƠNG ÁN"), `pending_clarification` được xoá về `None` — kết thúc vòng hỏi lại.

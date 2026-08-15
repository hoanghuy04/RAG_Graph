# 08. CalculationNode (Tính Toán Học Vụ)

## 1. Mạch Hoạt Động & Vai Trò Trong Graph

`CalculationNode` là một node tích hợp **LLM Agent + Python Code Tool** hoạt động ở **Tier 2 (Calculation)**. Node này xử lý các yêu cầu tính toán số học học vụ (GPA, số tín chỉ tích lũy/còn thiếu, định mức học phí) trực tiếp từ cơ sở dữ liệu điểm của sinh viên hoặc các tham số người dùng nhập vào mà **không đi qua RAG pipeline**.

---

## 2. Prompt Template Sử Dụng (Tầng Trích Xuất)

- **File prompt:** [agents/calculation_extractor.yaml](file:///e:/Project/self/Flow/KLTN/prompt_template/agents/calculation_extractor.yaml)

---

## 3. Các Công Thức & Nghiệp Vụ Hỗ Trợ

Hệ thống hỗ trợ 3 loại tính toán:

| Loại tính toán (`calculation_type`) | Công thức áp dụng                                     | Ví dụ                            |
| :---------------------------------- | :---------------------------------------------------- | :------------------------------- |
| `gpa_calculation`                   | `GPA = Σ(điểm_môn_i × tín_chỉ_i) / Σ(tín_chỉ_i)`      | Tính điểm trung bình kỳ/tích lũy |
| `credit_check`                      | `Số TC còn thiếu = Yêu cầu tốt nghiệp - Số TC đã đạt` | Kiểm tra tiến độ tốt nghiệp      |
| `tuition_calculation`               | `Học phí = Số TC đăng ký × Đơn giá ngành`             | Tính tổng học phí học kỳ         |

---

## 4. Quy Trình Xử Lý Chi Tiết

```
User Query ──► [Bước 1: Parameter Extractor] (agents/calculation_extractor.yaml)
                     │
                     ├──► Có missing_params? ──► build PendingClarification ──► GenerationSynthesisNode
                     │                            (deterministic, không LLM)     (hỏi qua {task_2})
                     │
                     └──► Tham số đầy đủ ──► [Bước 2: Python Calculator Tool]
                                                   │
                                                   └──► Cập nhật State.calculation_result ──► GenerationSynthesisNode
```

> **Cập nhật (2026-08)**: đã bỏ `AskUserClarificationNode` — nhánh thiếu tham số không còn route sang một node hỏi lại riêng. `missing_params` từ `calculation_extractor.yaml` giờ đã đúng shape `fields[]` của `ask_user_form` (`field`/`label`/`options` — xem [`agents/calculation_extractor.yaml`](../prompt_template/agents/calculation_extractor.yaml)), nên CalculationNode chỉ cần đóng gói nguyên vẹn thành `PendingClarification` và route thẳng sang `GenerationSynthesisNode`, dùng chung cơ chế hỏi lại `task_2.yaml`/`ask_user_form_guide.yaml` với luồng Advisory (Type B). Xem thảo luận đầy đủ ở [`missing_metadata_clarification_design.md`](../missing_metadata_clarification_design.md#1-5-hai-loại-field-thiếu-structural-type-a-tĩnh-vs-content-driven-type-b-động).

---

## 5. Input / Output Schema & State Update

### Input State

```json
{
  "user_query": "Kỳ này em được 8.5 môn Lập trình Web (3TC) và 7.0 môn Cơ sở dữ liệu (3TC). GPA em được mấy?",
  "academic_security_context": {
    "student_program": "Công nghệ thông tin"
  }
}
```

### Output State Update (sau khi chạy Calculator Tool)

```json
{
  "calculation_result": {
    "calculation_type": "gpa_calculation",
    "courses_evaluated": [
      { "name": "Lập trình Web", "credits": 3, "grade": 8.5 },
      { "name": "Cơ sở dữ liệu", "credits": 3, "grade": 7.0 }
    ],
    "calculated_gpa": 3.75,
    "grade_classification": "Giỏi (hệ 4 tương đương)"
  }
}
```

### Output State Update (khi thiếu tham số — không chạy Calculator Tool)

```json
{
  "calculation_result": null,
  "pending_clarification": {
    "origin_node": "CalculationNode",
    "pending_sub_query_id": null,
    "missing_fields": ["credits_registered", "price_per_credit"],
    "options": [null, null],
    "retry_count": 0
  }
}
```

---

## 6. Graph Routing Logic

Nếu trích xuất đầy đủ tham số, chạy Calculator Tool rồi chuyển sang tổng hợp, bỏ qua RAG:

```python
return GenerationSynthesisNode()
```

Nếu thiếu tham số (`len(missing_params) > 0`), đóng gói thẳng thành `PendingClarification` — bước này **deterministic, không gọi thêm LLM** vì `calculation_extractor.yaml` đã trả `missing_params` đúng shape `field`/`label`/`options`:

```python
state.pending_clarification = PendingClarification(
    origin_node="CalculationNode",
    pending_sub_query_id=None,          # Calculation không đi qua luồng MULTI sub-query
    missing_fields=[p["field"] for p in missing_params],
    options=[[o["id"] for o in p["options"]] if p["options"] else None for p in missing_params],
    retry_count=0,
)
return GenerationSynthesisNode()   # hỏi qua {task_2} trong chat_calculation_result.yaml, KHÔNG còn AskUserClarificationNode
```

### Resume — lượt sau khi người dùng điền tham số

Clarification Guard ở node 02 xử lý y hệt Type B: `resolve_form()` ghi giá trị vào `state.confirmed_metadata`, xoá `pending_clarification`, rồi route về `origin_node = "CalculationNode"`. Khi CalculationNode chạy lại, nó đọc giá trị vừa chốt từ `confirmed_metadata` theo đúng tên field (VD `confirmed_metadata["credits_registered"]`) để lấp vào `parameters` đã trích được ở lượt đầu, **bỏ qua bước gọi lại `calculation_extractor`** vì tham số đã đủ — rồi chạy thẳng Calculator Tool.

**Lưu ý ranh giới**: tham số tính toán (số tín chỉ, đơn giá, điểm số) đi vào `confirmed_metadata` giống thuộc tính tự khai của Type B, nhưng KHÔNG bao giờ được dùng để chọn nhánh văn bản hay lọt vào `<student_declared_attributes>` một cách gây nhiễu — hệ thống chỉ đọc lại đúng tên field CalculationNode cần, không đọc toàn bộ dict.

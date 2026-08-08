# Detail: CalculationNode (Node 08)

## 1. Vai Trò & Đặc Điểm

`CalculationNode` giải quyết nhu cầu **tính toán học vụ chính xác bằng số** mà sinh viên thường hỏi lặp đi lặp lại. Thay vì để LLM tự tính (dễ sai do LLM yếu toán học), node này áp dụng mô hình **Extractor LLM → Calculator Tool → Presenter LLM** đảm bảo kết quả luôn chính xác.

---

## 2. Pipeline 3 Bước

```
UserQuery
    │
    ▼
[Bước 1] calculation_extractor_agent (LLM)
    Trích xuất: loại tính toán, các tham số đầu vào
    → ClassificationResult + ParameterList + MissingParams
    │
    ├── [missing_params không rỗng] → Hỏi lại người dùng (clarification loop)
    │
    ▼
[Bước 2] Calculator Tool (Python thuần, không LLM)
    GPA Formula:     Σ(điểm_i × TC_i) / Σ(TC_i)
    Tuition Formula: số_TC_đăng_ký × đơn_giá_TC_ngành
    Credit Check:    TC_tích_lũy + TC_đăng_ký - TC_không_qua
    → CalculationResult (số chính xác + classification)
    │
    ▼
[Bước 3] GenerationSynthesisNode (LLM)
    Diễn giải kết quả bằng ngôn ngữ tự nhiên + nhận xét học lực
    Template: chat_calculation_result.yaml
```

---

## 3. 3 Loại Tính Toán Được Hỗ Trợ

### A. GPA Calculator
```python
def calculate_gpa(courses: List[Course]) -> GPAResult:
    total_weighted = sum(c.grade * c.credits for c in courses)
    total_credits  = sum(c.credits for c in courses)
    gpa            = total_weighted / total_credits
    classification = classify_gpa(gpa)  # Xuất sắc/Giỏi/Khá/TB/Yếu/Kém
    return GPAResult(gpa=round(gpa, 2), classification=classification)
```

### B. Tuition Calculator
```python
def calculate_tuition(program: str, credits_registered: int) -> TuitionResult:
    # Lookup đơn giá từ Tuition Rate Registry (hoặc VectorDB nếu cần)
    price_per_credit = lookup_price(program)
    total_tuition    = credits_registered * price_per_credit
    return TuitionResult(total=total_tuition, breakdown=...)
```

### C. Credit Progress Check
```python
def check_credit_progress(accumulated: int, required: int, current_semester: int) -> CreditResult:
    remaining = required - accumulated
    semesters_left = max(0, estimate_semesters(remaining))
    return CreditResult(remaining=remaining, estimated_semesters=semesters_left)
```

---

## 4. Fallback sang RAG

Nếu `calculation_type = "tuition_calculation"` nhưng **không có `price_per_credit` trong Tuition Registry** (ngành mới hoặc biểu phí chưa cập nhật):
```python
# Chuyển sang tìm kiếm trong VectorDB
return QueryTransformationNode(
    mode="SINGLE",
    prefill_query=f"Định mức học phí theo tín chỉ ngành {program} năm học hiện tại"
)
```

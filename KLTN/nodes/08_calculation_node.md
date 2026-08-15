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
                     ├──► Có missing_params? ──► [Hỏi lại User] (Bẻ luồng hỏi phụ)
                     │
                     └──► Tham số đầy đủ ──► [Bước 2: Python Calculator Tool]
                                                   │
                                                   └──► Cập nhật State.calculation_result
```

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

---

## 6. Graph Routing Logic

Nếu trích xuất đầy đủ tham số:

```python
return GenerationSynthesisNode() # Chuyển thẳng sang node tổng hợp, bỏ qua RAG
```

Nếu thiếu tham số (`len(missing_params) > 0`):

```python
return AskUserClarificationNode() # Bẻ luồng hỏi bổ sung tham số
```

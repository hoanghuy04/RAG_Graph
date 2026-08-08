# Luồng 5: `chat_calculation_result` — Tính Toán Học Vụ (Không Dùng RAG)

## Query Ví Dụ
> **"GPA học kỳ này của tôi là bao nhiêu?"**

---

## Bối Cảnh Người Dùng
- MSSV: `SV20247799` | Role: `SINH_VIEN` | Đơn vị: `KHOA_CNTT` | Level: `2`

---

## Điểm Đặc Biệt của Luồng Này

**Không đi qua RAG pipeline** (`QueryTransformationNode` → `RetrievalFilteringNode` → `PostRetrievalRerankNode`). Thay vào đó:
1. `MessageClassificationNode` → `academic_calculation` → route sang `CalculationNode`
2. `CalculationNode` gọi `CalculationExtractorAgent` (dùng `agents/calculation_extractor.yaml`) để trích xuất tham số
3. Python Calculator Tool tính toán trực tiếp từ DB điểm
4. `GenerationSynthesisNode` dùng `chat_calculation_result.yaml` để LLM diễn giải kết quả số thành ngôn ngữ tự nhiên

---

## Luồng Xử Lý Đầy Đủ (Node → Prompt → Input/Output)

---

### Node 01 — `GreetingDetectionNode` *(Deterministic)*
**Không dùng prompt.** Không phải chào → tiếp tục.

---

### Node 02 — `SecurityContextExtractionNode` *(JWT Decode)*
**Không dùng prompt.**

**State Output:**
```json
{
  "academic_security_context": {
    "user_id": "SV20247799",
    "role": "SINH_VIEN",
    "organization_scopes": ["KHOA_CNTT", "GLOBAL"],
    "max_access_level": 2,
    "student_program": "Công nghệ thông tin"
  }
}
```

---

### Node 03 — `MessageClassificationNode` *(LLM Agent)*
**Prompt template sử dụng:** `agents/message_classification.yaml`

**System Prompt (từ `message_classification.yaml`):**
```
## Intent Taxonomy
- academic_calculation: Yêu cầu tính toán số học liên quan đến học vụ.
  → "Em tính GPA học kỳ này được 3.2 không?", "18 TC ngành CNTT tốn bao nhiêu tiền?"

## Output Contract
{ "primary_intent": "academic_calculation", "routing_mode": null }
# routing_mode = null vì không cần Fan-out RAG
```

**Input:**
```
"GPA học kỳ này của tôi là bao nhiêu?"
```

**LLM Output:**
```json
{
  "primary_intent": "academic_calculation",
  "secondary_intents": [],
  "confidence": 0.96,
  "routing_mode": null
}
```

**State Update:**
```json
{ "intent_classification": { "primary_intent": "academic_calculation", "routing_mode": null } }
```

→ Chuyển sang `IntentRoutingNode`

---

### Node 04 — `IntentRoutingNode` *(Deterministic)*
**Không dùng prompt.**

```python
# intent = "academic_calculation"
return CalculationNode()
# KHÔNG gọi QueryTransformationNode hay RetrievalFilteringNode
```

---

### Node 08 — `CalculationNode` *(Calculator Tool + LLM Agent)*
**Prompt template sử dụng:** `agents/calculation_extractor.yaml`

Node này có **2 bước nội bộ**:

#### Bước 8a — Trích xuất tham số (LLM Agent)
**System Prompt (từ `calculation_extractor.yaml`):**
```
## Role
Bạn là Academic Calculation Parameter Extractor Agent.
Nhiệm vụ: Phân tích tin nhắn và trích xuất tham số cần thiết để tính toán học vụ.

## Calculation Types Supported
- gpa_calculation: Tính điểm trung bình học kỳ / tích lũy.
- credit_check: Tính tổng TC tích lũy, TC còn thiếu để tốt nghiệp.
- tuition_calculation: Tính học phí dựa trên số TC đăng ký và đơn giá ngành.

## Output Contract
{
  "calculation_type": "gpa_calculation" | "credit_check" | "tuition_calculation",
  "parameters": { "courses": [...], "program": "...", ... },
  "missing_params": ["<danh sách tham số còn thiếu>"]
}

## Fallback
Nếu missing_params không rỗng → hỏi lại người dùng trước khi tính.
```

**Input:**
```
"GPA học kỳ này của tôi là bao nhiêu?"
```

**LLM Output (Trường hợp sinh viên đã đăng nhập, hệ thống có dữ liệu điểm):**
```json
{
  "calculation_type": "gpa_calculation",
  "parameters": {
    "student_id": "SV20247799",
    "semester": "HK1/2024",
    "source": "DB_AUTO_FETCH"
  },
  "missing_params": []
}
```

> **Lưu ý:** `missing_params = []` vì sinh viên đã đăng nhập → hệ thống tự lấy dữ liệu điểm từ DB.
> Nếu sinh viên nhập điểm thủ công ("Toán 3TC 8.5đ, Lý 2TC 7.0đ..."), LLM sẽ parse ra `courses` array và `missing_params = []`.

#### Bước 8b — Python Calculator Tool chạy tính toán
Sau khi `calculation_extractor` xác nhận `missing_params = []`, **Python Calculator Tool** (không phải LLM) thực hiện tính toán:

```python
# Truy vấn DB điểm
courses = DB.get_grades(student_id="SV20247799", semester="HK1/2024")
# [
#   {"name": "Lập trình Web", "credits": 3, "grade": "A", "grade_point": 4.0},
#   {"name": "Cơ sở dữ liệu", "credits": 3, "grade": "B+", "grade_point": 3.5},
#   {"name": "Mạng máy tính", "credits": 3, "grade": "B",  "grade_point": 3.0},
#   {"name": "Tiếng Anh 3",   "credits": 2, "grade": "A-", "grade_point": 3.7}
# ]

# GPA = Σ(grade_point × credits) / Σ(credits)
# = (4.0×3 + 3.5×3 + 3.0×3 + 3.7×2) / (3+3+3+2)
# = (12.0 + 10.5 + 9.0 + 7.4) / 11
# = 38.9 / 11 = 3.536... ≈ 3.54
```

**State Update:**
```json
{
  "calculation_result": {
    "semester": "HK1/2024",
    "courses": [
      { "name": "Lập trình Web",  "credits": 3, "grade": "A",  "grade_point": 4.0 },
      { "name": "Cơ sở dữ liệu", "credits": 3, "grade": "B+", "grade_point": 3.5 },
      { "name": "Mạng máy tính",  "credits": 3, "grade": "B",  "grade_point": 3.0 },
      { "name": "Tiếng Anh 3",    "credits": 2, "grade": "A-", "grade_point": 3.7 }
    ],
    "semester_gpa": 3.54,
    "cumulative_gpa": 3.12,
    "total_credits_passed": 68
  }
}
```

→ Chuyển thẳng sang `GenerationSynthesisNode` (BỎ QUA Node 10, 11)

---

### Node 12 — `GenerationSynthesisNode` *(Fan-in LLM Agent)*
**Prompt template sử dụng:** `main/chat_calculation_result.yaml` + 4 khối `common/`

**Bước 12.1 — Xác định template chính:**
Luồng đến từ `academic_calculation` → `GenerationSynthesisNode` chọn `chat_calculation_result.yaml`:

```yaml
template: |
  {header}
  {academic_metadata}
  {security_access_control}
  {response_style}

  ## Chế Độ Hiển Thị Kết Quả Tính Toán Học Vụ
  Dưới đây là kết quả tính toán từ Calculator Tool:
  <calculation_result>
    {calculation_result}
  </calculation_result>

  Hãy diễn giải kết quả bằng ngôn ngữ tự nhiên, trang trọng,
  thêm nhận xét đánh giá ngắn gọn (VD: GPA 3.2/4.0 — Loại khá),
  và gợi ý câu hỏi tiếp theo liên quan nếu phù hợp.

  Câu hỏi gốc: {user_query}
```

> **Khác các template RAG:** Không có `{academic_domain_rules}`, `{citation_rules}`, `{prepared_context}` vì không dùng RAG — dữ liệu đến từ Calculator Tool đã xác thực.

**Bước 12.2 — Format các placeholder:**

| Placeholder | File nguồn | Giá trị điền |
|---|---|---|
| `{header}` | `common/header.yaml` | Giới thiệu AI Học vụ |
| `{academic_metadata}` | `common/academic_metadata.yaml` + State | SV20247799, SINH_VIEN, KHOA_CNTT, L2 |
| `{security_access_control}` | `common/security_access_control.yaml` | Phạm vi dữ liệu cá nhân |
| `{response_style}` | `common/response_style.yaml` | Trang trọng, nhận xét đánh giá |
| `{calculation_result}` | **State.calculation_result** (Python Tool) | JSON kết quả điểm HK1/2024 |
| `{user_query}` | Request gốc | "GPA học kỳ này của tôi là bao nhiêu?" |

**Bước 12.3 — System Prompt hoàn chỉnh:**
```
[HEADER] Bạn là Trợ Lý AI Học Vụ...
[ACADEMIC_METADATA] SV20247799 | SINH_VIEN | KHOA_CNTT | LEVEL_2
[SECURITY_ACCESS_CONTROL] ...
[RESPONSE_STYLE] ...

## Chế Độ Hiển Thị Kết Quả Tính Toán Học Vụ
Dưới đây là kết quả tính toán từ Calculator Tool:
<calculation_result>
  {
    "semester": "HK1/2024",
    "courses": [
      { "name": "Lập trình Web",  "credits": 3, "grade": "A",  "grade_point": 4.0 },
      { "name": "Cơ sở dữ liệu", "credits": 3, "grade": "B+", "grade_point": 3.5 },
      { "name": "Mạng máy tính",  "credits": 3, "grade": "B",  "grade_point": 3.0 },
      { "name": "Tiếng Anh 3",    "credits": 2, "grade": "A-", "grade_point": 3.7 }
    ],
    "semester_gpa": 3.54,
    "cumulative_gpa": 3.12,
    "total_credits_passed": 68
  }
</calculation_result>

Diễn giải bằng ngôn ngữ tự nhiên + nhận xét đánh giá + gợi ý tiếp theo.

Câu hỏi gốc: GPA học kỳ này của tôi là bao nhiêu?
```

**Bước 12.4 — LLM Output:**
> **Kết quả HK1/2024:**
>
> | Môn học | TC | Điểm | Hệ 4 |
> |---|---|---|---|
> | Lập trình Web | 3 | A | 4.0 |
> | Cơ sở dữ liệu | 3 | B+ | 3.5 |
> | Mạng máy tính | 3 | B | 3.0 |
> | Tiếng Anh 3 | 2 | A- | 3.7 |
>
> 🎓 **GPA học kỳ: 3.54/4.0 — Xếp loại Giỏi**
> 📊 **GPA tích lũy: 3.12/4.0 — Xếp loại Khá**
> 📚 **Tổng TC tích lũy: 68 TC**
>
> 💡 *GPA tích lũy 3.12 ≥ 3.0 → bạn đủ điều kiện đăng ký tối đa 28 TC học kỳ tới. Muốn xem lịch ĐKHP không?*

---

## Bản Đồ Node → Prompt Template

```
Node 01: GreetingDetectionNode      → Không dùng prompt (Regex)
Node 02: SecurityContextExtraction  → Không dùng prompt (JWT Decode)
Node 03: MessageClassificationNode  → agents/message_classification.yaml
Node 04: IntentRoutingNode          → Không dùng prompt (Logic Router)
Node 08: CalculationNode            → agents/calculation_extractor.yaml (bước 8a)
                                       + Python Calculator Tool (bước 8b, không LLM)
Node 12: GenerationSynthesisNode    → main/chat_calculation_result.yaml
                                       + 4 khối common/ (KHÔNG có prepared_context/citation)
```

## Sơ Đồ Luồng

```
"GPA học kỳ này của tôi là bao nhiêu?"
        │
[Node01] → không chào
[Node02] JWT → SV20247799, level=2
[Node03] message_classification.yaml → academic_calculation, routing_mode=null
[Node04] Router → CalculationNode  ← BỎ QUA QueryTransformation, VectorDB, Rerank
        │
[Node08a] calculation_extractor.yaml → { "gpa_calculation", missing_params=[] }
[Node08b] Python Calculator Tool → DB điểm → GPA=3.54, tích lũy=3.12, 68TC
        │
[Node12] chat_calculation_result.yaml + 4 common (không có RAG placeholders)
         System Prompt + calculation_result JSON → LLM chính
        │
✅ Bảng điểm đẹp + "Giỏi 3.54" + gợi ý câu hỏi tiếp
```

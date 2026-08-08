# Luồng 4: `chat_procedure_steps` — Hướng Dẫn Quy Trình Thủ Tục

## Query Ví Dụ
> **"Làm thủ tục bảo lưu kết quả học tập cần những bước gì?"**

---

## Bối Cảnh Người Dùng
- MSSV: `SV20219988` | Role: `SINH_VIEN` | Đơn vị: `KHOA_CNTT` | Level: `2`

---

## Điểm Đặc Biệt của Luồng Này

Intent `academic_procedure` được route sang `QueryTransformationNode(mode="PROCEDURE")`. Ở mode này, node KHÔNG sinh HyDE doc mà sinh query dạng **"Quy trình / Các bước để [hành động]"** để ưu tiên tìm tài liệu có cấu trúc tuần tự. Cuối cùng `GenerationSynthesisNode` dùng template `chat_procedure_steps.yaml` hướng dẫn LLM trình bày dạng Bước 1, Bước 2...

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
    "user_id": "SV20219988",
    "role": "SINH_VIEN",
    "organization_scopes": ["KHOA_CNTT", "GLOBAL"],
    "max_access_level": 2
  }
}
```

---

### Node 03 — `MessageClassificationNode` *(LLM Agent)*
**Prompt template sử dụng:** `agents/message_classification.yaml`

**System Prompt (từ `message_classification.yaml`):**
```
## Intent Taxonomy
- academic_procedure: Hỏi quy trình, các bước thực hiện thủ tục học vụ.
  → "Cách làm thủ tục bảo lưu học phần?", "Đăng ký vắng thi có cần gặp giáo vụ không?"
```

**Input:**
```
"Làm thủ tục bảo lưu kết quả học tập cần những bước gì?"
```

**LLM Output:**
```json
{
  "primary_intent": "academic_procedure",
  "secondary_intents": [],
  "confidence": 0.97,
  "routing_mode": "SINGLE"
}
```

**State Update:**
```json
{ "intent_classification": { "primary_intent": "academic_procedure", "routing_mode": "SINGLE" } }
```

→ Chuyển sang `IntentRoutingNode`

---

### Node 04 — `IntentRoutingNode` *(Deterministic)*
**Không dùng prompt.**

```python
# intent = "academic_procedure"
return QueryTransformationNode(mode="PROCEDURE")
```

---

### Node 06 — `QueryTransformationNode` *(LLM Agent, mode PROCEDURE)*
**Prompt template sử dụng:** `agents/hyde_generator.yaml`

Ở mode `PROCEDURE`, node vẫn dùng `hyde_generator.yaml` nhưng chỉ dẫn LLM tạo câu giả định theo dạng **"Quy trình / Các bước..."** thay vì dạng mô tả thông thường. Điều này giúp Dense Retrieval tìm được chunk tài liệu có cấu trúc tuần tự (Bước 1, Bước 2...).

**System Prompt (từ `hyde_generator.yaml`):**
```
## Role
Bạn là HyDE Generator Agent...

## Output Style
- Dùng: "Theo Quy chế...", "Sinh viên được phép...", "Các bước thực hiện..."
- Đoạn văn khẳng định sạch
```

**Input:**
```
"Làm thủ tục bảo lưu kết quả học tập cần những bước gì?"
```

**LLM Output — HyDE Document (dạng procedural):**
```
"Theo Hướng dẫn Thủ tục Học vụ của Phòng Đào tạo, quy trình bảo lưu kết
quả học tập bao gồm các bước: Sinh viên tải mẫu đơn P-02 từ cổng thông tin
sinh viên; Nộp hồ sơ tại Phòng Đào tạo kèm giấy tờ minh chứng lý do bảo
lưu trước ngày 15 của tháng đầu học kỳ; Phòng Đào tạo xét duyệt trong
vòng 05 ngày làm việc; Sinh viên nhận thông báo kết quả qua email."
```

**State Update:**
```json
{
  "transformed_queries": [
    "Theo hướng dẫn thủ tục học vụ, quy trình bảo lưu kết quả học tập: tải đơn P-02, nộp hồ sơ Phòng Đào tạo..."
  ]
}
```

→ Chuyển sang `RetrievalFilteringNode`

---

### Node 10 — `RetrievalFilteringNode` *(VectorDB, không dùng LLM)*
**Không dùng prompt.**

**Tầng 1 — Hard Pre-filter:**
```json
{ "must": [
  { "key": "doc_package", "match": { "any": ["KHOA_CNTT", "GLOBAL"] } },
  { "key": "min_access_level", "range": { "lte": 2 } }
]}
```

**Tầng 2 — Parallel Hybrid Search:**
- Dense: vector HyDE doc "quy trình bảo lưu" ↔ VectorDB
- BM25: keyword "bảo lưu", "Phòng Đào tạo", "đơn P-02", "học kỳ"

**Tầng 3 — RRF:** → 14 chunks thô

**State Output:**
```json
{
  "raw_retrieved_chunks": [
    { "chunk_id": "CHK_ĐTĐH_2023_016", "doc_name": "Quy chế ĐT 2023 — Điều 16: Bảo lưu", "rrf_score": 0.0401 },
    { "chunk_id": "CHK_HD_BAOLUU_2024", "doc_name": "Hướng dẫn Thủ tục Bảo lưu 2024 — P.ĐT", "rrf_score": 0.0387 },
    ... (12 chunks khác)
  ]
}
```

---

### Node 11 — `PostRetrievalRerankNode` *(Cross-Encoder)*
**Prompt template sử dụng:** `agents/reranker_compressor.yaml` *(config ngưỡng)*

**Cross-Encoder scoring:**
```
("Làm thủ tục bảo lưu cần những bước gì?", CHK_ĐTĐH_2023_016)  → score = 0.89 ✅
("Làm thủ tục bảo lưu cần những bước gì?", CHK_HD_BAOLUU_2024) → score = 0.93 ✅
... (12 chunks còn lại) → score < 0.70 ❌
```

**State Output:**
```json
{
  "has_valid_context": true,
  "filtered_context_chunks": [
    {
      "chunk_id": "CHK_ĐTĐH_2023_016",
      "doc_name": "Quy chế Đào tạo ĐH 2023 — Điều 16",
      "content": "Sinh viên có thể bảo lưu tối đa 2 học kỳ liên tiếp, tối đa 3 kỳ/khóa. Hồ sơ: đơn P-02 + giấy tờ minh chứng.",
      "rerank_score": 0.89
    },
    {
      "chunk_id": "CHK_HD_BAOLUU_2024",
      "doc_name": "Hướng dẫn Thủ tục Bảo lưu 2024 — Phòng Đào tạo",
      "content": "Bước 1: Tải mẫu P-02 tại portal. Bước 2: Nộp hồ sơ P.Đào tạo (P.201, NhàA) trước 15/tháng đầu kỳ. Bước 3: Xét duyệt 5 ngày. Bước 4: Nhận kết quả qua email.",
      "rerank_score": 0.93
    }
  ]
}
```

→ `has_valid_context = true` → Chuyển sang `GenerationSynthesisNode`

---

### Node 12 — `GenerationSynthesisNode` *(Fan-in LLM Agent)*
**Prompt template sử dụng:** `main/chat_procedure_steps.yaml` + 7 khối `common/`

**Bước 12.1 — Xác định template chính:**
Luồng đến từ `academic_procedure` → `GenerationSynthesisNode` chọn `chat_procedure_steps.yaml`:

```yaml
template: |
  {header}
  {academic_metadata}
  {security_access_control}
  {academic_domain_rules}
  {response_style}
  {citation_rules}
  {prepared_context}

  ## Chế Độ Hướng Dẫn Quy Trình Thủ Tục Học Vụ
  Câu hỏi về thủ tục / quy trình học vụ. Hãy trình bày theo dạng các bước tuần tự:
  1. Đánh số thứ tự rõ ràng (Bước 1, Bước 2...).
  2. Mỗi bước ghi rõ: Ai thực hiện, Làm gì, Nộp ở đâu / Liên hệ ai.
  3. Ghi chú thời gian xử lý dự kiến (nếu có trong tài liệu).
  4. Gắn chỉ số trích dẫn [1][2] và liệt kê nguồn ở cuối.

  Câu hỏi: {user_query}
```

> **Khác các template khác:** Có thêm chỉ dẫn cứng *"trình bày theo Bước 1, Bước 2... ghi rõ Ai, Làm gì, Ở đâu, Bao lâu"* — kiểm soát format output của LLM.

**Bước 12.2 — Format các placeholder:**

| Placeholder | File nguồn | Giá trị điền |
|---|---|---|
| `{header}` | `common/header.yaml` | Giới thiệu AI Học vụ |
| `{academic_metadata}` | `common/academic_metadata.yaml` + State | SV20219988, SINH_VIEN, KHOA_CNTT, L2 |
| `{security_access_control}` | `common/security_access_control.yaml` | Phạm vi KHOA_CNTT + GLOBAL |
| `{academic_domain_rules}` | `common/academic_domain_rules.yaml` | Ưu tiên tài liệu nội bộ |
| `{response_style}` | `common/response_style.yaml` | Trang trọng, danh sách rõ ràng |
| `{citation_rules}` | `common/citation_rules.yaml` | Gắn [1][2], liệt kê nguồn |
| `{prepared_context}` | `common/prepared_context.yaml` + State.filtered_chunks | 2 chunks bảo lưu |
| `{user_query}` | Request gốc | "Làm thủ tục bảo lưu cần những bước gì?" |

**Bước 12.3 — System Prompt hoàn chỉnh:**
```
[HEADER + METADATA + SECURITY + DOMAIN + STYLE + CITATION]...

[PREPARED_CONTEXT]
<academic_context>
  <chunk id="[1]">Quy chế ĐT 2023 — Điều 16: bảo lưu tối đa 2kỳ/lần, 3kỳ/khóa; đơn P-02</chunk>
  <chunk id="[2]">Hướng dẫn Bảo lưu 2024: Bước 1 tải P-02, Bước 2 nộp P.201 trước 15/tháng...Bước 4 nhận email</chunk>
</academic_context>

## Chế Độ Hướng Dẫn Quy Trình Thủ Tục Học Vụ
Trình bày theo dạng các bước tuần tự:
1. Đánh số Bước 1, Bước 2...
2. Ghi: Ai thực hiện, Làm gì, Nộp ở đâu.
3. Ghi chú thời gian xử lý.
4. Trích dẫn [1][2].

Câu hỏi: Làm thủ tục bảo lưu kết quả học tập cần những bước gì?
```

**Bước 12.4 — LLM Output:**
> **Bước 1 — Tải mẫu đơn** *(Sinh viên)*
> Tải mẫu **Đơn P-02** tại cổng thông tin sinh viên (portal) [2].
>
> **Bước 2 — Nộp hồ sơ** *(Sinh viên)*
> Nộp đơn P-02 + giấy tờ minh chứng tại **Phòng Đào tạo (P.201, Nhà A)**, trước ngày **15 của tháng đầu học kỳ** [2].
>
> **Bước 3 — Xét duyệt** *(Phòng Đào tạo)*
> Thời gian: **5 ngày làm việc** kể từ ngày nhận đủ hồ sơ [2].
>
> **Bước 4 — Nhận kết quả**
> Thông báo kết quả gửi qua **email sinh viên** [2].
>
> ⚠️ *Lưu ý: Tối đa 2 kỳ liên tiếp, không quá 3 kỳ/khóa* [1].

---

## Bản Đồ Node → Prompt Template

```
Node 01: GreetingDetectionNode      → Không dùng prompt (Regex)
Node 02: SecurityContextExtraction  → Không dùng prompt (JWT Decode)
Node 03: MessageClassificationNode  → agents/message_classification.yaml
Node 04: IntentRoutingNode          → Không dùng prompt (Logic Router)
Node 06: QueryTransformationNode    → agents/hyde_generator.yaml (mode=PROCEDURE)
Node 10: RetrievalFilteringNode     → Không dùng prompt (VectorDB)
Node 11: PostRetrievalRerankNode    → agents/reranker_compressor.yaml (config)
Node 12: GenerationSynthesisNode    → main/chat_procedure_steps.yaml
                                       + 7 khối common/
```

## Sơ Đồ Luồng

```
"thủ tục bảo lưu cần những bước gì?"
        │
[Node01] → không chào
[Node02] JWT → SV20219988, level=2
[Node03] message_classification.yaml → academic_procedure, SINGLE, 0.97
[Node04] Router → QueryTransformationNode(mode=PROCEDURE)
[Node06] hyde_generator.yaml (procedural) → HyDE "Bước 1 tải P-02, Bước 2 nộp..."
[Node10] VectorDB → Pre-filter → Dense+BM25 → RRF → 14 chunks
[Node11] reranker_compressor (config) → Cross-Encoder → 2 chunk ✅
[Node12] chat_procedure_steps.yaml + 7 common → chỉ dẫn "Bước 1, Bước 2..." → LLM
        │
✅ Trả lời 4 bước tuần tự: ai làm gì, nộp đâu, bao lâu, kèm [1][2]
```

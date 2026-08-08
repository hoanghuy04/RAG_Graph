# Luồng: `chat_academic_default` vs `chat_single_intent` — RAG Đơn Ý

## Tại Sao 2 Template Này Nằm Trong 1 File?

**Luồng node hoàn toàn giống nhau:**
```
Node01 → Node02 → Node03 → Node04 → Node06(HyDE) → Node10 → Node11 → Node12
```

Điểm khác biệt **duy nhất** nằm ở đoạn cuối của template được chọn trong **Node 12**:

| Template | Đoạn kết thúc |
|---|---|
| `chat_academic_default` | Chỉ có `## Câu Hỏi Của Người Dùng` → `{user_query}`. Không có chỉ dẫn đặc biệt |
| `chat_single_intent` | Thêm: *"Dựa vào bối cảnh trích xuất, hãy trả lời **trực diện, đầy đủ và chính xác vào câu hỏi đơn**"* |

`chat_single_intent` là phiên bản **tinh chỉnh hơn**: nhắc LLM đừng bị phân tâm, phải focus vào đúng 1 câu hỏi đơn.

---

## Query Ví Dụ

| Template | Query ví dụ |
|---|---|
| `chat_academic_default` | *"Sinh viên được phép đăng ký tối đa bao nhiêu tín chỉ mỗi học kỳ?"* |
| `chat_single_intent` | *"Điều kiện để được xét học bổng khuyến khích học tập là gì?"* |

---

## Bối Cảnh Người Dùng (dùng chung cho cả 2 ví dụ)
- Role: `SINH_VIEN` | Đơn vị: `KHOA_CNTT` | Level: `2`

---

## Luồng Xử Lý Đầy Đủ (dùng chung cho cả 2 template)

---

### Node 01 — `GreetingDetectionNode` *(Deterministic)*
**Không dùng prompt.** Regex kiểm tra → không phải chào hỏi → tiếp tục.

---

### Node 02 — `SecurityContextExtractionNode` *(JWT Decode)*
**Không dùng prompt.**

**State Output:**
```json
{
  "academic_security_context": {
    "user_id": "SV20248101",
    "role": "SINH_VIEN",
    "organization_scopes": ["KHOA_CNTT", "GLOBAL"],
    "max_access_level": 2
  }
}
```

---

### Node 03 — `MessageClassificationNode` *(LLM Agent)*
**Prompt template:** `agents/message_classification.yaml`

**System Prompt (từ `message_classification.yaml`):**
```
## Intent Taxonomy
- academic_advisory: Hỏi về quy chế, điều kiện, chính sách học vụ cụ thể.
  → "Điều kiện nhận học bổng loại giỏi là gì?", "Mấy điểm GPA thì bị cảnh báo học vụ?"

## Decision Rules
1. Ưu tiên academic_* nếu tin nhắn có bất kỳ liên quan học vụ.

## Output Contract
{ "primary_intent": "...", "routing_mode": "SINGLE" | "MULTI" | null }
```

**Ví dụ 1 — `chat_academic_default`:**
```
Input : "Sinh viên được phép đăng ký tối đa bao nhiêu tín chỉ mỗi học kỳ?"
Output: { "primary_intent": "academic_advisory", "routing_mode": "SINGLE", "confidence": 0.97 }
```

**Ví dụ 2 — `chat_single_intent`:**
```
Input : "Điều kiện để được xét học bổng khuyến khích học tập là gì?"
Output: { "primary_intent": "academic_advisory", "routing_mode": "SINGLE", "confidence": 0.96 }
```

→ Cả 2 đều ra `academic_advisory + SINGLE` → đi cùng nhánh tiếp theo.

---

### Node 04 — `IntentRoutingNode` *(Deterministic)*
**Không dùng prompt.**

```python
# Cả 2 ví dụ: intent = "academic_advisory", routing_mode = "SINGLE"
return QueryTransformationNode(mode="HYDE")
```

---

### Node 06 — `QueryTransformationNode` *(LLM Agent, mode HyDE)*
**Prompt template:** `agents/hyde_generator.yaml`

**System Prompt (từ `hyde_generator.yaml`):**
```
## Role
Bạn là HyDE Generator Agent cho hệ thống tra cứu quy chế đào tạo.

## Objective
Dựa trên câu hỏi thô, viết ra một đoạn văn bản câu trả lời giả định
mang văn phong hành chính, chuẩn mực của Quy chế đào tạo đại học.

## Output Style
- Dùng: "Theo Quy chế...", "Sinh viên được phép...", "Điều kiện bao gồm..."
- Đoạn văn khẳng định sạch, không chứa từ mập mờ
```

**Ví dụ 1 — HyDE Output:**
```
"Theo Quy chế Đào tạo Đại học Chính quy của Nhà trường, số tín chỉ tối
đa sinh viên được phép đăng ký trong một học kỳ chính là 25 tín chỉ.
Sinh viên có GPA từ 3.20 trở lên được đăng ký tối đa 28 tín chỉ.
Tối thiểu bắt buộc là 14 tín chỉ/kỳ, trừ học kỳ cuối khóa."
```

**Ví dụ 2 — HyDE Output:**
```
"Theo Quy chế Học bổng Khuyến khích Học tập, sinh viên được xét HBKKHT
khi đạt GPA học kỳ từ 3.20/4.00 trở lên, điểm rèn luyện xếp loại Tốt
(từ 80 điểm), và không bị kỷ luật từ mức Cảnh cáo trở lên."
```

→ Vector của HyDE doc (không phải câu hỏi gốc) được dùng để tìm kiếm VectorDB.

---

### Node 10 — `RetrievalFilteringNode` *(VectorDB)*
**Không dùng prompt.**

**Tầng 1 — Hard Pre-filter (Payload Filter trong Qdrant):**
```json
{ "must": [
  { "key": "doc_package", "match": { "any": ["KHOA_CNTT", "GLOBAL"] } },
  { "key": "min_access_level", "range": { "lte": 2 } }
]}
```

**Tầng 2 — Parallel Hybrid Search:**
- Dense Retrieval (vector HyDE ↔ VectorDB cosine similarity)
- Sparse Retrieval (BM25 keyword exact match)

**Tầng 3 — RRF Fusion:** → Top 15 chunks thô.

**State Output:**
```json
{
  "raw_retrieved_chunks": [
    { "chunk_id": "CHK_...", "doc_name": "...", "rrf_score": 0.03xx },
    ... (14 chunks khác)
  ]
}
```

---

### Node 11 — `PostRetrievalRerankNode` *(Cross-Encoder)*
**Prompt template:** `agents/reranker_compressor.yaml` *(config ngưỡng — không gọi LLM)*

Cross-Encoder nhận input **(câu hỏi GỐC ↔ từng chunk)** — dùng câu hỏi gốc, không phải HyDE doc:

**Ví dụ 1:**
```
("đăng ký tối đa bao nhiêu TC?", CHK_ĐTĐH_2023_012) → score=0.92 ✅
("đăng ký tối đa bao nhiêu TC?", CHK_ĐTĐH_2023_015) → score=0.61 ❌
...
```

**Ví dụ 2:**
```
("điều kiện xét học bổng KKHT?", CHK_HBKKHT_2023_05) → score=0.94 ✅
("điều kiện xét học bổng KKHT?", CHK_TB_HB_HK1_2024) → score=0.89 ✅
...
```

**State Output:**
```json
{
  "has_valid_context": true,
  "filtered_context_chunks": [ ...2 chunks sạch... ]
}
```

→ `has_valid_context = true` → Chuyển sang `GenerationSynthesisNode`

---

### Node 12 — `GenerationSynthesisNode` — ĐÂY LÀ ĐIỂM PHÂN NHÁNH DUY NHẤT

**Node 12 chọn template dựa trên `response_context` trong State.**

```python
# Trong GenerationSynthesisNode
if state.response_context == "SINGLE_ADVISORY_BASIC":
    template = templates.chat_academic_default
elif state.response_context == "SINGLE_ADVISORY_HYDE":
    template = templates.chat_single_intent
```

#### Trường hợp A — `chat_academic_default.yaml`

```yaml
template: |
  {header}
  {academic_metadata}
  {security_access_control}
  {academic_domain_rules}
  {response_style}
  {citation_rules}
  {prepared_context}

  ## Câu Hỏi Của Người Dùng
  {user_query}
  # ↑ Không có chỉ dẫn đặc biệt — LLM tự quyết định cách trả lời
```

**System Prompt kết quả (Ví dụ 1):**
```
[HEADER + METADATA + SECURITY + DOMAIN + STYLE + CITATION]
[PREPARED_CONTEXT]
<academic_context>
  <chunk id="[1]">Quy chế ĐT 2023 — Điều 12: tối đa 25TC / 28TC nếu GPA≥3.2</chunk>
  <chunk id="[2]">Hướng dẫn ĐKHP 2024: tối thiểu 14TC/kỳ</chunk>
</academic_context>

## Câu Hỏi Của Người Dùng
Sinh viên được phép đăng ký tối đa bao nhiêu tín chỉ mỗi học kỳ?
```

---

#### Trường hợp B — `chat_single_intent.yaml`

```yaml
template: |
  {header}
  {academic_metadata}
  {security_access_control}
  {academic_domain_rules}
  {response_style}
  {citation_rules}
  {prepared_context}

  ## Hướng Dẫn Xử Lý Single-Intent (HyDE Context)
  Dựa vào bối cảnh tài liệu trích xuất ở trên, hãy trả lời trực diện,
  đầy đủ và chính xác vào câu hỏi đơn của người dùng bên dưới.
  # ↑ Thêm câu nhắc: "trực diện, đúng 1 câu hỏi đơn"

  Câu hỏi: {user_query}
```

**System Prompt kết quả (Ví dụ 2):**
```
[HEADER + METADATA + SECURITY + DOMAIN + STYLE + CITATION]
[PREPARED_CONTEXT]
<academic_context>
  <chunk id="[1]">Quy chế HBKKHT 2023 — GPA≥3.2, rèn luyện Tốt, không kỷ luật</chunk>
  <chunk id="[2]">TB Xét HB HK1/2024 — hạn 15/11, nộp online</chunk>
</academic_context>

## Hướng Dẫn Xử Lý Single-Intent (HyDE Context)
Dựa vào bối cảnh tài liệu trích xuất ở trên, hãy trả lời trực diện,
đầy đủ và chính xác vào câu hỏi đơn của người dùng bên dưới.

Câu hỏi: Điều kiện để được xét học bổng khuyến khích học tập là gì?
```

---

## Tóm Tắt Điểm Khác Biệt Thực Sự

```
Cả 2 luồng: HOÀN TOÀN GIỐNG NHAU ở Node 01 → 02 → 03 → 04 → 06 → 10 → 11

Chỉ khác tại Node 12 — GenerationSynthesisNode:
┌─────────────────────────────┬──────────────────────────────────────┐
│   chat_academic_default     │       chat_single_intent             │
├─────────────────────────────┼──────────────────────────────────────┤
│ Dùng khi: câu hỏi đơn      │ Dùng khi: câu hỏi đơn + HyDE        │
│ thẳng, ngắn, rõ từ khóa    │ rõ hơn, nhiều tiêu chí               │
│                             │                                      │
│ Không có chỉ dẫn thêm      │ Thêm: "trả lời TRỰC DIỆN, ĐẦY ĐỦ   │
│ cho LLM                     │ vào câu hỏi ĐƠN" để LLM không bị   │
│                             │ phân tâm                             │
│ Câu kết thúc:               │ Câu kết thúc:                        │
│   "## Câu Hỏi Của NSD      │   "## Hướng Dẫn Single-Intent        │
│   {user_query}"             │   ...trực diện...{user_query}"       │
└─────────────────────────────┴──────────────────────────────────────┘

Thực tế: Đây là 2 biến thể nhỏ của cùng 1 luồng, không phải 2 luồng
         hoàn toàn khác nhau.
```

## Bản Đồ Node → Prompt Template (chung)

```
Node 01: GreetingDetectionNode      → Không dùng prompt
Node 02: SecurityContextExtraction  → Không dùng prompt (JWT Decode)
Node 03: MessageClassificationNode  → agents/message_classification.yaml
Node 04: IntentRoutingNode          → Không dùng prompt
Node 06: QueryTransformationNode    → agents/hyde_generator.yaml
Node 10: RetrievalFilteringNode     → Không dùng prompt (VectorDB)
Node 11: PostRetrievalRerankNode    → agents/reranker_compressor.yaml (config)
Node 12: GenerationSynthesisNode    → main/chat_academic_default.yaml  [biến thể A]
                                    → main/chat_single_intent.yaml     [biến thể B]
                                      + 7 khối common/ (giống nhau ở cả 2)
```

# Luồng 3: `chat_multi_intent_synthesis` — Tổng Hợp Đa Ý Song Song

## Query Ví Dụ
> **"Điều kiện xét tốt nghiệp và điều kiện được đăng ký học song hành khác nhau như thế nào?"**

---

## Bối Cảnh Người Dùng
- MSSV: `SV20223344` | Role: `SINH_VIEN` | Đơn vị: `KHOA_CNTT` | Level: `2`

---

## Tại Sao Đây Là Multi-Intent?
Câu hỏi chứa **2 vế rõ ràng cần tra cứu từ 2 tài liệu độc lập**:
1. Điều kiện xét tốt nghiệp
2. Điều kiện đăng ký học song hành

→ `routing_mode = MULTI` → Fan-out 2 luồng RAG song song → Fan-in tổng hợp bảng so sánh.

---

## Luồng Xử Lý Đầy Đủ (Node → Prompt → Input/Output)

---

### Node 01 — `GreetingDetectionNode` *(Deterministic)*
**Không dùng prompt.** Không phải chào hỏi → tiếp tục.

---

### Node 02 — `SecurityContextExtractionNode` *(JWT Decode)*
**Không dùng prompt.**

**State Output:**
```json
{
  "academic_security_context": {
    "user_id": "SV20223344",
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
- academic_advisory: Hỏi về quy chế, điều kiện, chính sách học vụ cụ thể.

## Decision Rules
4. Nếu một tin nhắn có 2 ý: 1 academic_advisory + 1 academic_advisory khác
   → routing_mode = "MULTI" vì cần tra cứu 2 nguồn độc lập.

## Output Contract
{ "primary_intent": "...", "routing_mode": "SINGLE" | "MULTI" | null }
```

**Input:**
```
"Điều kiện xét tốt nghiệp và điều kiện được đăng ký học song hành khác nhau như thế nào?"
```

**LLM Output:**
```json
{
  "primary_intent": "academic_advisory",
  "secondary_intents": [],
  "confidence": 0.93,
  "routing_mode": "MULTI"
}
```

**State Update:**
```json
{ "intent_classification": { "primary_intent": "academic_advisory", "routing_mode": "MULTI" } }
```

→ Chuyển sang `IntentRoutingNode`

---

### Node 04 — `IntentRoutingNode` *(Deterministic)*
**Không dùng prompt.**

```python
# routing_mode = "MULTI" + intent = "academic_advisory"
return QueryTransformationNode(mode="MULTI_QUERY_DECOMPOSITION")
```

---

### Node 06 — `QueryTransformationNode` *(LLM Agent, mode MULTI)*
**Prompt template sử dụng:** `agents/multi_query_decomposer.yaml`

Ở mode `MULTI`, node KHÔNG dùng `hyde_generator.yaml`. Thay vào đó, dùng `multi_query_decomposer.yaml` để phân rã câu hỏi phức hợp thành các sub-query độc lập.

**System Prompt (từ `multi_query_decomposer.yaml`):**
```
## Role
Bạn là Sub-Query Decomposition Agent trong hệ thống RAG đa truy vấn.

## Objective
Bẻ gãy câu hỏi phức hợp thành 2 đến 3 câu hỏi con độc lập, rõ ràng
để chuẩn bị cho tầng tìm kiếm song song vào VectorDB.

## Requirements
- Mỗi câu hỏi con PHẢI chứa đầy đủ thực thể và ngữ cảnh hành chính
  (không dùng đại từ "nó", "ngành đó").
- Trả về JSON: { "sub_queries": List[str] }

## Example
Input: "Điều kiện học song hành CNTT và Du lịch là gì, học phí chênh thế nào?"
Output: {
  "sub_queries": [
    "Quy định về điều kiện học vụ để đăng ký học song hành hai chương trình đào tạo",
    "Định mức học phí ngành Công nghệ thông tin theo khóa học",
    "Định mức học phí ngành Quản trị Dịch vụ Du lịch và Lữ hành theo khóa học"
  ]
}
```

**Input:**
```
"Điều kiện xét tốt nghiệp và điều kiện được đăng ký học song hành khác nhau như thế nào?"
```

**LLM Output:**
```json
{
  "sub_queries": [
    "Quy định về điều kiện học vụ để được xét tốt nghiệp chương trình đại học chính quy",
    "Quy định về điều kiện đăng ký học song hành hai chương trình đào tạo đại học"
  ]
}
```

**State Update:**
```json
{
  "transformed_queries": [
    "Quy định về điều kiện học vụ để được xét tốt nghiệp chương trình đại học chính quy",
    "Quy định về điều kiện đăng ký học song hành hai chương trình đào tạo đại học"
  ]
}
```

→ Chuyển sang `RetrievalFilteringNode` với 2 sub-queries

---

### Node 10 — `RetrievalFilteringNode` *(VectorDB, Fan-out Song Song)*
**Không dùng prompt.**

Node chạy **Fan-out**: mỗi sub-query được tìm kiếm **song song và độc lập** (`asyncio.gather`).

**Input:** 2 sub-queries + Pre-filter `{KHOA_CNTT, GLOBAL, level≤2}`

**Tầng 1 — Hard Pre-filter (áp dụng cho cả 2 nhánh):**
```json
{ "must": [
  { "key": "doc_package", "match": { "any": ["KHOA_CNTT", "GLOBAL"] } },
  { "key": "min_access_level", "range": { "lte": 2 } }
]}
```

**Tầng 2 — Parallel Hybrid Search × 2 sub-query:**
```
Sub-query 1: "...điều kiện tốt nghiệp..."
  → Dense Retrieval: vector sub-query 1 ↔ VectorDB
  → BM25: keyword "tốt nghiệp", "điều kiện", "GPA tích lũy"

Sub-query 2: "...điều kiện học song hành..."
  → Dense Retrieval: vector sub-query 2 ↔ VectorDB
  → BM25: keyword "học song hành", "chương trình thứ hai", "60 tín chỉ"
```

**Tầng 3 — RRF Fusion:** Gộp tất cả kết quả từ cả 2 nhánh → Top 20 chunks thô.

**State Output:**
```json
{
  "raw_retrieved_chunks": [
    { "chunk_id": "CHK_ĐTĐH_2023_038", "doc_name": "Quy chế ĐT 2023 — Điều 38: Tốt nghiệp", "rrf_score": 0.0412 },
    { "chunk_id": "CHK_ĐTĐH_2023_022", "doc_name": "Quy chế ĐT 2023 — Điều 22: Song hành", "rrf_score": 0.0389 },
    ... (18 chunks khác)
  ]
}
```

---

### Node 11 — `PostRetrievalRerankNode` *(Cross-Encoder)*
**Prompt template sử dụng:** `agents/reranker_compressor.yaml` *(config ngưỡng)*

**Lưu ý quan trọng:** Cross-Encoder so sánh **câu hỏi GỐC** (không phải sub-query) với từng chunk để đảm bảo chunk nào cũng liên quan đến câu hỏi tổng thể.

**Cross-Encoder scoring:**
```
Câu hỏi gốc: "Điều kiện tốt nghiệp và học song hành khác nhau thế nào?"

("...tốt nghiệp...", CHK_ĐTĐH_2023_038)  → score = 0.91 ✅
("...học song hành...", CHK_ĐTĐH_2023_022) → score = 0.88 ✅
("...học phí...", CHK_HP_2024_CNTT)       → score = 0.53 ❌
... (17 chunks còn lại) score < 0.70 ❌
```

**State Output:**
```json
{
  "has_valid_context": true,
  "filtered_context_chunks": [
    {
      "chunk_id": "CHK_ĐTĐH_2023_038",
      "doc_name": "Quy chế Đào tạo ĐH 2023 — Điều 38",
      "content": "Điều kiện tốt nghiệp: tích lũy đủ số TC chương trình; GPA tích lũy ≥ 2.0/4.0; hoàn thành nghĩa vụ học phí; rèn luyện không Kém.",
      "rerank_score": 0.91
    },
    {
      "chunk_id": "CHK_ĐTĐH_2023_022",
      "doc_name": "Quy chế Đào tạo ĐH 2023 — Điều 22",
      "content": "Điều kiện học song hành: đã tích lũy ≥ 60 TC chương trình chính; GPA tích lũy ≥ 2.5/4.0; không bị cảnh cáo học vụ.",
      "rerank_score": 0.88
    }
  ]
}
```

→ `has_valid_context = true` → Chuyển sang `GenerationSynthesisNode`

---

### Node 12 — `GenerationSynthesisNode` *(Fan-in LLM Agent)*
**Prompt template sử dụng:** `main/chat_multi_intent_synthesis.yaml` + 7 khối `common/`

**Bước 12.1 — Xác định template chính:**
Luồng đến từ `MULTI` routing → `GenerationSynthesisNode` chọn `chat_multi_intent_synthesis.yaml`:

```yaml
template: |
  {header}
  {academic_metadata}
  {security_access_control}
  {academic_domain_rules}
  {response_style}
  {citation_rules}
  {prepared_context}

  ## Hướng Dẫn Tổng Hợp Đa Truy Vấn (Multi-Intent Synthesis)
  Người dùng đang đưa ra một câu hỏi phức hợp gồm nhiều ý/thực thể khác nhau.
  Bối cảnh trong <academic_context> là tổng hợp kết quả truy xuất song song từ:
  {sub_queries_list}

  Nhiệm vụ:
  1. Tổng hợp thông tin hệ thống, dùng bảng so sánh (nếu có tiêu chí chênh lệch).
  2. Đảm bảo trả lời trọn vẹn từng vế trong câu hỏi gốc.
  3. Gắn chỉ số [1], [2] và liệt kê nguồn ở cuối.

  Câu hỏi gốc: {user_query}
```

**Bước 12.2 — Format các placeholder (bao gồm `{sub_queries_list}` đặc biệt):**

| Placeholder | File nguồn | Giá trị điền |
|---|---|---|
| `{header}` | `common/header.yaml` | Giới thiệu AI Học vụ |
| `{academic_metadata}` | `common/academic_metadata.yaml` + State | SV20223344, SINH_VIEN, KHOA_CNTT, L2 |
| `{security_access_control}` | `common/security_access_control.yaml` | Phạm vi KHOA_CNTT + GLOBAL |
| `{academic_domain_rules}` | `common/academic_domain_rules.yaml` | Ưu tiên tài liệu nội bộ |
| `{response_style}` | `common/response_style.yaml` | Trang trọng, dùng bảng khi so sánh |
| `{citation_rules}` | `common/citation_rules.yaml` | Gắn [1][2], liệt kê nguồn |
| `{prepared_context}` | `common/prepared_context.yaml` + State.filtered_chunks | 2 chunks (Điều 38 + Điều 22) |
| `{sub_queries_list}` | State.transformed_queries | "1. điều kiện tốt nghiệp\n2. điều kiện học song hành" |
| `{user_query}` | Request gốc | Câu hỏi gốc đầy đủ |

**Bước 12.3 — System Prompt hoàn chỉnh:**
```
[HEADER + METADATA + SECURITY + DOMAIN + STYLE + CITATION]...

[PREPARED_CONTEXT]
<academic_context>
  <chunk id="[1]">Quy chế ĐT 2023 — Điều 38: Tốt nghiệp: GPA≥2.0, đủ TC, học phí, rèn luyện không Kém</chunk>
  <chunk id="[2]">Quy chế ĐT 2023 — Điều 22: Song hành: ≥60TC, GPA≥2.5, không cảnh cáo học vụ</chunk>
</academic_context>

## Hướng Dẫn Tổng Hợp Đa Truy Vấn (Multi-Intent Synthesis)
Bối cảnh tổng hợp từ:
1. Quy định về điều kiện học vụ để được xét tốt nghiệp chương trình đại học chính quy
2. Quy định về điều kiện đăng ký học song hành hai chương trình đào tạo đại học

Nhiệm vụ: Tổng hợp + bảng so sánh + trích dẫn [1][2].

Câu hỏi gốc: Điều kiện xét tốt nghiệp và học song hành khác nhau thế nào?
```

**Bước 12.4 — LLM Output:**

| Tiêu chí | Xét Tốt Nghiệp [1] | Học Song Hành [2] |
|---|---|---|
| GPA tích lũy | ≥ 2.0/4.0 | ≥ 2.5/4.0 |
| Tín chỉ | Đủ số TC chương trình | ≥ 60 TC đã tích lũy |
| Học phí | Hoàn thành nghĩa vụ | Không yêu cầu |
| Kỷ luật | Rèn luyện không Kém | Không bị cảnh cáo học vụ |

**Nguồn:** [1] Quy chế ĐTĐH 2023, Điều 38 | [2] Quy chế ĐTĐH 2023, Điều 22

---

## Bản Đồ Node → Prompt Template

```
Node 01: GreetingDetectionNode      → Không dùng prompt (Regex)
Node 02: SecurityContextExtraction  → Không dùng prompt (JWT Decode)
Node 03: MessageClassificationNode  → agents/message_classification.yaml
Node 04: IntentRoutingNode          → Không dùng prompt (Logic Router)
Node 06: QueryTransformationNode    → agents/multi_query_decomposer.yaml  ← KHÁC với single
Node 10: RetrievalFilteringNode     → Không dùng prompt (VectorDB Fan-out × 2)
Node 11: PostRetrievalRerankNode    → agents/reranker_compressor.yaml (config)
Node 12: GenerationSynthesisNode    → main/chat_multi_intent_synthesis.yaml  ← KHÁC với single
                                       + 7 khối common/
                                       + {sub_queries_list} (placeholder đặc biệt)
```

## Sơ Đồ Luồng

```
"tốt nghiệp vs học song hành khác nhau thế nào?"
        │
[Node01] → không chào
[Node02] JWT → SV20223344, level=2
[Node03] message_classification.yaml → academic_advisory, MULTI, 0.93
[Node04] Router → QueryTransformationNode(MULTI_QUERY_DECOMPOSITION)
[Node06] multi_query_decomposer.yaml → 2 sub-queries độc lập
        │
      ┌─┴──────────────────────────────┐
      ▼                                ▼
[Node10] VectorDB Sub-query 1        [Node10] VectorDB Sub-query 2
Dense+BM25 "tốt nghiệp"             Dense+BM25 "học song hành"
      │                                │
      └──────────────┬─────────────────┘
                     ▼
              RRF Fusion → 20 chunks thô
                     │
[Node11] reranker_compressor.yaml (config)
         Cross-Encoder (câu gốc ↔ chunk) → 2 chunk ✅
                     │
[Node12] chat_multi_intent_synthesis.yaml + 7 common + sub_queries_list
         System Prompt → LLM chính
                     │
✅ Bảng so sánh 2 điều kiện, kèm [1][2]
```

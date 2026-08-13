# Luồng 8: `chat_ticket_fallback` — Không Đủ Context → Ticket Hỗ Trợ

## Query Ví Dụ
> **"Sinh viên nước ngoài diện học bổng Chính phủ có được miễn học phí không?"**

---

## Bối Cảnh Người Dùng
- MSSV: `SV20244455` | Role: `SINH_VIEN` | Đơn vị: `KHOA_CNTT` | Level: `2`

---

## Điểm Đặc Biệt của Luồng Này

Đây là **luồng phòng thủ cuối cùng** — câu hỏi hợp lệ về học vụ nhưng VectorDB không tìm được tài liệu vượt ngưỡng tin cậy. Thay vì để LLM **hallucinate** (tự bịa quy chế), hệ thống chuyển sang `TicketFallbackNode` và hướng dẫn người dùng tạo Ticket hỗ trợ.

**Điều kiện kích hoạt:** `PostRetrievalRerankNode` trả về `has_valid_context = false` (tất cả chunks đều có `rerank_score < 0.70`).

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
    "user_id": "SV20244455",
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
1. Ưu tiên academic_* nếu tin nhắn có bất kỳ liên quan nào đến học vụ.
```

**Input:**
```
"Sinh viên nước ngoài diện học bổng Chính phủ có được miễn học phí không?"
```

**LLM Output:**
```json
{
  "primary_intent": "academic_advisory",
  "secondary_intents": [],
  "confidence": 0.91,
  "routing_mode": "SINGLE"
}
```

**State Update:**
```json
{ "intent_classification": { "primary_intent": "academic_advisory", "routing_mode": "SINGLE" } }
```

→ Chuyển sang `IntentRoutingNode`

---

### Node 04 — `IntentRoutingNode` *(Deterministic)*
**Không dùng prompt.**

```python
# intent = "academic_advisory" + routing_mode = "SINGLE"
return QueryTransformationNode(mode="HYDE")
```

---

### Node 06 — `QueryTransformationNode` *(LLM Agent, mode HyDE)*
**Prompt template sử dụng:** `agents/hyde_generator.yaml`

**System Prompt (từ `hyde_generator.yaml`):**
```
## Role
Bạn là HyDE Generator Agent...
## Output Style
- Dùng: "Theo Quy chế...", "Sinh viên được phép...", "Điều kiện bao gồm..."
```

**Input:**
```
"Sinh viên nước ngoài diện học bổng Chính phủ có được miễn học phí không?"
```

**LLM Output — HyDE Document:**
```
"Theo Quy chế Học phí và Chính sách Miễn giảm Học phí của Nhà trường,
sinh viên nước ngoài được tiếp nhận theo diện Học bổng Chính phủ
(Government Scholarship) được miễn toàn bộ học phí trong thời gian
đào tạo theo quy định của Bộ Giáo dục và Đào tạo."
```

**State Update:**
```json
{
  "transformed_queries": [
    "Theo Quy chế học phí, sinh viên nước ngoài diện học bổng Chính phủ được miễn học phí..."
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
- Dense: vector HyDE "sinh viên nước ngoài học bổng Chính phủ miễn học phí"
- BM25: keyword "sinh viên nước ngoài", "học bổng Chính phủ", "miễn học phí"

> **Vấn đề:** Chủ đề này rất đặc thù. Trong VectorDB nội bộ không có tài liệu quy chế về sinh viên nước ngoài diện học bổng Chính phủ (đây là chính sách cấp Bộ, không có trong quy chế của trường).

**Tầng 3 — RRF Fusion:** → 5 chunks thô với score thấp

**State Output:**
```json
{
  "raw_retrieved_chunks": [
    { "chunk_id": "CHK_HP_2024_001", "doc_name": "Quy chế Học phí 2024 — Điều 8", "rrf_score": 0.0089 },
    { "chunk_id": "CHK_HP_2024_003", "doc_name": "Biểu phí 2024-2025 — Mục 2", "rrf_score": 0.0071 },
    ... (3 chunks khác, rrf_score rất thấp)
  ]
}
```

---

### Node 11 — `PostRetrievalRerankNode` *(Cross-Encoder)*
**Prompt template sử dụng:** `agents/reranker_compressor.yaml` *(config ngưỡng)*

**Cross-Encoder scoring — tất cả đều thất bại:**
```
("SV nước ngoài diện HB Chính phủ có miễn học phí không?", CHK_HP_2024_001) → score = 0.52 ❌
("SV nước ngoài diện HB Chính phủ có miễn học phí không?", CHK_HP_2024_003) → score = 0.41 ❌
("SV nước ngoài diện HB Chính phủ có miễn học phí không?", ...)              → score < 0.70 ❌

→ valid_chunks = []  (không có chunk nào vượt ngưỡng 0.70)
```

**State Output:**
```json
{
  "has_valid_context": false,
  "filtered_context_chunks": []
}
```

**Conditional Routing:**
```python
if state.has_valid_context:
    return GenerationSynthesisNode()
else:
    return TicketFallbackNode()  # ← Kích hoạt luồng fallback
```

→ `has_valid_context = false` → Chuyển sang **`TicketFallbackNode`** (Node 13)

---

### Node 13 — `TicketFallbackNode` *(Deterministic + Template)*
**Không gọi LLM.** Node này build response từ **template tĩnh** (không phải YAML agent prompt) và State.

**Nguyên tắc tuyệt đối:** Không để LLM tự đoán quy chế khi không có nguồn tri thức xác nhận → Zero Hallucination.

**Input State:**
```json
{
  "user_query": "Sinh viên nước ngoài diện học bổng Chính phủ có được miễn học phí không?",
  "has_valid_context": false,
  "academic_security_context": {
    "user_id": "SV20244455",
    "role": "SINH_VIEN",
    "organization_scopes": ["KHOA_CNTT", "GLOBAL"]
  }
}
```

**Action Payload Output (JSON, không phải text):**
```json
{
  "response_type": "TICKET_FALLBACK",
  "message_text": "Hiện tại hệ thống chưa tìm thấy quy định hoặc hướng dẫn
    cụ thể về trường hợp này trong kho tài liệu học vụ chính thức của Nhà trường.\n\n
    Để được giải đáp chính xác, bạn có thể tạo một Ticket hỗ trợ học vụ
    gửi trực tiếp đến Phòng Đào tạo / Phòng Công tác Sinh viên.",
  "action_buttons": [
    {
      "label": "Tạo Ticket Hỗ Trợ Học Vụ",
      "action_type": "CREATE_TICKET_MODAL",
      "payload": {
        "prefill_subject": "Chính sách học phí sinh viên nước ngoài diện học bổng Chính phủ",
        "prefill_department": "PHONG_DAOTAO",
        "user_id": "SV20244455",
        "original_query": "Sinh viên nước ngoài diện học bổng Chính phủ có được miễn học phí không?"
      }
    }
  ]
}
```

---

### Node 12 — `GenerationSynthesisNode` *(Fan-in LLM Agent)* — Tùy chọn
**Prompt template sử dụng:** `main/chat_ticket_fallback.yaml` + 4 khối `common/`

Trong một số triển khai, `TicketFallbackNode` có thể vẫn gọi LLM để tạo **thông điệp fallback tự nhiên hơn** (thay vì template cứng). Khi đó, `GenerationSynthesisNode` được gọi với `chat_ticket_fallback.yaml`:

```yaml
template: |
  {header}
  {academic_metadata}
  {security_access_control}
  {response_style}
  {ticket_fallback}

  ## Thông Báo Người Dùng
  Không tìm thấy tài liệu quy chế phù hợp vượt ngưỡng tin cậy cho câu hỏi:
  "{user_query}".
  Hãy tạo phản hồi fallback chuẩn mực và hướng dẫn người dùng nhấn nút
  tạo Ticket hỗ trợ.
```

> **Khác tất cả template khác:** Có `{ticket_fallback}` từ `common/ticket_fallback.yaml` (hướng dẫn LLM nói về nút Ticket) — KHÔNG có `{academic_domain_rules}`, `{citation_rules}`, `{prepared_context}`.

**Format các placeholder:**

| Placeholder | File nguồn | Giá trị điền |
|---|---|---|
| `{header}` | `common/header.yaml` | Giới thiệu AI Học vụ |
| `{academic_metadata}` | `common/academic_metadata.yaml` + State | SV20244455, SINH_VIEN, KHOA_CNTT, L2 |
| `{security_access_control}` | `common/security_access_control.yaml` | Phạm vi truy cập |
| `{response_style}` | `common/response_style.yaml` | Lịch sự, hướng dẫn rõ ràng |
| `{ticket_fallback}` | `common/ticket_fallback.yaml` | Hướng dẫn: khi không có data → gợi ý tạo Ticket |
| `{user_query}` | Request gốc | Câu hỏi gốc |

**LLM Output (nếu có gọi LLM):**
> Xin lỗi bạn, hiện tại tôi **chưa tìm thấy quy định cụ thể** về chính sách học phí cho sinh viên nước ngoài diện học bổng Chính phủ trong cơ sở tài liệu học vụ của trường.
>
> 👉 **Vui lòng nhấn nút "Tạo Ticket Hỗ trợ"** để gửi câu hỏi đến **Phòng Đào tạo**. Bộ phận phụ trách sẽ phản hồi trong **1-2 ngày làm việc**.
>
> *Liên hệ trực tiếp: Phòng Đào tạo — P.201, Nhà A.*

---

## Bản Đồ Node → Prompt Template

```
Node 01: GreetingDetectionNode      → Không dùng prompt (Regex)
Node 02: SecurityContextExtraction  → Không dùng prompt (JWT Decode)
Node 03: MessageClassificationNode  → agents/message_classification.yaml
Node 04: IntentRoutingNode          → Không dùng prompt (Logic Router)
Node 06: QueryTransformationNode    → agents/hyde_generator.yaml
Node 10: RetrievalFilteringNode     → Không dùng prompt (VectorDB)
Node 11: PostRetrievalRerankNode    → agents/reranker_compressor.yaml (config)
         → has_valid_context = FALSE → chuyển hướng!
Node 13: TicketFallbackNode         → Không dùng prompt (Template tĩnh / JSON)
Node 12: GenerationSynthesisNode    → main/chat_ticket_fallback.yaml  [tùy chọn]
                                       + 4 khối common/ + {ticket_fallback}
```

## Sơ Đồ Luồng

```
"SV nước ngoài diện HB Chính phủ có miễn học phí không?"
        │
[Node01] → không chào
[Node02] JWT → SV20244455, level=2
[Node03] message_classification.yaml → academic_advisory, SINGLE, 0.91
[Node04] Router → QueryTransformationNode(HYDE)
[Node06] hyde_generator.yaml → HyDE doc "...sinh viên nước ngoài HB Chính phủ miễn học phí..."
[Node10] VectorDB → Pre-filter → Dense+BM25 → RRF → 5 chunks thô (rrf_score thấp)
[Node11] reranker_compressor (config) → Cross-Encoder
         → TẤT CẢ score < 0.70 → has_valid_context = FALSE
         │
         ▼ RẼNHÁNH KHÁC
[Node13] TicketFallbackNode → JSON payload {message_text, action_buttons}
         └── [Tùy chọn] chat_ticket_fallback.yaml → LLM tạo thông điệp tự nhiên hơn
         │
✅ UI hiển thị: "Không tìm thấy..." + Nút [Tạo Ticket Hỗ trợ]
```

---

## Tóm Tắt Toàn Bộ 7 Luồng — Node → Prompt Mapping

> **Cập nhật gộp flow (2026-08)**: Advisory/Single/Procedure/Document/Calendar nay dùng chung 1 flow (node 06 → 10 → 11 → 12 với template `chat_academic_advisory`). Không còn `CalendarLookupNode` (09) / Calendar DB riêng — Calendar chỉ là một dạng câu hỏi HyDE+RAG như Advisory, không có bảng riêng nữa.

| Node | 01 Advisory (đơn/HyDE, gồm cả Calendar) | 03 Multi | 04 Procedure/Document | 05 Calculation | 07 Direct | 08 Fallback |
|---|---|---|---|---|---|---|
| **03 Classification** | `message_classification` | ← | ← | ← | ← | ← |
| **06 QueryTransform** | `hyde_generator` | `multi_query_decomposer` | `hyde_generator` (PROC/DOC) | ❌ | ❌ | `hyde_generator` |
| **08 Calculation** | ❌ | ❌ | ❌ | `calculation_extractor` | ❌ | ❌ |
| **10 Retrieval** | VectorDB | ← Fan-out×2 | ← | ❌ | ❌ | VectorDB |
| **11 Rerank** | `reranker_compressor` | ← | ← | ❌ | ❌ | `reranker_compressor` → **FALSE** |
| **12/05A Gen.** | `chat_academic_advisory` | `chat_multi_intent_synthesis` | `chat_academic_advisory` | `chat_calculation_result` | `chat_direct_llm` (05A) | `chat_ticket_fallback` |

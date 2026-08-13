# Luồng 7: `chat_direct_llm` — Trả Lời Trực Tiếp Không Cần RAG

## Query Ví Dụ
> **"Tín chỉ (credit) là gì? Khác gì với đơn vị học trình?"**

---

## Bối Cảnh Người Dùng
- MSSV: `SV20246677` | Role: `SINH_VIEN` | Đơn vị: `KHOA_CNTT` | Level: `2`

---

## Điểm Đặc Biệt của Luồng Này

Đây là luồng **đơn giản nhất** — chỉ cần 5 node (01→02→03→04→05A→12), bỏ qua hoàn toàn:
- `QueryTransformationNode` (không cần biến đổi query)
- `RetrievalFilteringNode` (không cần VectorDB)
- `PostRetrievalRerankNode` (không có chunks để rerank)
- `CalculationNode` (không phải câu hỏi tính toán)

`DirectLLMNode` (Node 05A) nhận câu hỏi và **gọi thẳng LLM với system prompt `chat_direct_llm.yaml`** mà không cần `{prepared_context}`.

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
    "user_id": "SV20246677",
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
- general_knowledge: Câu hỏi kiến thức phổ thông không cần tra quy chế nhà trường.
  → "1+1 bằng mấy?", "Python là gì?", "Thủ đô Hà Nội ở đâu?"

## Decision Rules
3. Trả general_knowledge chỉ khi câu hỏi CHẮC CHẮN không cần tra quy chế nhà trường.
   (Câu hỏi về khái niệm chung trong giáo dục, không hỏi về chính sách riêng của trường)

## Output Contract
{ "primary_intent": "general_knowledge", "routing_mode": null }
# routing_mode = null → không cần Fan-out
```

**Input:**
```
"Tín chỉ (credit) là gì? Khác gì với đơn vị học trình?"
```

**LLM Output:**
```json
{
  "primary_intent": "general_knowledge",
  "secondary_intents": [],
  "confidence": 0.94,
  "routing_mode": null
}
```

> **Tại sao không phải `academic_advisory`?** Nếu câu hỏi là *"Trường tôi quy định tín chỉ như thế nào?"* → sẽ là `academic_advisory` → RAG. Câu hỏi này hỏi **khái niệm chung** (tín chỉ là gì theo tiêu chuẩn giáo dục) → `general_knowledge`.

**State Update:**
```json
{ "intent_classification": { "primary_intent": "general_knowledge", "routing_mode": null } }
```

→ Chuyển sang `IntentRoutingNode`

---

### Node 04 — `IntentRoutingNode` *(Deterministic)*
**Không dùng prompt.**

```python
# intent = "general_knowledge"
return DirectLLMNode()
# KHÔNG gọi bất kỳ node nào trong RAG pipeline
```

---

### Node 05A — `DirectLLMNode` *(LLM trực tiếp, Tier 1)*
**Prompt template sử dụng:** `main/chat_direct_llm.yaml` + 4 khối `common/`

Node này **không phải** `GenerationSynthesisNode` — nó là `DirectLLMNode` riêng biệt. Tuy nhiên cơ chế build prompt giống nhau: lấy template chính từ `main/` và điền các khối `common/`.

**Template chính `chat_direct_llm.yaml`:**
```yaml
template: |
  {header}
  {academic_metadata}
  {security_access_control}
  {response_style}

  ## Chế Độ Trả Lời Trực Tiếp (Direct Answer Mode)
  Câu hỏi thuộc loại kiến thức phổ thông, không yêu cầu tra cứu tài liệu
  quy chế nội bộ của Nhà trường. Hãy trả lời ngắn gọn, chính xác và lịch sự.

  Nếu câu hỏi vừa có yếu tố kiến thức phổ thông VỪA có liên quan đến học vụ,
  hãy trả lời phần kiến thức phổ thông và gợi ý người dùng đặt câu hỏi
  học vụ cụ thể hơn.

  Câu hỏi của người dùng: {user_query}
```

> **Không có:** `{academic_domain_rules}`, `{citation_rules}`, `{prepared_context}` — vì không dùng RAG, không cần quy tắc domain hay trích dẫn.

**Format các placeholder:**

| Placeholder | File nguồn | Giá trị điền |
|---|---|---|
| `{header}` | `common/header.yaml` | Giới thiệu AI Học vụ |
| `{academic_metadata}` | `common/academic_metadata.yaml` + State | SV20246677, SINH_VIEN, KHOA_CNTT, L2 |
| `{security_access_control}` | `common/security_access_control.yaml` | Phạm vi truy cập |
| `{response_style}` | `common/response_style.yaml` | Trang trọng, ngắn gọn, chính xác |
| `{user_query}` | Request gốc | "Tín chỉ là gì? Khác gì đơn vị học trình?" |

**System Prompt hoàn chỉnh:**
```
[HEADER] Bạn là Trợ Lý AI Học Vụ...
[ACADEMIC_METADATA] SV20246677 | SINH_VIEN | KHOA_CNTT | LEVEL_2
[SECURITY_ACCESS_CONTROL] ...
[RESPONSE_STYLE] ...

## Chế Độ Trả Lời Trực Tiếp (Direct Answer Mode)
Câu hỏi thuộc kiến thức phổ thông, không cần tra cứu tài liệu quy chế nội bộ.
Trả lời ngắn gọn, chính xác, lịch sự.

Nếu vừa có yếu tố phổ thông VỪA học vụ → trả lời phổ thông + gợi ý hỏi thêm.

Câu hỏi: Tín chỉ (credit) là gì? Khác gì với đơn vị học trình?
```

**LLM Output:**
> **Tín chỉ (Credit)** là đơn vị đo khối lượng học tập. 1 tín chỉ = 15 tiết học lý thuyết (hoặc tương đương với thực hành/tự học).
>
> **Đơn vị học trình (ĐVHT)** là khái niệm cũ hơn, phổ biến trước năm 2007. Về bản chất, 1 ĐVHT ≈ 1 tín chỉ, đều = 15 tiết, nhưng cách tính điểm tích lũy và cấu trúc chương trình có thể khác nhau tùy hệ thống.
>
> 💡 *Nếu bạn muốn biết số tín chỉ tối đa/tối thiểu theo quy chế riêng của trường, tôi có thể tra cứu ngay!*

---

## Bản Đồ Node → Prompt Template

```
Node 01: GreetingDetectionNode      → Không dùng prompt (Regex)
Node 02: SecurityContextExtraction  → Không dùng prompt (JWT Decode)
Node 03: MessageClassificationNode  → agents/message_classification.yaml
Node 04: IntentRoutingNode          → Không dùng prompt (Logic Router)
Node 05A: DirectLLMNode             → main/chat_direct_llm.yaml
                                       + 4 khối common/ (KHÔNG có domain_rules/citation/prepared_context)

[Node 06, 10, 11, 12 — BỎ QUA HOÀN TOÀN]
```

## So Sánh 5 Luồng: Node nào dùng Prompt nào?

| Node | chat_academic_default | chat_single_intent | chat_multi_intent | chat_procedure | chat_direct_llm |
|---|---|---|---|---|---|
| 03 MessageClassification | `message_classification.yaml` | ← | ← | ← | ← |
| 06 QueryTransformation | `hyde_generator.yaml` | `hyde_generator.yaml` | `multi_query_decomposer.yaml` | `hyde_generator.yaml` (PROCEDURE) | ❌ bỏ qua |
| 10 RetrievalFiltering | *(VectorDB — không prompt)* | ← | ← Fan-out × 2 | ← | ❌ bỏ qua |
| 11 PostRetrievalRerank | `reranker_compressor.yaml` | ← | ← | ← | ❌ bỏ qua |
| 12/05A Generation | `chat_academic_default.yaml` | `chat_single_intent.yaml` | `chat_multi_intent_synthesis.yaml` | `chat_procedure_steps.yaml` | `chat_direct_llm.yaml` (Node 05A) |

## Sơ Đồ Luồng

```
"Tín chỉ là gì? Khác gì đơn vị học trình?"
        │
[Node01] → không chào
[Node02] JWT → SV20246677, level=2
[Node03] message_classification.yaml → general_knowledge, routing_mode=null
[Node04] Router → DirectLLMNode  ← BỎ QUA mọi node RAG
        │
[Node05A] chat_direct_llm.yaml + 4 common → System Prompt → LLM trực tiếp
        │
✅ Giải thích tín chỉ ≈ ĐVHT, gợi ý hỏi thêm quy chế trường
```

# Detail: MessageClassificationNode (Node 03)

## 1. Vai Trò & Đặc Điểm

`MessageClassificationNode` là "bộ não điều phối" của toàn bộ Graph. Node này nhận câu hỏi thô từ người dùng (đã qua lớp GreetingDetection + SecurityContext) và thực hiện **phân loại vào đúng 1 trong 9 intent types** để IntentRoutingNode định hướng luồng xử lý chi phí tối ưu nhất.

Node chạy **đồng thời** với SecurityContextExtractionNode thông qua `asyncio.gather` để tối thiểu độ trễ.

---

## 2. Taxonomy 9 Intent Types — Phân Tích Quyết Định

### Nhóm Fast Path (Tier 0-1, không cần RAG)
| Intent | Trigger | Route đến |
| :--- | :--- | :--- |
| `social_chat` | Phản ứng xã giao ngắn, không có câu hỏi | Template tĩnh |
| `general_knowledge` | Kiến thức phổ thông (toán, địa lý, định nghĩa) | DirectLLMNode |
| `off_topic` | Hoàn toàn ngoài học vụ (giá vàng, nấu ăn) | OffTopicRejectNode |

### Nhóm Tool-Based (Tier 2-3, không cần VectorDB)
| Intent | Trigger | Route đến |
| :--- | :--- | :--- |
| `academic_calculation` | Có số liệu cụ thể, cần tính GPA/học phí/TC | CalculationNode |
| `academic_calendar` | Hỏi ngày tháng, lịch biểu cụ thể | CalendarLookupNode |

### Nhóm RAG Pipeline (Tier 4-5, cần VectorDB)
| Intent | RAG Mode | Route đến |
| :--- | :--- | :--- |
| `academic_advisory` | SINGLE hoặc MULTI | QueryTransformationNode |
| `academic_comparison` | MULTI — Parallel Fan-out | AcademicComparisonNode → Retrieval |
| `academic_procedure` | PROCEDURE mode | QueryTransformationNode |
| `academic_document` | DOCUMENT mode | QueryTransformationNode |

---

## 3. Toàn Bộ System Prompt (`message_classification.yaml`)

```yaml
# agents/message_classification.yaml
# Xem đầy đủ tại: Flow/KLTN/prompt_template/agents/message_classification.yaml
```

---

## 4. Xử Lý Đa Intent (Multi-Intent Support)

Tương tự như LISA hỗ trợ `intents: List[Intent]`, `MessageClassificationNode` hỗ trợ `secondary_intents`:

**Ví dụ đa intent**:
- Input: *"Ngành CNTT học phí bao nhiêu, và bao giờ hết hạn đóng học phí?"*
  → `primary_intent: "academic_comparison"`, `secondary_intents: ["academic_calendar"]`
  → Graph xử lý Comparison RAG trước, sau đó gọi CalendarLookupNode, gộp vào GenerationSynthesisNode

---

## 5. Edge Routing Code

```python
async def run_intent_routing_node(state: AcademicChatState) -> BaseNode:
    intent = state.classification_result.primary_intent
    mode   = state.classification_result.routing_mode
    
    routing_map = {
        "social_chat":           lambda: EndSocialTemplate(),
        "general_knowledge":     lambda: DirectLLMNode(),
        "off_topic":             lambda: OffTopicRejectNode(),
        "academic_advisory":     lambda: QueryTransformationNode(mode=mode or "SINGLE"),
        "academic_comparison":   lambda: AcademicComparisonNode(),
        "academic_calculation":  lambda: CalculationNode(),
        "academic_procedure":    lambda: QueryTransformationNode(mode="PROCEDURE"),
        "academic_calendar":     lambda: CalendarLookupNode(),
        "academic_document":     lambda: QueryTransformationNode(mode="DOCUMENT"),
    }
    
    handler = routing_map.get(intent, lambda: QueryTransformationNode(mode="SINGLE"))
    return handler()
```

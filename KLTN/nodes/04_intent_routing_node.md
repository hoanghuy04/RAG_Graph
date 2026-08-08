# 04. IntentRoutingNode (Định Tuyến Logic)

## 1. Mạch Hoạt Động & Vai Trò Trong Graph
`IntentRoutingNode` là node mang tính chất **Deterministic (Logic Router)**, không sử dụng LLM. Node này đọc trạng thái `intent_classification` từ State và thực hiện chuyển hướng Graph sang Node xử lý chuyên biệt tiếp theo.

Việc định tuyến deterministic giúp tiết kiệm token, giảm độ trễ và phân phối tác vụ chính xác về các nhánh con.

---

## 2. Bảng Định Tuyến Logic (Routing Table)

Hệ thống định tuyến theo sơ đồ sau:

```python
ROUTING_MAP = {
    "social_chat":           "EndSocialTemplate",         # Trả về câu xã giao tĩnh lập tức
    "general_knowledge":     "DirectLLMNode",             # Trả lời trực tiếp bằng LLM (không RAG)
    "off_topic":             "OffTopicRejectNode",        # Từ chối khéo léo (Deterministic template)
    "academic_advisory":     "QueryTransformationNode",   # Nhánh RAG đơn/đa ý
    "academic_comparison":   "AcademicComparisonNode",    # Nhánh so sánh song song
    "academic_calculation":  "CalculationNode",           # Nhánh tính toán (GPA, học phí)
    "academic_procedure":    "QueryTransformationNode",   # Nhánh RAG hướng quy trình
    "academic_calendar":     "CalendarLookupNode",        # Nhánh tra cứu lịch DB
    "academic_document":     "QueryTransformationNode"    # Nhánh RAG hướng dẫn giấy tờ
}
```

---

## 3. Input / Output Schema

### Input State
```json
{
  "intent_classification": {
    "primary_intent": "academic_calculation",
    "routing_mode": null,
    "confidence": 0.96
  }
}
```

### Chuyển Tiếp Graph (Graph Target)
```python
# Ví dụ với input trên:
return CalculationNode()
```

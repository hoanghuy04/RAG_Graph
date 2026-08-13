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
    "academic_advisory":     "QueryTransformationNode",   # Flow Advisory hợp nhất
    "academic_comparison":   "AcademicComparisonNode",    # Nhánh so sánh song song (KHÔNG gộp — khác cấu trúc output: bảng so sánh)
    "academic_calculation":  "CalculationNode",           # Nhánh tính toán (GPA, học phí) (KHÔNG gộp — Tool-based, không RAG)
    "academic_procedure":    "QueryTransformationNode",   # Flow Advisory hợp nhất
    "academic_calendar":     "QueryTransformationNode",   # Flow Advisory hợp nhất (trước đây route riêng sang CalendarLookupNode)
    "academic_document":     "QueryTransformationNode"    # Flow Advisory hợp nhất
}
```

> **Cập nhật gộp flow (2026-08)**: `academic_advisory`, `academic_procedure`, `academic_document`, `academic_calendar` nay đều trỏ chung về `QueryTransformationNode` (06) để giảm số nhánh routing từ 4 xuống 1. KLTN không quản lý DB lịch học riêng — `academic_calendar` chỉ là một dạng câu hỏi advisory khác, dùng chung pipeline HyDE + RAG, không có Tool/DB riêng — xem [`06_query_transformation_node.md`](06_query_transformation_node.md). `academic_comparison` và `academic_calculation` giữ nguyên route riêng vì output shape khác hẳn (bảng so sánh song song / kết quả tính toán số học, không phải văn bản trích dẫn).

---

## 3. Input / Output Schema

### Input State
```json
{
  "intent_classification": {
    "primary_intent": "academic_calendar",
    "routing_mode": null,
    "confidence": 0.96
  }
}
```

### Chuyển Tiếp Graph (Graph Target)
```python
# Ví dụ với input trên (trước đây route sang CalendarLookupNode, nay hợp nhất về flow Advisory):
return QueryTransformationNode(mode="ADVISORY")
```

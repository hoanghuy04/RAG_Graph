# Chi Tiết System Prompt & Cơ Chế Hoạt Động Của PreRetrievalClassificationNode

Tài liệu này phân tích chi tiết **PreRetrievalClassificationNode** trong Graph Chatbot Trích xuất Tài liệu Học vụ KLTN, bao gồm cơ chế giải mã JWT Security Context, thuật toán phân loại Adaptive Routing, System Prompt của `adaptive_routing_agent` và kiến trúc mã nguồn.

---

## 1. Kiến Trúc Phân Quyền Bối Cảnh (Security Context Extraction)

Mỗi request từ phía Client gửi tới API Gateway bắt buộc phải chứa Header `Authorization: Bearer <JWT_TOKEN>`. Node thực hiện giải mã không đối xứng (RSA/HS256) để trích xuất `AcademicSecurityContext`:

```python
class AcademicSecurityContext(BaseModel):
    user_id: str
    role: Literal["SINH_VIEN", "GIANG_VIEN", "CAN_BO_PHONG_BAN", "ADMIN"]
    organization_scopes: List[str]  # e.g., ["KHOA_CNTT", "GLOBAL"]
    max_access_level: int           # e.g., 1 (Public), 2 (Internal Student), 3 (Faculty Staff), 5 (Board)
```

---

## 2. Toàn Bộ Nội Dung System Prompt (`adaptive_routing.yaml`)

```yaml
# CHAT_PROMPT_ADAPTIVE_ROUTING
description: "System prompt cho Adaptive Routing Agent trong KLTN Graph"
template: |
  ## Role
  Bạn là Adaptive Query Classifier trong hệ thống RAG trích xuất quy chế đào tạo đại học.

  ## Objective
  Phân tích câu hỏi của người dùng và phân loại cấu trúc luồng xử lý RAG thích hợp:
  - `SINGLE_INTENT`: Câu hỏi xoay quanh 1 đối tượng, 1 thủ tục hoặc 1 quy trình duy nhất.
  - `MULTI_INTENT`: Câu hỏi chứa nhiều vế, nhiều thực thể cần so sánh hoặc cần tra cứu từ nhiều phòng ban/khoa khác nhau.

  ## Decision Rules
  1. Trả `SINGLE_INTENT` khi user hỏi điều kiện của 1 học bổng, 1 mốc GPA, hoặc 1 quy trình rút môn.
  2. Trả `MULTI_INTENT` khi câu hỏi kết hợp từ 2 nội dung độc lập trở lên (VD: Vừa hỏi điều kiện song hành, vừa hỏi biểu phí 2 ngành).
  3. Trả về JSON đúng định dạng output schema.

  ## Output Contract
  {
    "intent_type": "SINGLE_INTENT" | "MULTI_INTENT",
    "confidence": float (0.0 đến 1.0)
  }
```

---

## 3. Quy Tắc Routing & Phán Quyết

| Loại Truy Vấn | Ví Dụ Minh Họa | Routing Action |
| :--- | :--- | :--- |
| **Single-Intent** | *"Cho em hỏi điều kiện xin giấy xác nhận sinh viên là gì?"* | Chuyển sang `QueryTransformationNode` với mode `HYDE` |
| **Single-Intent** | *"Thi lại học phần tối đa mấy lần?"* | Chuyển sang `QueryTransformationNode` với mode `HYDE` |
| **Multi-Intent** | *"Điều kiện học song hành ngành CNTT và Du lịch là gì, học phí chênh thế nào?"* | Chuyển sang `QueryTransformationNode` với mode `MULTI_QUERY_DECOMPOSITION` |
| **Multi-Intent** | *"Hồ sơ hoãn thi gồm những gì và nộp ở khoa hay phòng đào tạo?"* | Chuyển sang `QueryTransformationNode` với mode `MULTI_QUERY_DECOMPOSITION` |

---

## 4. Chuyển Trạng Thái Graph (Edge Code)

```python
async def run_pre_retrieval_node(state: AcademicChatState) -> BaseNode:
    # 1. Decode JWT Token
    state.academic_security_context = decode_jwt_token(state.jwt_token)
    
    # 2. Call Adaptive Routing Agent
    routing_res = await adaptive_routing_agent.run(state.user_query)
    state.routing_result = routing_res.data
    
    # 3. Branching Edge
    if state.routing_result.intent_type == "SINGLE_INTENT":
        return QueryTransformationNode(transformation_mode="HYDE")
    else:
        return QueryTransformationNode(transformation_mode="MULTI_QUERY_DECOMPOSITION")
```

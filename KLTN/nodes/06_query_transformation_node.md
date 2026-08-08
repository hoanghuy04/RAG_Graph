# 06. QueryTransformationNode (Biến Đổi Truy Vấn Song Song)

## 1. Vai Trò & Cơ Chế Hoạt Động
`QueryTransformationNode` chịu trách nhiệm giải quyết hiện tượng **Semantic Mismatch** (sự lệch pha ngữ nghĩa giữa câu hỏi dùng từ lóng, viết ngắn của sinh viên và ngôn ngữ hành chính, trang trọng trong văn bản quy chế).

Node này là một **LLM Agent Node** hoạt động với 4 chế độ biến đổi dựa trên tín hiệu `mode` hoặc `intent` được định tuyến từ bước trước:

---

## 2. 4 Chế Độ Biến Đổi (Modes)

### 1. HyDE (Hypothetical Document Embeddings) — Dành cho `SINGLE_INTENT`
- **Prompt sử dụng:** [agents/hyde_generator.yaml](file:///e:/Project/self/Flow/KLTN/prompt_template/agents/hyde_generator.yaml)
- **Cơ chế:** LLM tự sinh một câu trả lời giả định mang văn phong hành chính. Vector của câu trả lời giả này sẽ được dùng để search VectorDB.
- **Ví dụ:**
  - *Input:* "Mấy GPA thì bị rớt xuống cảnh báo học vụ vậy?"
  - *HyDE Doc:* "Theo Quy chế Đào tạo đại học chính quy, sinh viên bị cảnh báo học vụ nếu điểm trung bình tích lũy đạt dưới..."

### 2. Multi-Step Query Decomposition — Dành cho `MULTI_INTENT`
- **Prompt sử dụng:** [agents/multi_query_decomposer.yaml](file:///e:/Project/self/Flow/KLTN/prompt_template/agents/multi_query_decomposer.yaml)
- **Cơ chế:** LLM đóng vai trò Decomposer Agent bẻ gãy câu phức hợp thành 2 - 3 câu hỏi con độc lập, đầy đủ thực thể để chạy RAG song song (Fan-out).
- **Ví dụ:**
  - *Input:* "Điều kiện học song hành CNTT và Du lịch là gì, học phí chênh thế nào?"
  - *Decomposed:*
    1. "Quy định về điều kiện học vụ để đăng ký học song hành tại trường"
    2. "Định mức học phí ngành Công nghệ thông tin"
    3. "Định mức học phí ngành Quản trị Du lịch"

### 3. Procedure Transformation Mode
- **Cơ chế:** Sinh câu truy vấn tập trung vào quy trình thủ tục hành chính, cấu trúc dạng "Quy trình / Các bước để [hành động]".

### 4. Document Guidance Mode
- **Cơ chế:** Sinh câu truy vấn hướng đến việc xin cấp giấy tờ: "Điều kiện / Hướng dẫn xin cấp [tên giấy tờ]".

---

## 3. Input / Output Schema & State Update

### Input State
```json
{
  "mode": "MULTI_QUERY_DECOMPOSITION",
  "user_query": "Điều kiện học song hành CNTT và Du lịch là gì, học phí chênh thế nào?"
}
```

### Output State Update
Kết quả biến đổi được cập nhật vào mảng `transformed_queries` trong State:

```json
{
  "transformed_queries": [
    "Quy định về điều kiện học vụ để đăng ký học song hành hai chương trình đào tạo",
    "Định mức học phí ngành Công nghệ thông tin theo khóa học",
    "Định mức học phí ngành Quản trị Dịch vụ Du lịch và Lữ hành theo khóa học"
  ]
}
```

---

## 4. Graph Routing Logic
```python
# Chuyển tiếp sang tầng truy xuất tri thức
return RetrievalFilteringNode()
```

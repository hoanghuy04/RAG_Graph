# 07. AcademicComparisonNode (Phân Tích So Sánh Học Vụ)

## 1. Mạch Hoạt Động & Vai Trò Trong Graph
`AcademicComparisonNode` là một **LLM Agent Node** hoạt động ở **Tier 5 (Parallel RAG)**. Node này chịu trách nhiệm trích xuất các thực thể và tiêu chí so sánh từ câu hỏi của sinh viên, sau đó tự động sinh ra các sub-query song song để gửi tới tầng tìm kiếm.

Sự khác biệt với `multi_query_decomposer` là node này tập trung hoàn toàn vào việc phân tích đối sánh (Comparison) để chuẩn bị định dạng hiển thị dạng bảng (Comparison Table) ở tầng Generation.

---

## 2. Prompt Template Sử Dụng
- **File prompt:** [agents/academic_comparison.yaml](file:///e:/Project/self/Flow/KLTN/prompt_template/agents/academic_comparison.yaml)

---

## 3. Output Contract (Ràng buộc định dạng đầu ra)
LLM bắt buộc phải trả về JSON khớp với schema sau:

```json
{
  "entities": ["<thực thể 1>", "<thực thể 2>"],
  "comparison_criteria": ["<tiêu chí 1>", "<tiêu chí 2>"],
  "sub_queries": [
    "<query thực thể 1 × tiêu chí 1>",
    "<query thực thể 2 × tiêu chí 1>"
  ],
  "comparison_table_columns": ["<thực thể 1>", "<thực thể 2>"]
}
```

---

## 4. Input / Output Schema & State Update

### Input State
```json
{
  "user_query": "Ngành CNTT và Kế toán học phí mỗi tín chỉ chênh nhau bao nhiêu?"
}
```

### Output State Update
Node này ghi nhận kết quả trích xuất so sánh vào State:

```json
{
  "comparison_meta": {
    "entities": ["Công nghệ thông tin", "Kế toán"],
    "comparison_criteria": ["Định mức học phí theo tín chỉ"],
    "comparison_table_columns": ["Công nghệ thông tin", "Kế toán"]
  },
  "transformed_queries": [
    "Định mức học phí theo tín chỉ ngành Công nghệ thông tin năm học 2024-2025",
    "Định mức học phí theo tín chỉ ngành Kế toán năm học 2024-2025"
  ]
}
```

---

## 5. Graph Routing Logic
```python
# Đẩy các transformed_queries song song sang VectorDB
return RetrievalFilteringNode()
```

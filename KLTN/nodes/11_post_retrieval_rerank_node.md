# 11. PostRetrievalRerankNode (Tinh Lọc Bối Cảnh & Chấm Điểm Cross-Encoder)

## 1. Mạch Hoạt Động
`PostRetrievalRerankNode` nhận danh sách Top 15 - 20 chunks thô từ `RetrievalFilteringNode` và thực hiện tinh lọc bối cảnh nhằm:
1. Loại bỏ các chunk thông tin rác, nhiễu hoặc sai ngữ cảnh.
2. Giảm dung lượng token nạp vào LLM chính để tối ưu chi phí và tăng tốc sinh.
3. Đảm bảo những chunk thực sự có giá trị giải đáp trực tiếp cho câu hỏi gốc sẽ được đưa lên hàng đầu.

---

## 2. Các Bước Xử Lý Chi Tiết

### Bước 1: Cross-Encoder Re-ranking (Chấm điểm chéo)
Hệ thống nạp **Câu hỏi gốc của người dùng (`user_query`)** cùng danh sách 20 chunks thô vào mô hình Cross-Encoder mã nguồn mở (ví dụ: `bge-reranker-base` self-hosted).

Khác với Dense Vector (Bi-Encoder) tính điểm cosine độc lập, Cross-Encoder cho phép các từ trong câu hỏi và chunk tương tác trực tiếp qua Attention mechanism, chấm lại điểm số liên quan thực tế (`rerank_score` từ 0.0 đến 1.0).

### Bước 2: Score Thresholding (Lọc ngưỡng an toàn)
Hệ thống áp dụng ngưỡng an toàn cứng:
```python
score_threshold = 0.70
valid_chunks = [c for c in reranked_chunks if c.rerank_score >= score_threshold]
```
- Nếu `len(valid_chunks) == 0`: Đánh dấu trạng thái `has_valid_context = False` để điều hướng Graph sang `TicketFallbackNode`.
- Nếu `len(valid_chunks) > 0`: Tiếp tục nén và cắt lấy **Top 3 - 5 chunks hoàn hảo nhất**.

### Bước 3: Context Compression (Nén ngữ cảnh)
Tiến hành nén, loại bỏ các câu từ dư thừa rác trong chunk (khoảng trắng thừa, tiêu đề trang lặp lại), chỉ giữ lại các điều khoản và bảng biểu cốt lõi.

---

## 3. Output Schema & Conditional Routing

### Output State Update (Trường hợp thành công)
```json
{
  "has_valid_context": true,
  "filtered_context_chunks": [
    {
      "chunk_id": "CHK_QĐ2023_045",
      "doc_name": "Quy chế Đào tạo Đại học 2023",
      "section": "Điều 12",
      "content": "Sinh viên được đăng ký học chương trình thứ hai (học song hành) nếu tích lũy đủ 30 tín chỉ và GPA >= 2.50.",
      "rerank_score": 0.94
    }
  ]
}
```

### Conditional Routing Edge
```python
if state.has_valid_context:
    return GenerationSynthesisNode()
else:
    return TicketFallbackNode()
```

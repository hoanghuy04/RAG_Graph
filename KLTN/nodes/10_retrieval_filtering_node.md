# 10. RetrievalFilteringNode (Truy Xuất Tri Thức Song Song & Lọc Cứng Metadata)

## 1. Mạch Hoạt Động
`RetrievalFilteringNode` tiếp nhận danh sách các câu truy vấn đã biến đổi (`transformed_queries`) từ node trước và bối cảnh phân quyền `academic_security_context`. Đây là node thực thi truy xuất tri thức cốt lõi với cấu trúc 3 tầng bảo mật và hiệu năng cao:

```
transformed_queries + academic_security_context
        │
        ├──► [Tầng 1: Pre-Filtering] (Qdrant Payload Filter phân quyền cứng)
        │
        ├──► [Tầng 2: Parallel Hybrid Retrieval] (Dense Vector + Sparse BM25)
        │
        └──► [Tầng 3: Reciprocal Rank Fusion (RRF)] (Hợp nhất luồng kết quả -> Top 15-20)
```

---

## 2. Chi Tiết Các Tầng Thực Thi

### Tầng 1: Hard Pre-filtering (Lọc cứng theo phân quyền)
Trước khi tính toán tương đồng vector, hệ thống chuyển các thuộc tính bảo mật trong `academic_security_context` thành điều kiện lọc cứng (Payload Filter) trên Qdrant VectorDB:

```json
{
  "must": [
    {
      "key": "doc_package",
      "match": { "any": ["KHOA_CNTT", "GLOBAL"] }
    },
    {
      "key": "min_access_level",
      "range": { "lte": 2 }
    }
  ]
}
```
*Tác dụng:* Đảm bảo tuyệt đối không rò rỉ dữ liệu quy chế nội bộ giữa các Khoa hoặc các tài liệu có cấp độ bảo mật cao hơn quyền hạn của người dùng.

### Tầng 2: Parallel Hybrid Retrieval (Tìm kiếm kết hợp song song)
Hệ thống kích hoạt các luồng worker truy vấn song song qua `asyncio.gather` cho từng query con:
- **Dense Retrieval (Vector Search):** Sử dụng embedding model (ví dụ: `multilingual-e5-large`) để tìm kiếm tương đồng ngữ nghĩa.
- **Sparse Retrieval (BM25 Keyword Search):** So khớp từ khóa chính xác tuyệt đối (Mã môn học `INT1001`, tên văn bản `QĐ-45/2023`).

### Tầng 3: Reciprocal Rank Fusion (RRF Algorithm)
Hợp nhất tất cả kết quả từ các Sub-queries và 2 luồng Dense + Sparse bằng công thức RRF:
$$RRF\_Score(d) = \sum_{m \in M} \frac{1}{k + r_m(d)}$$
(với $k=60$). Thu về danh sách **Top 15 - 20 chunks thô sạch quyền truy cập**.

---

## 3. Input / Output Schema & State Update

### Input State
- `transformed_queries`: List các câu hỏi đã biến đổi (HyDE hoặc Multi-query).
- `academic_security_context`: Cấu hình phân quyền tài khoản (JWT decoded).

### Output State Update
Node cập nhật mảng `raw_retrieved_chunks` trong State:

```json
{
  "raw_retrieved_chunks": [
    {
      "chunk_id": "CHK_QĐ2023_045",
      "doc_name": "Quyết định 45/QĐ-ĐHKB về Học song hành",
      "section": "Điều 5, Khoản 2",
      "content": "Điều kiện đăng ký học song hành: Sinh viên đã hoàn thành năm thứ nhất, GPA tích lũy đạt từ 2.50 trở lên...",
      "access_level": 1,
      "rrf_score": 0.0325
    }
  ]
}
```

---

## 4. Graph Routing Logic
```python
return PostRetrievalRerankNode()
```

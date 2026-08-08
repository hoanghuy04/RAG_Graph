# Chi Tiết Cơ Chế Hoạt Động Của PostRetrievalRerankNode

Tài liệu này phân tích chi tiết **PostRetrievalRerankNode** - node tái sắp xếp điểm số bằng Cross-Encoder (`bge-reranker-base`), lọc ngưỡng điểm an toàn và nén bối cảnh.

---

## 1. Quy Trình Chấm Điểm Cross-Encoder & Score Thresholding

```python
SCORE_THRESHOLD = 0.70

async def rerank_and_filter_chunks(
    user_query: str, 
    raw_chunks: List[DocumentChunk]
) -> Tuple[bool, List[DocumentChunk]]:
    if not raw_chunks:
        return False, []
    
    # 1. Cross-Encoder rerank pairs: (user_query, chunk.content)
    pairs = [(user_query, chunk.content) for chunk in raw_chunks]
    scores = cross_encoder_model.predict(pairs)
    
    for chunk, score in zip(raw_chunks, scores):
        chunk.rerank_score = float(score)
        
    # Sort descending by score
    sorted_chunks = sorted(raw_chunks, key=lambda x: x.rerank_score, reverse=True)
    
    # 2. Score threshold filter
    valid_chunks = [c for c in sorted_chunks if c.rerank_score >= SCORE_THRESHOLD]
    
    if not valid_chunks:
        return False, []
        
    # 3. Take Top 3 - 5 clean chunks
    top_chunks = valid_chunks[:5]
    return True, top_chunks
```

---

## 2. Decision Logic & Edge Routing

```python
async def run_post_retrieval_rerank_node(state: AcademicChatState) -> BaseNode:
    has_valid, clean_chunks = await rerank_and_filter_chunks(
        state.user_query, 
        state.raw_retrieved_chunks
    )
    state.has_valid_context = has_valid
    state.filtered_context_chunks = clean_chunks
    
    if state.has_valid_context:
        return GenerationSynthesisNode()
    else:
        return TicketFallbackNode()
```

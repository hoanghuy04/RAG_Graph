# Chi Tiết Cơ Chế Hoạt Động Của RetrievalFilteringNode

Tài liệu này phân tích chi tiết **RetrievalFilteringNode** - node đảm nhận Hard Pre-filtering theo phân quyền, Parallel Hybrid Retrieval và Reciprocal Rank Fusion (RRF).

---

## 1. Thuật Toán Hard Pre-filtering (Lọc Cứng Phân Quyền)

Mọi truy vấn vector đều phải kèm theo Payload Filter được dựng tự động từ `AcademicSecurityContext`:

```python
def build_security_payload_filter(ctx: AcademicSecurityContext) -> QdrantFilter:
    return QdrantFilter(
        must=[
            FieldCondition(
                key="doc_package",
                match=MatchAny(any=ctx.organization_scopes)
            ),
            FieldCondition(
                key="min_access_level",
                range=Range(lte=ctx.max_access_level)
            )
        ]
    )
```

---

## 2. Thực Thi Retrieval Song Song & RRF Fusion

```python
async def execute_parallel_hybrid_retrieval(
    queries: List[str], 
    security_filter: QdrantFilter
) -> List[DocumentChunk]:
    tasks = []
    for q in queries:
        tasks.append(vector_db.dense_search(q, filter=security_filter))
        tasks.append(bm25_db.sparse_search(q, filter=security_filter))
    
    # Run all search queries in parallel fan-out
    results_list = await asyncio.gather(*tasks)
    
    # Apply Reciprocal Rank Fusion (RRF)
    merged_chunks = apply_rrf_fusion(results_list, top_k=20, k=60)
    return merged_chunks
```

---

## 3. Chuyển Tiếp Node

```python
async def run_retrieval_node(state: AcademicChatState) -> BaseNode:
    filter_cond = build_security_payload_filter(state.academic_security_context)
    state.raw_retrieved_chunks = await execute_parallel_hybrid_retrieval(
        state.transformed_queries, 
        filter_cond
    )
    return PostRetrievalRerankNode()
```

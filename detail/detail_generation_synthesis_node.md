# Chi Tiết System Prompt & Cơ Chế Hoạt Động Của GenerationSynthesisNode

Tài liệu này phân tích chi tiết **GenerationSynthesisNode** - node tổng hợp kết quả (Fan-in Aggregation) và sinh câu trả lời kèm trích dẫn văn bản minh bạch (`Citation Engine`).

---

## 1. Lắp Ghép Prompt Template Động (Dynamic Builder)

Sử dụng `PromptLoader` nạp từ `Flow/KLTN/prompt_template/`:

```python
def build_synthesis_prompt(state: AcademicChatState) -> str:
    # Format context chunks into XML string
    context_str = ""
    for idx, chunk in enumerate(state.filtered_context_chunks, 1):
        context_str += f"""
        <chunk id="[{idx}]">
          <doc_name>{chunk.doc_name}</doc_name>
          <version>{chunk.doc_version}</version>
          <section>{chunk.section}</section>
          <content>{chunk.content}</content>
        </chunk>
        """
    
    if state.routing_result.intent_type == "MULTI_INTENT":
        template_name = "chat_multi_intent_synthesis.yaml"
    else:
        template_name = "chat_single_intent.yaml"
        
    full_prompt = prompt_loader.render(
        template_name,
        user_id=state.academic_security_context.user_id,
        role=state.academic_security_context.role,
        organization_scopes=state.academic_security_context.organization_scopes,
        max_access_level=state.academic_security_context.max_access_level,
        context_chunks=context_str,
        user_query=state.user_query,
        sub_queries_list="\n".join(f"- {q}" for q in state.transformed_queries)
    )
    return full_prompt
```

---

## 2. Format Output & Citation Extraction Logic

LLM sẽ tự động chèn `[1]`, `[2]` theo từng khẳng định. Tầng response handler sẽ bóc tách danh mục `Citations` để render thẻ giao diện clickable đính kèm liên kết đến tệp PDF/Doc gốc của Trường.

```python
class FinalAcademicResponse(BaseModel):
    answer_markdown: str
    citations: List[CitationItem]
    
class CitationItem(BaseModel):
    citation_index: int
    doc_name: str
    doc_version: str
    section: str
    download_url: str
```

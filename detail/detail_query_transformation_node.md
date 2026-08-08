# Chi Tiết System Prompt & Cơ Chế Hoạt Động Của QueryTransformationNode

Tài liệu này phân tích chi tiết **QueryTransformationNode** - node xử lý biến đổi câu hỏi song song (Parallel Query Transformation) trong hệ thống KLTN Academic Chatbot.

---

## 1. Cơ Chế Hoạt Động & System Prompts

Node này vận hành hai Agent chuyên biệt tùy theo tín hiệu `transformation_mode`:

### Agent A: HyDE Generator Agent (`hyde_generator.yaml`)
```yaml
# CHAT_PROMPT_HYDE_GENERATOR
description: "System prompt cho HyDE Generator Agent"
template: |
  ## Role
  Bạn là chuyên gia soạn thảo văn bản hành chính học vụ đại học.

  ## Objective
  Viết một đoạn văn bản trả lời giả định (Hypothetical Document) mang văn phong quy chế chính thức cho câu hỏi của sinh viên.

  ## Input User Query
  {user_query}

  ## Output Style
  Đoạn văn khẳng định trang trọng, đồng nhất không gian ngữ nghĩa với văn bản quy chế thật.
```

### Agent B: Sub-Query Decomposer Agent (`multi_query_decomposer.yaml`)
```yaml
# CHAT_PROMPT_MULTI_QUERY_DECOMPOSER
description: "System prompt cho Multi-Step Query Decomposition Agent"
template: |
  ## Role
  Bạn là Sub-Query Decomposition Agent.

  ## Objective
  Phân rã câu hỏi phức hợp thành 2 - 3 câu hỏi con độc lập, chứa đầy đủ từ khóa ngữ cảnh.

  ## Output Schema
  {
    "sub_queries": ["sub_query_1", "sub_query_2", "sub_query_3"]
  }
```

---

## 2. Mã Nguồn Thực Thi Song Song & State Transition

```python
class QueryTransformationNode(BaseNode):
    transformation_mode: Literal["HYDE", "MULTI_QUERY_DECOMPOSITION"]
    
    async def run(self, state: AcademicChatState) -> BaseNode:
        if self.transformation_mode == "HYDE":
            hyde_res = await hyde_agent.run(state.user_query)
            state.transformed_queries = [hyde_res.data]
        else:
            decomp_res = await decomposer_agent.run(state.user_query)
            state.transformed_queries = decomp_res.data.sub_queries
            
        return RetrievalFilteringNode()
```

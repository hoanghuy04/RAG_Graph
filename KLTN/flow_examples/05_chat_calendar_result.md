# Luồng 6: `chat_calendar_result` — Tra Cứu Lịch Học Vụ (Không Dùng RAG)

## Query Ví Dụ
> **"Hạn đăng ký học phần học kỳ 2 năm học 2024-2025 là khi nào?"**

---

## Bối Cảnh Người Dùng
- MSSV: `SV20241122` | Role: `SINH_VIEN` | Đơn vị: `KHOA_CNTT` | Level: `2`

---

## Điểm Đặc Biệt của Luồng Này

Tương tự Calculation, luồng này **bỏ qua toàn bộ RAG pipeline**. `CalendarLookupNode` truy vấn trực tiếp **Calendar Database** (cơ sở dữ liệu lịch học vụ chính thức — không phải VectorDB), sau đó `GenerationSynthesisNode` dùng `chat_calendar_result.yaml` để LLM trình bày các mốc thời gian rõ ràng.

---

## Luồng Xử Lý Đầy Đủ (Node → Prompt → Input/Output)

---

### Node 01 — `GreetingDetectionNode` *(Deterministic)*
**Không dùng prompt.** Không phải chào → tiếp tục.

---

### Node 02 — `SecurityContextExtractionNode` *(JWT Decode)*
**Không dùng prompt.**

**State Output:**
```json
{
  "academic_security_context": {
    "user_id": "SV20241122",
    "role": "SINH_VIEN",
    "organization_scopes": ["KHOA_CNTT", "GLOBAL"],
    "max_access_level": 2
  }
}
```

---

### Node 03 — `MessageClassificationNode` *(LLM Agent)*
**Prompt template sử dụng:** `agents/message_classification.yaml`

**System Prompt (từ `message_classification.yaml`):**
```
## Intent Taxonomy
- academic_calendar: Hỏi mốc thời gian, ngày tháng, lịch học vụ.
  → "Khi nào đóng học phí học kỳ 2?", "Lịch thi cuối kỳ khi nào ra?"

## Output Contract
{ "primary_intent": "academic_calendar", "routing_mode": null }
# routing_mode = null vì không cần Fan-out RAG
```

**Input:**
```
"Hạn đăng ký học phần học kỳ 2 năm học 2024-2025 là khi nào?"
```

**LLM Output:**
```json
{
  "primary_intent": "academic_calendar",
  "secondary_intents": [],
  "confidence": 0.97,
  "routing_mode": null
}
```

**State Update:**
```json
{ "intent_classification": { "primary_intent": "academic_calendar", "routing_mode": null } }
```

→ Chuyển sang `IntentRoutingNode`

---

### Node 04 — `IntentRoutingNode` *(Deterministic)*
**Không dùng prompt.**

```python
# intent = "academic_calendar"
return CalendarLookupNode()
# KHÔNG gọi QueryTransformationNode hay RetrievalFilteringNode
```

---

### Node 09 — `CalendarLookupNode` *(Calendar DB + LLM, không dùng VectorDB)*
**Không có prompt agent riêng cho node này.** Node sử dụng logic tự động trích xuất tham số từ câu hỏi (NLP đơn giản hoặc rule-based), sau đó gọi Calendar DB API.

#### Bước 9a — Trích xuất tham số lịch
Node phân tích câu hỏi và xác định:
```
Câu hỏi: "Hạn đăng ký học phần học kỳ 2 năm học 2024-2025 là khi nào?"

Tham số trích xuất:
  - semester   : "HK2/2024-2025"
  - event_type : "dang_ky_hoc_phan"
```

#### Bước 9b — Calendar DB Query
**Calendar DB** được gọi (không phải VectorDB):
```
GET /calendar/events?semester=HK2/2024-2025&event_type=dang_ky_hoc_phan
```

**Calendar DB Response:**
```json
{
  "semester": "HK2/2024-2025",
  "events": [
    {
      "event_name": "Đăng ký học phần đợt 1",
      "start_date": "2025-01-06",
      "end_date": "2025-01-12",
      "note": "Ưu tiên sinh viên năm 4 và năm 3"
    },
    {
      "event_name": "Đăng ký học phần đợt 2 (bổ sung)",
      "start_date": "2025-01-15",
      "end_date": "2025-01-18",
      "note": "Mở cho tất cả sinh viên"
    },
    {
      "event_name": "Hủy học phần / học phần trả lại",
      "start_date": "2025-01-20",
      "end_date": "2025-01-24",
      "note": "Sau thời gian này cổng đóng hoàn toàn"
    }
  ]
}
```

#### Bước 9c — Fallback nếu Calendar DB không có dữ liệu
```python
if not calendar_data:
    # Chuyển sang QueryTransformationNode để thử tìm trong VectorDB
    return QueryTransformationNode(mode="SINGLE")
```

**State Update (trường hợp có dữ liệu):**
```json
{
  "calendar_data": {
    "semester": "HK2/2024-2025",
    "events": [ ...3 sự kiện ĐKHP... ]
  }
}
```

→ Chuyển thẳng sang `GenerationSynthesisNode` (BỎ QUA Node 10, 11)

---

### Node 12 — `GenerationSynthesisNode` *(Fan-in LLM Agent)*
**Prompt template sử dụng:** `main/chat_calendar_result.yaml` + 4 khối `common/`

**Bước 12.1 — Xác định template chính:**
Luồng đến từ `academic_calendar` + có `calendar_data` → `GenerationSynthesisNode` chọn `chat_calendar_result.yaml`:

```yaml
template: |
  {header}
  {academic_metadata}
  {security_access_control}
  {response_style}

  ## Chế Độ Hiển Thị Lịch Học Vụ
  Dưới đây là dữ liệu lịch học vụ chính thức từ hệ thống:
  <calendar_data>
    {calendar_data}
  </calendar_data>

  Hãy trình bày các mốc thời gian rõ ràng, dùng định dạng ngày chuẩn (DD/MM/YYYY),
  làm nổi bật các deadline quan trọng bằng in đậm,
  và nhắc nhở người dùng chú ý thời hạn.

  Câu hỏi gốc: {user_query}
```

> **Khác template RAG:** Không có `{academic_domain_rules}`, `{citation_rules}`, `{prepared_context}` vì dữ liệu đến từ Calendar DB — đã chính xác tuyệt đối, không cần trích dẫn văn bản quy chế.

**Bước 12.2 — Format các placeholder:**

| Placeholder | File nguồn | Giá trị điền |
|---|---|---|
| `{header}` | `common/header.yaml` | Giới thiệu AI Học vụ |
| `{academic_metadata}` | `common/academic_metadata.yaml` + State | SV20241122, SINH_VIEN, KHOA_CNTT, L2 |
| `{security_access_control}` | `common/security_access_control.yaml` | Phạm vi KHOA_CNTT + GLOBAL |
| `{response_style}` | `common/response_style.yaml` | In đậm deadline, ngày DD/MM/YYYY |
| `{calendar_data}` | **State.calendar_data** (Calendar DB) | JSON 3 sự kiện ĐKHP HK2/2024-2025 |
| `{user_query}` | Request gốc | "Hạn đăng ký học phần HK2/2024-2025?" |

**Bước 12.3 — System Prompt hoàn chỉnh:**
```
[HEADER + METADATA + SECURITY + STYLE]...

## Chế Độ Hiển Thị Lịch Học Vụ
Dữ liệu lịch học vụ chính thức từ hệ thống:
<calendar_data>
  {
    "semester": "HK2/2024-2025",
    "events": [
      { "Đăng ký đợt 1": 06/01 - 12/01/2025, ưu tiên năm 3-4 },
      { "Đăng ký đợt 2 (bổ sung)": 15/01 - 18/01/2025 },
      { "Hủy học phần": 20/01 - 24/01/2025 }
    ]
  }
</calendar_data>

Trình bày rõ ràng, in đậm deadline quan trọng, nhắc chú ý thời hạn.

Câu hỏi gốc: Hạn đăng ký học phần học kỳ 2 năm học 2024-2025 là khi nào?
```

**Bước 12.4 — LLM Output:**
> **Lịch Đăng ký Học phần HK2/2024-2025:**
>
> 📅 **Đợt 1** (ưu tiên năm 3, năm 4): **06/01/2025 → 12/01/2025**
> 📅 **Đợt 2 — Bổ sung** (mở cho tất cả): **15/01/2025 → 18/01/2025**
> 🔄 **Hủy học phần**: **20/01/2025 → 24/01/2025** *(cổng đóng sau ngày này)*
>
> ⚠️ *Sau ngày **24/01/2025** sẽ không thể thay đổi.*

---

## Bản Đồ Node → Prompt Template

```
Node 01: GreetingDetectionNode      → Không dùng prompt (Regex)
Node 02: SecurityContextExtraction  → Không dùng prompt (JWT Decode)
Node 03: MessageClassificationNode  → agents/message_classification.yaml
Node 04: IntentRoutingNode          → Không dùng prompt (Logic Router)
Node 09: CalendarLookupNode         → Không dùng prompt agent (Calendar DB API)
                                       [Fallback: agents/hyde_generator.yaml nếu không có data]
Node 12: GenerationSynthesisNode    → main/chat_calendar_result.yaml
                                       + 4 khối common/ (KHÔNG có prepared_context/citation)
```

## Sơ Đồ Luồng

```
"hạn ĐKHP HK2/2024-2025 là khi nào?"
        │
[Node01] → không chào
[Node02] JWT → SV20241122, level=2
[Node03] message_classification.yaml → academic_calendar, routing_mode=null
[Node04] Router → CalendarLookupNode  ← BỎ QUA QueryTransformation, VectorDB, Rerank
        │
[Node09] Calendar DB API → 3 mốc ĐKHP (06/01, 15/01, 20/01)
         └── Nếu không có data → fallback sang QueryTransformationNode(SINGLE)
        │
[Node12] chat_calendar_result.yaml + 4 common (không có RAG placeholders)
         System Prompt + calendar_data JSON → LLM chính
        │
✅ 3 mốc thời gian rõ ràng, in đậm deadline, nhắc hạn cuối
```

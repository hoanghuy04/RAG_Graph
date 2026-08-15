# Thiết Kế Cơ Chế Hỏi Lại Metadata Bắt Buộc (Missing Metadata Clarification)
# Tổng Hợp Field / Input / Output Toàn Graph + Prompt Hỏi Lại (tham khảo lisa-ai-agent)

---

## 1. Bối Cảnh & Vấn Đề

Graph KLTN hiện có **13 node** (xem [`nodes/kltn_graph_overview.md`](nodes/kltn_graph_overview.md)). Trong số các flow cần tham số đầu vào cụ thể để hoàn thành tác vụ, chỉ **`CalculationNode` (08)** có cơ chế phát hiện + hỏi lại tường minh khi thiếu tham số:

```
CalculationNode → missing_params → AskUserClarificationNode
                                  → pending_clarification (State, node 02 guard)
```

Cơ chế này được `SecurityContextExtractionNode` (02) ghi rõ là **chỉ dành riêng cho CalculationNode**:

> "Đây là điểm rẽ nhánh duy nhất của node này, và nó tồn tại để bảo vệ luồng hỏi-lại-tham số của `CalculationNode`." — [`nodes/02_security_context_extraction_node.md:62`](nodes/02_security_context_extraction_node.md#L62)

Các flow khác — **Advisory/Procedure/Document (06)**, **Comparison (07)**, **Calendar (09)** — cũng cần metadata bắt buộc để trả lời đúng, nhưng hiện **không có cơ chế hỏi lại**, chỉ âm thầm route thẳng vào RAG/DB rồi trông chờ ngưỡng rerank `≥ 0.70` (node 11) làm lưới an toàn cuối cùng.

Dự án `lisa-ai-agent` đã có sẵn cơ chế tương đương cho miền visa — `priority_missing_metadata_to_confirm` + `TASK_2` (quy tắc đặt câu hỏi) + `ASK_USER_CHOICE_GUIDE` (quy tắc sinh JSON option cho UI). Tài liệu này tổng hợp field/input/output của từng node KLTN và đề xuất áp dụng lại cùng cơ chế đó, tổng quát hoá thay vì bó cứng vào Calculation.

> **⚠️ Sửa lại tư duy ban đầu (2026-08)**: bản nháp đầu của tài liệu này liệt kê field bắt buộc của Advisory/Procedure/Document/Calendar thành 1 bảng cố định giống hệt cách lisa-ai-agent liệt kê `M0001`-`M0005` (registry tĩnh trong `extraction_policy.yaml`). Điều đó **sai** với KLTN: lisa-ai-agent chỉ có 1 miền cố định (visa) nên field cần hỏi biết trước được hết; KLTN là **RAG mở trên toàn bộ quy chế** — mỗi chủ đề văn bản khác nhau có thể phụ thuộc một thuộc tính sinh viên khác nhau (GDQP phụ thuộc hệ đào tạo, học bổng phụ thuộc khóa nhập học, học phí phụ thuộc ngành...), không thể liệt kê hết trước. Xem Mục 1.5 để hiểu đúng 2 loại field và vì sao chỉ 1 loại áp được danh sách tĩnh.

---

## 1.5. Hai Loại Field Thiếu: Structural (Type A, tĩnh) vs Content-Driven (Type B, động)

| | **Type A — Structural** | **Type B — Content-Driven** |
| :--- | :--- | :--- |
| Áp dụng cho | `CalculationNode` (08), `AcademicComparisonNode` (07) | `QueryTransformationNode` (06) → Advisory/Procedure/Document/Calendar |
| Field cần hỏi biết trước được không? | ✅ Có — cố định theo cấu trúc output của node (VD: Calculation luôn cần `calculation_type`; Comparison luôn cần ≥2 `entities`) | ❌ Không — phụ thuộc chủ đề quy chế mà câu hỏi chạm tới, chỉ lộ ra SAU khi đọc văn bản trả về |
| Thời điểm phát hiện | TRƯỚC khi truy vấn — ngay khi node parse xong `user_query` (chưa cần RAG) | SAU khi RAG trả `<academic_context>` — chỉ LLM tổng hợp câu trả lời (node 12) mới thấy được văn bản chia nhánh theo thuộc tính gì |
| Cơ chế | `missing_params` do chính node đó tính ra, giống hệt mô hình lisa (registry cố định, ví dụ đã có sẵn ở `calculation_extractor.yaml`) | Rule mới trong `task_1.yaml` ("PHÁT HIỆN NHÁNH PHỤ THUỘC THUỘC TÍNH SINH VIÊN") — LLM tại node 12 tự đặt tên field + tự trích option từ nhãn nhánh trong chính văn bản, KHÔNG tra registry nào |
| File prompt liên quan | `agents/calculation_extractor.yaml`, `agents/academic_comparison.yaml` (đã có output schema) | `common/task_1.yaml` (rule phát hiện) + `common/task_2.yaml` + `common/ask_user_form_guide.yaml` (file thật, đã tạo — xem Mục 6-7) |

Vì Type B không thể liệt kê trước, Mục 3 dưới đây **không phải một checklist bắt buộc kiểm tra hết** cho Advisory/Procedure/Document/Calendar — chỉ là các VÍ DỤ minh hoạ loại thuộc tính hay gặp, để chứng minh vấn đề tồn tại, không phải danh sách field cần code cứng vào node 06.

---

## 2. Hai Nguồn Metadata: Profile (JWT) vs Query

Khác với `lisa-ai-agent` (người dùng visa chưa có tài khoản → **mọi** metadata phải trích từ hội thoại), KLTN có sẵn `academic_security_context` từ JWT ngay ở node 02 — đây là nguồn **miễn phí, không cần hỏi lại**.

| Nguồn | Field | Lấy ở đâu | Dùng cho |
| :--- | :--- | :--- | :--- |
| **Profile (JWT, suy ra)** | `program` | **Suy ra từ scope cụ thể nhất trong `organization_scopes`** (VD: `KHOA_CNTT` → "Công nghệ thông tin"), không cần field `student_program` riêng | default `program` trong Calculation, default 1 vế so sánh trong Comparison |
| **Profile (JWT)** | `role`, `faculty_code` | `academic_security_context` | default `event_type`/hệ đào tạo trong Calendar |
| **Profile (JWT)** | `organization_scopes`, `max_access_level` | `academic_security_context` | Pre-filter Qdrant (node 10) — luôn bắt buộc, không thể thiếu vì lấy từ JWT |
| **Query (LLM extract)** | Tuỳ node — xem Mục 3 | Trích từ `user_query` qua LLM Agent | Khi field không nằm trong JWT, bắt buộc phải hỏi user nếu LLM không trích được |

> **Lưu ý**: `student_program` (khai báo trong `AcademicSecurityContext` ở [nodes/02](nodes/02_security_context_extraction_node.md#L37-L48)) là field **dư thừa** — `organization_scopes` đã chứa scope cụ thể nhất (VD: `["KHOA_CNTT", "GLOBAL"]` → phần tử không phải `GLOBAL`/cấp trường chính là đơn vị hẹp nhất của sinh viên). Chỉ cần một bảng tra `mã_khoa → tên_ngành` (VD: `KHOA_CNTT → "Công nghệ thông tin"`) là suy ra được `program`, không cần lưu/duy trì thêm field riêng dễ lệch dữ liệu với `organization_scopes` khi 2 nguồn không đồng bộ.

**Quy tắc thiết kế**: field nào có trong profile thì **tuyệt đối không đưa vào diện hỏi lại** — chỉ field không có sẵn trong JWT và LLM không trích được từ query mới được liệt vào `missing_params`.

---

## 3. Field Bắt Buộc — Type A (checklist thật) vs Type B (ví dụ minh hoạ, KHÔNG phải checklist)

| # | Node | Type | Field | Nguồn ưu tiên | Nếu thiếu, ảnh hưởng gì | Có cơ chế hỏi lại hiện tại? |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 08 | `CalculationNode` | A (tĩnh) | `calculation_type`, `courses[]`/`credits_registered`/`price_per_credit` tuỳ loại | Query | Không tính được số liệu | ✅ Có — `missing_params` → `AskUserClarificationNode` |
| 08 | `CalculationNode` | A (tĩnh) | `program` | Profile → fallback Query | Sai đơn giá tín chỉ nếu suy nhầm ngành | ⚠️ Có schema (`parameters.program`) nhưng chưa ghi rõ có default từ profile hay không |
| 07 | `AcademicComparisonNode` | A (tĩnh) | `entities[]` (bắt buộc ≥ 2) | Query, vế đầu có thể default = `program` suy ra từ `organization_scopes` | Chỉ so sánh được 1 vế → sub-query fan-out vô nghĩa | ❌ Không có — route thẳng `RetrievalFilteringNode()` dù `len(entities) < 2` |
| 07 | `AcademicComparisonNode` | A (tĩnh) | `comparison_criteria[]` | Query | Không rõ so sánh theo tiêu chí gì (học phí/tín chỉ/GPA) → sub-query mơ hồ | ❌ Không có |
| 06 | `QueryTransformationNode` (mọi mode — advisory/procedure/document/calendar) | **B (động, VÍ DỤ minh hoạ, không phải danh sách đầy đủ)** | VD: hệ đào tạo (GDQP), khóa nhập học (quy chế cũ/mới), ngành (học phí), loại giấy tờ cụ thể (document)... — chủ đề nào phát sinh thuộc tính nào chỉ biết được sau khi đọc `<academic_context>` | Văn bản trả về (chỉ lộ ra sau RAG) | Trả lời theo nhầm nhánh, hoặc liệt kê lan man tất cả nhánh | ✅ Có — rule "PHÁT HIỆN NHÁNH PHỤ THUỘC THUỘC TÍNH SINH VIÊN" trong `common/task_1.yaml` + `common/task_2.yaml` + `common/ask_user_form_guide.yaml` (file thật, xem Mục 6-7) |

> Field của node 02 (`user_id`, `role`, `organization_scopes`, `max_access_level`) không nằm trong bảng vì luôn có sẵn 100% từ JWT decode — không thuộc diện "có thể thiếu".
>
> **Type A** (08, 07) là checklist thật — nên code cứng kiểm tra đủ tham số trước khi chạy, đúng model của lisa-ai-agent. **Type B** (06) KHÔNG được code cứng danh sách field — cách đúng là để LLM tự phát hiện tại thời điểm tổng hợp câu trả lời, vì input là toàn bộ kho quy chế mở, không phải 1 miền cố định như visa.

---

## 4. Input / Output Đầy Đủ 13 Node (Tóm Tắt Tham Chiếu)

| # | Node | Input chính | Output chính | File chi tiết |
| :--- | :--- | :--- | :--- | :--- |
| 01 | `GreetingDetectionNode` | `user_query`, `history_len` | `EndNode(GREETING_TEMPLATE)` hoặc chuyển tiếp | [nodes/01](nodes/01_greeting_detection_node.md) |
| 02 | `SecurityContextExtractionNode` | `jwt_token` | `academic_security_context` | [nodes/02](nodes/02_security_context_extraction_node.md) |
| 03 | `MessageClassificationNode` | `user_query`, `conversation_history` | `intent_classification{primary_intent, secondary_intents, confidence, routing_mode}` | [nodes/03](nodes/03_message_classification_node.md) |
| 04 | `IntentRoutingNode` | `intent_classification` | Điều hướng node kế tiếp (không đổi State) | [nodes/04](nodes/04_intent_routing_node.md) |
| 05A | `DirectLLMNode` | `user_query`, `academic_security_context` | Câu trả lời trực tiếp (End) | [nodes/05a](nodes/05a_direct_llm_node.md) |
| 05B | `OffTopicRejectNode` | `user_query`, `intent_classification` | Template từ chối tĩnh (End) | [nodes/05b](nodes/05b_off_topic_reject_node.md) |
| 06 | `QueryTransformationNode` | `mode`, `user_query` | `transformed_queries[]` | [nodes/06](nodes/06_query_transformation_node.md) |
| 07 | `AcademicComparisonNode` | `user_query` | `comparison_meta{entities, comparison_criteria, comparison_table_columns}`, `transformed_queries[]` | [nodes/07](nodes/07_academic_comparison_node.md) |
| 08 | `CalculationNode` | `user_query`, `program` (suy ra từ `organization_scopes`) | `calculation_result{...}` hoặc `missing_params[]` | [nodes/08](nodes/08_calculation_node.md) |
| 06 (mode Calendar) | `QueryTransformationNode` | `user_query` | `transformed_queries[]` — giống hệt các mode khác, KHÔNG có `calendar_data` vì không có DB lịch riêng | [nodes/06](nodes/06_query_transformation_node.md) |
| 10 | `RetrievalFilteringNode` | `transformed_queries[]`, `academic_security_context` | `raw_retrieved_chunks[]` (Top 15-20) | [nodes/10](nodes/10_retrieval_filtering_node.md) |
| 11 | `PostRetrievalRerankNode` | `raw_retrieved_chunks[]`, `user_query` | `has_valid_context`, `filtered_context_chunks[]` (Top 3-5) | [nodes/11](nodes/11_post_retrieval_rerank_node.md) |
| 12 | `GenerationSynthesisNode` | `filtered_context_chunks`/`calculation_result`, `academic_security_context` | Câu trả lời cuối + Citation `[1][2]` (End) | [nodes/12](nodes/12_generation_synthesis_node.md) |
| 13 | `TicketFallbackNode` | `user_query`, `has_valid_context=false`, `academic_security_context` | `response_type: TICKET_FALLBACK` + `action_buttons[]` (End) | [nodes/13](nodes/13_ticket_fallback_node.md) |

---

## 5. Đề Xuất: Tổng Quát Hoá `pending_clarification`

Hiện `pending_clarification` (node 02) đã có cấu trúc generic (`origin_node`, `retry_count`), chỉ cần **bỏ ràng buộc "chỉ dành cho CalculationNode"** trong docs và cho phép mọi node ở Mục 3 ghi vào cùng State field này.

```python
class PendingClarification(BaseModel):
    origin_node: str                     # Điểm QUAY LẠI khi user trả lời — KHÔNG luôn trùng điểm PHÁT HIỆN thiếu field (xem lưu ý Type B bên dưới)
    pending_sub_query_id: str | None     # Luồng MULTI: sub-query nào còn treo; None nếu SINGLE
    missing_fields: list[str]            # 1 hoặc NHIỀU field hỏi cùng lúc — Type A: tên cố định; Type B: LLM tự đặt tại chỗ
    options: list[list[str] | None]      # SONG SONG với missing_fields; None = ô nhập tự do (điểm số, số tín chỉ)
    retry_count: int = 0

MAX_CLARIFICATION_RETRY = 2     # số lần hỏi lại tối đa cho CÙNG một field
```

**Đã bỏ trường `field_definition`.** Bản trước lưu nguyên văn câu hỏi LLM vừa viết vào trường mang tên "definition" — tên nói một đằng nội dung một nẻo. Sau khi có `confirmed_metadata`, nó cũng hết tác dụng: sinh viên trả lời rõ thì giá trị đã nằm trong `<student_declared_attributes>`; trả lời không rõ thì LLM đọc lại `<academic_context>` và tự diễn đạt câu hỏi mới, chính xác hơn câu cũ vì lượt này có thể đã retrieval ra văn bản khác. Định nghĩa field chỉ tồn tại ở đúng một nơi — nhãn nhánh trong văn bản quy chế — và không được nhân bản vào state. Vì vậy `task_2.yaml` cũng không còn placeholder `{missing_field_definition}`.

**KLTN không tạo file registry `missing_field_definitions.yaml`** như `lisa-ai-agent` có. Registry tĩnh chỉ hợp lệ với miền đóng 5 field (`M0001`–`M0005`); với RAG mở trên toàn bộ quy chế, nó vừa không liệt kê hết được (đúng sai lầm đã sửa ở Mục 1.5), vừa tạo nguồn sự thật thứ hai cạnh tranh với văn bản và lệch ngay khi Nhà trường ban hành quyết định mới. Định nghĩa của Type A đã nằm sẵn trong `calculation_extractor.yaml` / `academic_comparison.yaml`.

### `confirmed_metadata` — Thuộc tính đã chốt, tích luỹ qua hội thoại

```python
confirmed_metadata: dict[str, str] = {}      # {"training_type": "chinh_quy"}
```

Không có state này thì sinh viên chốt hệ đào tạo ở lượt 2, đến lượt 8 hỏi về học bổng (văn bản cũng chia nhánh theo hệ đào tạo) lại bị hỏi đúng câu cũ.

**Single-writer: Clarification Guard ở node 02.** Chỉ guard mới có đồng thời `pending.options` và câu trả lời của sinh viên. Node 12 phát hiện field thiếu và **hỏi**, nhưng không bao giờ tự ghi giá trị — câu trả lời đến ở lượt sau. Việc map câu trả lời → `option.id` là deterministic (khớp chính xác id khi bấm chip UI, khớp không dấu khi gõ tay), không cần thêm một lần gọi LLM.

**Ranh giới bảo mật**: `confirmed_metadata` là lời tự khai chưa xác minh. Nó chỉ dùng để **chọn nhánh trả lời**, tuyệt đối không được đưa vào Qdrant payload filter ở node 10 — nếu lẫn, sinh viên chỉ cần khai "em là học viên cao học" là đọc được tài liệu ngoài quyền. Trong prompt, hai nguồn được render thành hai thẻ XML tên khác hẳn nhau (`<academic_user_context>` vs `<student_declared_attributes>`) để LLM không nhầm.

### Vì sao `missing_fields` là mảng: ca khách vãng lai

`AcademicSecurityContext` cho phép `role = "KHACH"` (không có JWT) — cổng tra cứu mở cho thí sinh và phụ huynh. Khi đó `organization_scopes = ["GLOBAL"]`, `max_access_level = 1`, và **hồ sơ trống hoàn toàn**.

Hệ quả: một câu hỏi tưởng đơn giản như *"sinh viên năm nhất GPA 3.6 thì có được học bổng không?"* đụng ngay 4 thuộc tính chưa biết cùng lúc — hệ đào tạo, khóa nhập học, chương trình đào tạo, điểm rèn luyện. Không thuộc tính nào suy ra được từ các thuộc tính còn lại.

Hỏi lẻ từng cái qua 4 lượt thì phần lớn người dùng bỏ giữa chừng. Vì vậy `ask_user_form` (một field) đã được thay bằng **`ask_user_form`** (mảng `fields`), và `PendingClarification` mang hai mảng song song với bất biến `len(options) == len(missing_fields)`.

**Điền một phần vẫn được giữ**: `resolve_form()` trả về `(resolved, unresolved)`; phần đã điền vào thẳng `confirmed_metadata`, phần trống được `_keep_only()` thu lại thành form nhỏ hơn cho lượt sau. Vứt cả form vì thiếu một ô là cách nhanh nhất để mất người dùng.

**Chỉ gom field ĐỘC LẬP vào một form.** Field lồng nhau (tập `options` của B phụ thuộc giá trị của A, VD `loai_hinh_clc` chỉ tồn tại khi `he_dao_tao = chinh_quy_clc`) phải tách sang lượt sau — hỏi cùng lúc là bắt người dùng trả lời một câu có thể vô nghĩa với họ.

**Khách vãng lai không được nới quyền.** Lời khai của `KHACH` đi vào `confirmed_metadata` để chọn nhánh trả lời, không đi vào payload filter — khai "em là sinh viên K22 khoa CNTT" vẫn chỉ đọc được tài liệu `GLOBAL` cấp 1.

**Không lưu hàng đợi field chưa hỏi.** Type B phát hiện lại từ đầu mỗi lượt: sau khi sinh viên chốt field thứ nhất, node 06 sinh query mới → node 10 retrieval lại → `<academic_context>` là tập văn bản **khác**, và thuộc tính thứ hai định lưu sẵn có thể đã không còn liên quan. Một mảng `missing_fields[]` ở đây là cache của suy luận hết hiệu lực, không chỉ thừa mà sai. Vòng lặp tự sửa: hỏi 1 → chốt → retrieval mới → phát hiện lại nếu vẫn còn thiếu.

**Không có trần cho tổng số field được hỏi trong một phiên.** `MAX_CLARIFICATION_RETRY` chỉ chặn tình huống bệnh lý — hỏi mãi cùng một field vì câu trả lời không parse được. Còn việc hỏi một field **mới** là hành vi đúng: sau khi có CÁCH 1, đường hỏi lại chỉ kích hoạt khi không dựng được bảng và sinh viên không tự biết thuộc tính, nên mỗi câu hỏi còn lại đều là nhu cầu thông tin thật. Số lượng cũng đã tự bị chặn vì field chốt xong nằm lại trong `confirmed_metadata` và không bao giờ hỏi lần hai. Một ngân sách cứng chỉ khiến câu hỏi chính đáng thứ N bị trả lời hớ hênh vì hết quota. Thay vì chặn, hãy **đo** — field nào bị hỏi lặp bất thường là tín hiệu nên gắn metadata tốt hơn ở tầng index hoặc đưa thuộc tính đó vào JWT.

**Lưu ý quan trọng cho Type B — điểm phát hiện ≠ điểm quay lại**: field bị thiếu chỉ lộ ra tại node 12 (khi LLM đọc `<academic_context>` và áp rule "PHÁT HIỆN NHÁNH PHỤ THUỘC THUỘC TÍNH SINH VIÊN" trong `task_1.yaml`), nhưng `origin_node` để quay lại **phải là `QueryTransformationNode` (06)**, không phải node 12. Lý do: node 12 chỉ có `filtered_context_chunks` đã rerank cho câu hỏi CHƯA có thuộc tính bổ sung — nếu quay lại thẳng node 12 thì LLM vẫn chỉ thấy đúng context cũ, không tận dụng được câu trả lời mới của user để retrieval chính xác hơn. Quay lại node 06 để nó ghép thuộc tính user vừa cung cấp vào `user_query`/HyDE doc rồi chạy lại toàn bộ 06 → 10 → 11 → 12, cho kết quả đúng nhánh hơn. Type A (Calculation/Comparison) không có vấn đề này vì điểm phát hiện và điểm quay lại là cùng 1 node.

**`pending_clarification` được GHI vào state như thế nào (chiều ngược lại việc nạp vào prompt ở Mục 6-7)**: một bước deterministic (không dùng LLM) chạy ngay sau khi node 12 nhận response thô — trích khối ```json cuối cùng trong response, parse, và build lại `PendingClarification` object. Chi tiết đầy đủ + code minh hoạ ở [`nodes/12_generation_synthesis_node.md`](nodes/12_generation_synthesis_node.md#5-response-post-processing--thu-thập-pending_clarification-từ-output-llm) — bước này cũng là nơi quyết định `retry_count` giữ nguyên hay reset (giữ nguyên nếu field trùng lượt trước, reset về 0 nếu là field mới).

Routing guard ở node 02 (mục 5, [nodes/02_security_context_extraction_node.md](nodes/02_security_context_extraction_node.md#5-clarification-guard-nối-lại-ngữ-cảnh-đa-lượt)) bỏ ràng buộc hard-code `"CalculationNode"`, nhận mọi `origin_node`. **`retry_count` chỉ tăng khi câu trả lời KHÔNG khớp option nào** — nếu khớp, giá trị được ghi vào `confirmed_metadata` và `pending_clarification` xoá về `None`.

**Nguyên tắc bổ sung khi tổng quát hoá**: trước khi ghi field vào `missing_params`/`pending_clarification`, mỗi node **bắt buộc** kiểm tra `academic_security_context` trước — field nào default được từ profile thì gán ngầm, không hỏi (xem Mục 2).

---

## 6-7. Prompt Hỏi Lại — Đã Tạo Thành File Thật (không còn là bản nháp inline)

> **Cập nhật (2026-08)**: 2 mục này ban đầu chỉ là YAML nháp viết tay mô phỏng lisa-ai-agent với field cố định (`event_type` của Calendar...) — đã **lỗi thời** sau khi sửa lại theo mô hình Type A/Type B ở Mục 1.5. 3 file thật đã được tạo/sửa trực tiếp trong `prompt_template/`, không lặp lại nội dung ở đây nữa:

| File | Vai trò | Điểm khác so với bản nháp/lisa gốc |
| :--- | :--- | :--- |
| [`prompt_template/common/task_1.yaml`](prompt_template/common/task_1.yaml) | Thêm rule **"PHÁT HIỆN NHÁNH PHỤ THUỘC THUỘC TÍNH SINH VIÊN"** — đây là bước Type B mới, lisa-ai-agent không có tương đương vì miền visa không cần "tự phát hiện" field, field luôn có sẵn trong registry | Chỉ KLTN mới cần, vì RAG mở trên nhiều chủ đề quy chế khác nhau |
| [`prompt_template/common/task_2.yaml`](prompt_template/common/task_2.yaml) | Đặt đúng 1 câu hỏi cuối cùng — nhận field từ 1 trong 2 nguồn: `<missing_metadata_to_confirm>` (Type A, node trước truyền vào) HOẶC tự phát hiện ở NHIỆM VỤ 1 (Type B) | Thêm hẳn 1 đoạn phân biệt 2 nguồn field ngay đầu file — bản gốc lisa chỉ có 1 nguồn (registry) |
| [`prompt_template/common/ask_user_form_guide.yaml`](prompt_template/common/ask_user_form_guide.yaml) | Quy tắc sinh JSON `ask_user_form` | Với Type B, `options.id` phải copy nguyên văn nhãn nhánh **trong chính văn bản vừa đọc**, không phải từ registry — có ví dụ minh hoạ riêng cho case này |

Cả 2 file `task_2` và `ask_user_form_guide` đã được wire vào `{task_2}` trong [`main/chat_academic_advisory.yaml`](prompt_template/main/chat_academic_advisory.yaml) và [`main/chat_multi_intent_synthesis.yaml`](prompt_template/main/chat_multi_intent_synthesis.yaml) — 2 template RAG duy nhất có khả năng gặp Type B (Calculation/Comparison dùng `AskUserClarificationNode` riêng cho Type A, không qua 2 file này).

---

## 8. Ví Dụ Luồng Có Câu Hỏi — Advisory Flow Thiếu `khóa/hệ đào tạo`

**Query**: *"Sinh viên năm cuối có được miễn học phần Giáo dục Quốc phòng không?"*
**Profile**: `role: SINH_VIEN`, `organization_scopes: ["KHOA_CNTT", "GLOBAL"]` → suy ra `program = "Công nghệ thông tin"` qua bảng tra `mã_khoa → tên_ngành`, JWT **không có** field hệ đào tạo (chính quy/liên thông/VLVH) — quy chế miễn giảm khác nhau giữa các hệ.

```
Node 01 GreetingDetectionNode      → không phải chào → tiếp tục
Node 02 SecurityContextExtraction  → academic_security_context{role: SINH_VIEN, organization_scopes: ["KHOA_CNTT", "GLOBAL"], ...}
                                      → program = "Công nghệ thông tin" (suy ra từ organization_scopes)
                                      (không có pending_clarification) → MessageClassificationNode
Node 03 MessageClassificationNode  → { primary_intent: "academic_advisory", routing_mode: "SINGLE", confidence: 0.95 }
Node 04 IntentRoutingNode          → QueryTransformationNode(mode="HYDE")
```

**Node 06 — `QueryTransformationNode` (mode HyDE)**: KHÔNG có gì để kiểm tra trước — sinh HyDE doc bình thường từ câu hỏi gốc, vì tại thời điểm này chưa biết GDQP miễn giảm sẽ phân nhánh theo thuộc tính gì (đây là Type B, không có checklist tĩnh — xem Mục 1.5).

```
Node 06 QueryTransformationNode → transformed_queries = ["Theo Quy chế Giáo dục Quốc phòng...
                                    sinh viên được xét miễn học phần nếu..."]  (HyDE doc, chưa biết hệ đào tạo)
Node 10 RetrievalFilteringNode  → raw_retrieved_chunks[] (đã lọc theo organization_scopes)
Node 11 PostRetrievalRerankNode → rerank_score ≥ 0.70 → has_valid_context = true
                                    filtered_context_chunks = [
                                      "Điều 8: Sinh viên hệ CHÍNH QUY được miễn học phần GDQP nếu đã
                                       hoàn thành chương trình GDQP tại bậc THPT có xác nhận... [1]
                                       Sinh viên hệ LIÊN THÔNG và VLVH không thuộc diện áp dụng
                                       điều khoản miễn giảm này, phải học đầy đủ học phần. [1]"
                                    ]
```

**Node 12 — `GenerationSynthesisNode` — ĐIỂM PHÁT HIỆN THIẾU METADATA (Type B)**

LLM áp rule "PHÁT HIỆN NHÁNH PHỤ THUỘC THUỘC TÍNH SINH VIÊN" trong `task_1.yaml`: thấy văn bản chia 2 nhánh theo **hệ đào tạo**, thuộc tính này không có trong `academic_security_context` lẫn `user_query`. Chuyển sang `task_2.yaml`, tự đặt field `training_type`, tự lấy option từ đúng 2 nhãn xuất hiện trong chunk (`CHÍNH QUY` / `LIÊN THÔNG và VLVH`, tách thành 3 option cho rõ), sinh phản hồi:

```markdown
Về việc miễn học phần Giáo dục Quốc phòng, quy định miễn giảm hiện khác nhau
tuỳ theo hệ đào tạo sinh viên đang theo học [1].

Bạn đang học theo hệ đào tạo nào ạ?

```json
{"type": "ask_user_form", "fields": [
  {"field": "training_type", "label": "Hệ đào tạo", "options": [
    {"id": "chinh_quy", "label": "Chính quy"},
    {"id": "lien_thong", "label": "Liên thông"},
    {"id": "vlvh", "label": "Vừa làm vừa học"}
  ]}
]}
```
```

State cập nhật `pending_clarification` với `origin_node = "QueryTransformationNode"` (06) — **không phải node 12**, dù chính node 12 là nơi phát hiện (xem lý do "điểm phát hiện ≠ điểm quay lại" ở Mục 5):

```json
{
  "pending_clarification": {
    "origin_node": "QueryTransformationNode",
    "pending_sub_query_id": null,
    "missing_fields": ["training_type"],
    "options": [["chinh_quy", "lien_thong", "vlvh"]],
    "retry_count": 0
  }
}
```

**Lượt tiếp theo** — Sinh viên chọn `"chinh_quy"` (hoặc gõ tay "Chính quy"):

```
Node 02 SecurityContextExtraction (Clarification Guard)
  → resolve_answer("Chính quy ạ", ["chinh_quy","lien_thong","vlvh"]) = "chinh_quy"
  → confirmed_metadata["training_type"] = "chinh_quy"
  → pending_clarification = None
  → route thẳng về pending.origin_node = "QueryTransformationNode"
  → BỎ QUA MessageClassificationNode (tránh bị phân loại nhầm "social_chat")
```

Từ lượt này trở đi, `<student_declared_attributes>` trong prompt luôn mang `training_type: chinh_quy` — mọi câu hỏi sau chạm văn bản chia nhánh theo hệ đào tạo đều trả lời thẳng, không hỏi lại.

**Node 06 (lượt 2)** — ghép `training_type = "chinh_quy"` vào câu hỏi, sinh lại HyDE doc chính xác hơn:

```
"Theo Quy chế Giáo dục Quốc phòng và An ninh áp dụng cho sinh viên hệ chính quy,
sinh viên năm cuối được xét miễn học phần nếu đã hoàn thành..."
```

```
Node 10 RetrievalFilteringNode  → raw_retrieved_chunks[] (retrieval mới, đã có hệ đào tạo trong query)
Node 11 PostRetrievalRerankNode → rerank_score ≥ 0.70 → has_valid_context = true
Node 12 GenerationSynthesisNode → không còn nhánh nào thiếu thuộc tính (đã biết hệ chính quy)
                                    → câu trả lời cuối kèm citation [1], không hỏi lại nữa
```

**Trường hợp sinh viên trả lời không rõ 2 lượt liên tiếp** (VD: gõ lại câu hỏi gốc thay vì chọn hệ đào tạo):

```python
retry_count = 2  # đã chạm MAX_CLARIFICATION_RETRY
pending_clarification = None  # node 02 dọn state, tránh kẹt vòng lặp
→ route về MessageClassificationNode như bình thường
→ QueryTransformationNode sinh HyDE KHÔNG có training_type
   (Node 12 buộc phải chọn 1 cách xử lý an toàn: nêu đủ cả 2 nhánh kèm điều kiện,
   theo đúng rule "SO SÁNH PHƯƠNG ÁN"/nhánh trong task_1.yaml, thay vì đoán 1 nhánh)
```

---

## 9. Việc Còn Lại Nếu Triển Khai

- [x] Tạo 2 file prompt thật `prompt_template/common/task_2.yaml` + `ask_user_form_guide.yaml`, wire `{task_2}` vào `main/chat_academic_advisory.yaml` + `main/chat_multi_intent_synthesis.yaml`.
- [x] Thêm rule "PHÁT HIỆN NHÁNH PHỤ THUỘC THUỘC TÍNH SINH VIÊN" (Type B) vào `prompt_template/common/task_1.yaml`.
- [ ] Thêm bảng field bắt buộc Type A (Mục 3) vào `nodes/07`, `nodes/08` như một mục `## Missing Metadata Handling` riêng (Type B của node 06 KHÔNG cần bảng field — bản chất là "không có bảng", đã giải thích ở Mục 1.5).
- [x] Bỏ câu "chỉ dành cho CalculationNode" ở [`nodes/02_security_context_extraction_node.md`](nodes/02_security_context_extraction_node.md), đổi thành mô tả generic theo Mục 5 (bao gồm cả lưu ý "điểm phát hiện ≠ điểm quay lại" cho Type B).
- [x] Thêm `confirmed_metadata` vào State; guard node 02 làm single-writer; render thành `<student_declared_attributes>` trong [`common/academic_metadata.yaml`](prompt_template/common/academic_metadata.yaml).
- [x] Bỏ `field_definition` khỏi `PendingClarification` và placeholder `{missing_field_definition}` khỏi [`common/task_2.yaml`](prompt_template/common/task_2.yaml).
- [x] Cấm dùng thuộc tính tự khai cho phân quyền — [`common/security_access_control.yaml`](prompt_template/common/security_access_control.yaml) mục 2.
- [x] Thêm `pending_sub_query_id` để luồng MULTI resume đúng sub-query còn treo, không chạy lại các ý đã trả lời xong (LLM phát ra qua khoá `sub_query_id` trong `ask_user_form`).
- [x] Bổ sung tiêu chí phân định **CÁCH 1 (trả lời phân nhánh bằng bảng)** vs **CÁCH 2 (hỏi lại)** vào `task_1.yaml`, kèm bố cục bảng ở `response_style.yaml` mục 4. CÁCH 1 là mặc định khi 2-4 nhánh cùng bộ tiêu chí và thuộc tính là thứ sinh viên tự biết — hỏi lại lúc đó chỉ tốn một lượt mà không thu được thông tin gì mới.
- [ ] Kiểm tra node 10 (`RetrievalFilteringNode`) **không** đọc `confirmed_metadata` khi build Qdrant payload filter — hiện chưa có ràng buộc nào chặn ở tầng code.
- [ ] Cập nhật ví dụ GDQP ở Mục 8: 3 nhánh cùng tiêu chí "được miễn hay không" nên theo rule mới nó thuộc **CÁCH 1** (bảng), không còn minh hoạ đúng đường hỏi lại — nên đổi sang một case không dựng được bảng, như ví dụ miễn giảm học phí trong [`design/overview_flow.html`](design/overview_flow.html).
- [ ] Cho `AcademicComparisonNode` (07) kiểm tra `len(entities) < 2` trước khi route sang `RetrievalFilteringNode()`, ghi `missing_params` giống mô hình `CalculationNode` (Type A) — hiện chưa có, đã nêu ở Mục 3.
- [ ] Bổ sung node mới `AskUserClarificationNode` vào sơ đồ Mermaid tổng ([`nodes/kltn_graph_overview.md`](nodes/kltn_graph_overview.md)) cho nhánh Type A (Calculation/Comparison) — Type B (Advisory) không cần node riêng, xử lý ngay trong `GenerationSynthesisNode`/`task_2.yaml`.

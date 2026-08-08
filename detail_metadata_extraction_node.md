# Chi Tiết Luồng MetadataExtractionNode Khi Không Có Tool100 (LISA AI Agent)

Tài liệu này phân tích chi tiết 6 bước xử lý của **`MetadataExtractionNode`** khi hệ thống hoạt động ở chế độ **không có bộ trích xuất quy tắc `tool100`** (hoặc `tool100` không bóc tách được trường nào). Khi đó, hệ thống chuyển hoàn toàn sang cơ chế **Full LLM Extraction Pass** kết hợp với kiến trúc chạy song song PAI Agents và bảo toàn dữ liệu bằng Accumulator Overlay Pattern.

---

## 1. Sơ Đồ Luồng 6 Bước (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    actor Client as Khách hàng / Frontend
    participant Node as MetadataExtractionNode
    participant State as ChatState Accumulator
    participant PAI as PAI Agents (LLM Units)
    participant Registry as MetadataRegistry (Validation)

    Note over Node: Bắt đầu MetadataExtractionNode.run()

    rect rgb(240, 248, 255)
    Note over Node, State: BƯỚC 1: Seed Metadata & User Choice Bypass
    Node->>State: Seed request_metadata & full_metadata ban đầu
    alt Có tương tác Widget (ask_user_choice)
        Node->>Node: Match câu trả lời với option label & validate registry
        Node->>State: Ghi đè thẳng user_choice_delta vào full_metadata
        Node-->>Client: Chuyển tiếp ngay -> IntentDetectionNode() (Latency < 5ms)
    end
    end

    rect rgb(255, 250, 240)
    Note over Node: BƯỚC 2: Phân Nhóm 100% Các Trường (Full LLM Partitioning)
    Node->>Node: exact_field_ids = [] (vì không có tool100)
    Node->>Node: Gom 100% field sống trong policy thành các _ExtractionUnitSpec
    Node->>Node: Phân định Prompt (per_field vs batched) & nén history
    end

    rect rgb(240, 255, 240)
    Note over Node, PAI: BƯỚC 3: Kích Hoạt Đội Ngũ LLM Chạy Song Song (asyncio.gather)
    par Task 1: Batch N Unit Extractions
        Node->>PAI: run_extraction_unit() per group spec
        PAI-->>Node: Trả về ExtractionUnitOutcome (per-unit error isolation)
    and Task 2: Intent Detection
        Node->>PAI: run_intent_detection()
        PAI-->>Node: Trả về IntentDetectionOutcome
    and Task 3: Information Needs
        Node->>PAI: extract_information_needs()
        PAI-->>Node: Trả về InformationNeedOutcome
    end
    end

    rect rgb(255, 240, 245)
    Note over Node, Registry: BƯỚC 4: Kiểm Duyệt & Lọc Chất Lượng Dữ Liệu
    Node->>Registry: Đối chiếu value với allowed_values & FREE_METADATA
    Registry-->>Node: Loại bỏ value rác/ngoài danh mục (metadata_value_off_allowed)
    Node->>Node: Lọc llm_metadata chỉ giữ status HIGH / EXACTLY
    end

    rect rgb(245, 245, 255)
    Node over Node, State: BƯỚC 5: Post-Processing C-Fail & Accumulate State
    Node->>Node: Scan negative regex -> Gán sentinel "Không có" cho c-fail metadata
    Node->>Node: Scan retract regex -> Xóa đè blacklist cũ bằng "Không có"
    Node->>State: accumulate_metadata(merged_llm, merge=overlay)
    Node->>State: Cập nhật mentioned_fields & information_needs
    end

    rect rgb(250, 250, 250)
    Note over Node: BƯỚC 6: Telemetry Stats & Transition
    Node->>Node: Log extraction_stats (tool_count=0, llm_count=N)
    Node-->>Client: Chuyển giao điều khiển -> IntentDetectionNode()
    end
```

---

## 2. Diễn Giải Chi Tiết 6 Bước Xử Lý (Ngôn Ngữ Tự Nhiên & Code Engineering)

### 📍 BƯỚC 1: Khởi Tạo Accumulator & Kiểm Tra Bấm Nút Nhanh (User Choice Bypass)

* **Khởi tạo An Toàn (`_seed_request_metadata`)**:
  * Ngay khi bước vào node, hệ thống lập tức lấy dữ liệu metadata do client truyền lên (`ctx.deps.request.metadata.visa_applicant`) để nạp vào `request_metadata` và `full_metadata`.
  * **Mục đích**: Bảo đảm nguyên tắc bất biến Accumulator. Dù phía sau có xảy ra sự cố hay ngoại lệ gì, thông tin hồ sơ cũ của khách hàng vẫn được bảo toàn trọn vẹn.

* **Kiểm Tra User Choice Bypass (`_handle_user_choice_bypass`)**:
  * Nếu câu nói trước đó của Lisa có gửi kèm widget lựa chọn (`ask_user_choice` JSON block) và câu trả lời của khách hàng khớp với một trong các lựa chọn đó (ví dụ khách bấm nút hoặc gõ *"Theo tour"*):
    1. Hệ thống đọc trực tiếp `field_id` (ví dụ `M0004`) và `value_to_apply` từ widget.
    2. Kiểm tra `value` hợp lệ trong `MetadataRegistry` (qua `_resolve_allowed_values`).
    3. Ghi đè trực tiếp kết quả vào `full_metadata` via `ctx.state.accumulate_metadata`.
    4. Gán ý định `user_intent = Intent.UNKNOWN` (với độ tin cậy `1.0`).
    5. **Kết thúc node ngay lập tức** và chuyển tiếp sang `IntentDetectionNode()`. Latency của lượt này đạt cực thấp (**< 5ms**), hoàn toàn không tốn token hay thời gian chờ LLM.

---

### 📍 BƯỚC 2: Phân Nhóm 100% Các Trường Cho Đội Ngũ LLM (Full LLM Partitioning)

* **Tập `exact_field_ids` Rỗng**:
  * Bình thường, nếu có `tool100`, những trường đã được Tool chốt chính xác (`EXACTLY`) sẽ được đưa vào `exact_field_ids` để skip không gửi cho LLM.
  * Vì không có `tool100`, hàm `_collect_exact_field_ids` trả về một tập rỗng (`frozenset()`).

* **Phân Nhóm Toàn Bộ Trường (`_partition_extraction_units`)**:
  * Hệ thống duyệt qua tất cả các nhóm khai báo trong file cấu hình policy (`metadata_extract_config.groups`).
  * Do không có trường nào bị skip, **100% các nhóm trường** đều được đóng gói thành các đối tượng spec `_ExtractionUnitSpec` để chuẩn bị cho LLM bóc tách.

* **Chuẩn Bị Prompt & Nén Lịch Sử Hội Thoại**:
  * Với các unit có định dạng `per_field`, câu thoại được gán prefix `Khách` / `Lisa`.
  * Với các unit định dạng `batched`, câu thoại được gán prefix `Khách hàng` / `Tư vấn viên`. Đồng thời, các câu trả lời dài hoặc chứa khối JSON của Lisa sẽ được nén qua hàm `_summarize_assistant_turn` để tiết kiệm dung lượng prompt gửi cho LLM.

---

### 📍 BƯỚC 3: Kích Hoạt Đội Ngũ LLM Chạy Song Song (`asyncio.gather`)

Hệ thống sử dụng cơ chế bất đồng bộ `asyncio.gather` để đồng thời kích hoạt 3 tác vụ chính trong một nhịp xử lý duy nhất:

1. **Trích Xuất Metadata Theo Nhóm (`run_extraction_unit`)**:
   * Khởi tạo các mô hình PAI Agents tương ứng với từng nhóm (`create_pai_model(agent_name)`).
   * Gửi prompt đã chuẩn bị đến từng Agent để thực hiện trích xuất dữ liệu song song.
   * **Cơ chế cô lập lỗi (Error Isolation)**: Mỗi unit được bọc trong `_timed_extraction`. Nếu 1 nhóm LLM gặp sự cố (timeout, lỗi JSON parse), chỉ nhóm đó bị đánh dấu lỗi (`ExtractionUnitOutcome`), kết quả từ các nhóm LLM khác vẫn được giữ nguyên và sử dụng bình thường.

2. **Phát Hiện Ý Định (`run_intent_detection`)**:
   * Song song với việc bóc tách metadata, một agent chuyên trách sẽ phân tích câu thoại để xác định ý định chính của người dùng (`CHECK_ELIGIBILITY`, `CONSULT`...).

3. **Phân Tích Nhu Cầu Thông Tin (`extract_information_needs`)**:
   * Phân tích xem người dùng đang chủ động hỏi về trường thông tin nào, hoặc có nhu cầu so sánh điều kiện giữa các quốc gia hay không.

---

### 📍 BƯỚC 4: Kiểm Duyệt & Lọc Chất Lượng Dữ Liệu (Registry Validation)

Sau khi đội ngũ LLM hoàn tất trích xuất, kết quả sẽ được đưa qua quy trình kiểm duyệt nghiêm ngặt (`_convert_unit_results`):

* **Lọc Placeholder**: Loại bỏ hoàn toàn các giá trị rác dạng placeholder (như `"chưa rõ"`, `"n/a"`, `"khách không nói"`...).
* **Validate Danh Mục Chuẩn (`MetadataRegistry`)**:
  * So sánh giá trị LLM trích xuất được với danh sách `allowed_values` được định nghĩa trong Registry của hệ thống.
  * Nếu trường đó không thuộc danh mục tự do (`FREE_METADATA`) và giá trị trích xuất không hợp lệ (ví dụ: LLM tự bịa ra một loại visa không tồn tại), hệ thống sẽ đổ warning log `metadata_value_off_allowed` và **loại bỏ ngay lập tức**.
* **Lọc Độ Tin Cậy (Confidence Threshold)**:
  * Kết quả từ LLM được lọc qua `metadata_service.filter_include_statuses`. Chỉ giữ lại những dữ liệu mà LLM trích xuất đạt độ tin cậy `HIGH` hoặc `EXACTLY`.

---

### 📍 BƯỚC 5: Post-Processing C-Fail / Blacklist & Tích Lũy Metadata State

Đối với các thông tin nhạy cảm có nguy cơ dẫn đến từ chối hồ sơ (nợ xấu, tiền án tiền sự, bị hoãn xuất cảnh... gọi chung là c-fail metadata), hệ thống áp dụng 2 quy tắc kiểm tra Regex sau:

1. **Phủ Định Phản Hồi (`_apply_c_fail_negative_regex`)**:
   * Khi Assistant đang hỏi về một thông tin c-fail và khách hàng trả lời phủ định ("không", "chưa từng", "không bị"...), hệ thống tự động thiết lập giá trị sentinel `C_FAIL_NONE_VALUE` (`"Không có"`) với độ tin cậy `HIGH`.
2. **Đính Chính Hồ Sơ (`_apply_c_fail_retract`)**:
   * Nếu trong `full_metadata` trước đó lỡ mang giá trị c-fail rủi ro (do nhập nhầm/seed nhầm) và khách hàng đưa ra câu đính chính ("tôi không bị nợ xấu", "ghi nhầm đấy"...), hệ thống chủ động ghi đè giá trị `"Không có"` vào kết quả `merged`.

* **Tích Lũy Về Accumulator (`ctx.state.accumulate_metadata`)**:
  * Do không có `tool_metadata`, dữ liệu `merged` chính là toàn bộ kết quả LLM đã qua kiểm duyệt và post-processing.
  * Áp dụng cơ chế **Overlay Pattern** (`metadata_service.overlay`) để đè `merged` lên `full_metadata`, cập nhật thông tin hồ sơ khách hàng trên `ChatState`.
  * Tinh chỉnh lại danh sách `mentioned_fields` (bỏ đi các trường đã có giá trị chính thức trong `full_metadata`).

---

### 📍 BƯỚC 6: Telemetry Stats & Chuyển Tiếp Sang Node Tiếp Theo

* **Ghi Nhận Thống Kê (`extraction_stats`)**:
  * Lưu lại nhật ký thống kê chi tiết của lượt trích xuất:
    ```python
    ctx.state.extraction_stats = {
        "tool": {
            "duration_ms": 0,
            "field_count": 0,  # 0 vì không có tool100
            "error": None,
        },
        "llm": {
            "duration_ms": llm_duration_ms,
            "field_count": llm_count,  # Tổng số field LLM bóc được
            "error": llm_error,
        },
        "merged_field_count": merged_count,
    }
    ```
* **Chuyển Giao Luồng**:
  * Trả về `IntentDetectionNode()`, chính thức hoàn thành nhiệm vụ của `MetadataExtractionNode` và chuyển sang giai đoạn phát hiện ý định nâng cao/xử lý nghiệp vụ tiếp theo trong Graph.

---

## 3. Tổng Kết Đánh Giá Luồng Khi Vắng Tool100

1. **Tính Kháng Lỗi Cao (High Fault Tolerance)**: Nhờ thiết kế phân nhóm độc lập và cô lập lỗi per-unit, việc không có `tool100` không làm đứt gãy luồng xử lý của hệ thống.
2. **Đảm Bảo Chất Lượng Dữ Liệu (Data Integrity)**: Quy trình kiểm duyệt đối chiếu `MetadataRegistry` ở Bước 4 và xử lý Regex C-Fail ở Bước 5 đảm bảo dữ liệu do LLM bóc tách luôn chính xác và an toàn.
3. **Đổi Lại Phí Latency/Token**: Việc dồn 100% công việc cho LLM sẽ khiến thời gian phản hồi (latency) và chi phí token của lượt chat tăng lên so với khi có `tool100` hỗ trợ lọc bớt dữ liệu quy tắc.

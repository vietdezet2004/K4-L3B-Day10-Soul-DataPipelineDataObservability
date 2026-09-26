# Group Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin bài nộp

| Thông tin         | Nội dung                  |
| ------------------ | -------------------------- |
| Khóa/Lớp         | K4              |
| Tên nhóm         | Soul     |
| Repository         | https://github.com/vietdezet2004/K4-L3B-Day10-Soul-DataPipelineDataObservability.git |
| Ngày hoàn thành | 2026-09-26               |

### Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò chính | Module/deliverable sở hữu |
| --: | --- | --- | --- | --- |
| 1 | Nguyễn Công Duẩn | 2A202602716 | Trưởng nhóm / Pipeline Integrator | `core/config.py`, `src/pipelines/phase1.py`, `src/pipelines/corruption_flow.py`, `src/retrieval/index.py` |
| 2 | Phùng Quốc Việt | 2A202602456 | Data Ingestion & Recovery | `src/ingestion/crossref.py`, `src/ingestion/cleaning.py`, `src/ingestion/corruption.py`, raw data |
| 3 | Phan Hoàng Vũ | 2A202602450 | Data Observability & Evaluation | `src/observability/quality.py` (GX 1.x + Freshness), `src/evaluation/testset.py`, `src/observability/reporting.py` |

--- 

## 2. Tóm tắt kết quả

Nhóm Soul đã hoàn thành trọn vẹn 100% mục tiêu của bài Lab Day 10 từ Checkpoint 0 đến Checkpoint 5. Hệ thống pipeline thu thập 24 bài báo khoa học từ Crossref API (hỗ trợ fallback snapshot offline), tiền xử lý bóc tách thẻ XML JATS, khử trùng lặp và tính toán `age_days` cùng `text_for_embedding` 5 phần chuẩn hóa. 

Tại Phase 1 Baseline, chốt kiểm dịch Great Expectations 1.x (4 Expectations) và Freshness SLA đều đạt chuẩn (PASSED/FRESH), vector database ChromaDB nạp 24 embeddings phục vụ RAG Agent đạt **100.0% Retrieval Hit Rate** và **1.0000 Mean Token F1**.

Tại Phase 2, nhóm thiết lập 6 kịch bản tiêm lỗi dữ liệu tổng hợp (Drop 20% bài mới, Blank summary, Noise injection, Truncate title, Stale date, Duplicate rows). Kết quả đo lường hiện tượng **Silent Failure** cho thấy: dù hệ thống AI không hề crash hay báo lỗi runtime, hiệu năng truy vấn sụt giảm nghiêm trọng từ **100.0% xuống 60.0%** Hit Rate và Token F1 giảm một nửa từ **1.0000 xuống 0.5000**. Chốt kiểm dịch GX 1.x và Freshness SLA đã cảnh báo chính xác vi phạm (FAILED/STALE). 

Nhờ nguyên tắc Raw Snapshot Preservation, cơ chế tự phục hồi **Idempotent Repair** đã tái tạo lại toàn bộ dữ liệu sạch và vector index, phục hồi toàn bộ chỉ số về mốc hoàn hảo ban đầu (100.0% Hit Rate, 1.0000 F1). Giới hạn còn lại là dữ liệu hiện tại dừng ở quy mô snapshot nhỏ và có thể mở rộng thêm mô hình Judge LLM tự động trực tiếp trên đám mây.

---

## 3. Kiến trúc và luồng dữ liệu

### Luồng end-to-end

```text
Crossref REST API (hoặc Snapshot data/raw/crossref_response.json)
    │
    ▼
[CP0: Ingestion] ➔ data/raw/crossref_records.json (24 records)
    │
    ▼
[CP1: Cleaning] ➔ Bóc tách XML JATS, tính age_days, ghép text_for_embedding ➔ papers_clean.csv/.json
    │
    ├── [CP1: Data Observability Gate] ➔ Great Expectations 1.x + Freshness SLA ➔ baseline_quality_report.json
    │
    ▼
[CP2: Embedding & Indexing] ➔ all-MiniLM-L6-v2 ➔ ChromaDB collection 'papers-baseline'
    │
    ├── [CP2: Evaluation Benchmark] ➔ testset.py sinh 10 câu hỏi ground-truth ➔ test_set.json
    │
    ▼
[CP3: Baseline Run] ➔ Evaluation trên papers-baseline ➔ baseline_metrics.json (Hit Rate: 100%)
    │
    ▼
[CP4: Synthetic Corruption] ➔ Tiêm 6 lỗi ➔ papers-corrupted ➔ corrupted_metrics.json (Hit Rate: 60%)
    │                                         └── GX Gate FAILED & Freshness STALE
    ▼
[CP5: Idempotent Repair] ➔ Đọc từ raw snapshot ➔ Clean lại ➔ papers-repaired ➔ repaired_metrics.json (100%)
    │
    ▼
[CP5: Báo Cáo Đối Chiếu] ➔ generate_corruption_report ➔ data/reports/corruption_report.md
```

### Trách nhiệm của từng khối

| Khối             | Input          | Xử lý chính             | Output/artifact          | Owner          |
| ----------------- | -------------- | -------------------------- | ------------------------ | -------------- |
| Ingestion         | Crossref REST API / Snapshot | Fetch HTTP GET, timeout, retry 3 lần, fallback snapshot offline | `data/raw/crossref_records.json` | Phùng Quốc Việt (TV1) |
| Cleaning          | `crossref_records.json` | Khử XML `<jats:p>`, khử trùng `paper_id`, tính `age_days`, tạo `text_for_embedding` | `data/clean/papers_clean.csv`, `.json` | Phùng Quốc Việt (TV1) |
| Embedding/index   | `papers_clean.json` | Sinh dense vector qua `all-MiniLM-L6-v2`, nạp ChromaDB persistent | `data/chroma/`, `data/embeddings/` | Nguyễn Công Duẩn (TV3) |
| Evaluation        | Clean papers & Chroma collection | Sinh 10 benchmark Q&A bao phủ 4 nghiệp vụ, đo Hit Rate & Token F1 | `data/eval/test_set.json`, `data/results/*_metrics.json` | Phan Hoàng Vũ (TV2) |
| Observability     | Clean/Corrupted DataFrame | GX 1.x Ephemeral context (4 Expectations) & Freshness SLA (>180 ngày) | `data/quality/*_quality_report.json`, `*_freshness_report.json` | Phan Hoàng Vũ (TV2) |
| Corruption/repair | Clean DataFrame & Raw records | Tiêm 6 kịch bản lỗi, sau đó phục hồi idempotent từ raw records | `data/results/corruption_log.json`, `data/clean/*_repaired.*` | Phùng Quốc Việt & Nguyễn Công Duẩn |
| Orchestration     | Toàn bộ các module | Kết nối luồng Phase 1 và Phase 2 Corruption flow, xuất báo cáo | `script/run_phase1.py`, `script/run_corruption_flow.py` | Nguyễn Công Duẩn (TV3) & Phan Hoàng Vũ (TV2) |

---

## 4. Cách tái hiện kết quả

### Cấu hình không chứa secret

| Biến/cấu hình             | Giá trị sử dụng |
| ---------------------------- | ------------------- |
| `LLM_PROVIDER`             | `gemini` (hoặc mock fallback) |
| `LLM_MODEL`                | `gemini-2.5-flash` |
| Embedding model              | `sentence-transformers/all-MiniLM-L6-v2` |
| Số lượng Crossref records | `24` bài báo khoa học |
| Retrieval `top_k`           | `3` văn bản liên quan nhất |
| Freshness threshold          | `180` ngày (cảnh báo nếu >25% bài quá hạn) |
| Random seed                 | `42` (cố định để đảm bảo tính tái lập) |

### Lệnh cài đặt

Kích hoạt môi trường và cài đặt dependencies từ requirements/pyproject:

```bash
.\.venv\Scripts\activate
pip install -r requirements.txt
```

### Lệnh chạy

1. **Chạy Baseline Pipeline (Phase 1):**
```bash
python script/run_phase1.py
```

2. **Chạy Corruption & Idempotent Repair Flow (Phase 2):**
```bash
python script/run_corruption_flow.py
```

### Kết quả tái hiện

| Lệnh             | Trạng thái                                    | Thời điểm chạy gần nhất | Bằng chứng                         |
| ----------------- | ----------------------------------------------- | ----------------------------- | ------------------------------------ |
| Baseline pipeline | Thành công (Exit code 0) | 2026-09-26 22:50 | `data/results/baseline_metrics.json`, `data/reports/phase1_report.md` |
| Corruption flow   | Thành công (Exit code 0) | 2026-09-26 23:07 | `data/results/repaired_metrics.json`, `data/reports/corruption_report.md` |

---

## 5. Ingestion, cleaning và data contract

### Nguồn dữ liệu

| Thuộc tính                | Giá trị                             |
| --------------------------- | ------------------------------------- |
| Source                      | `https://api.crossref.org/works` |
| Query/filter                | `filter=type:journal-article,has-abstract:true`, `rows=24` |
| Thời điểm lấy dữ liệu | 2026-09-26 |
| Số record nhận được    | 24 bản ghi |
| Cơ chế retry/backoff      | Exponential backoff 3 lần; Fallback đọc từ `data/raw/crossref_response.json` khi lỗi mạng |

### Raw và clean schema

| Trường        | Kiểu dữ liệu | Bắt buộc?  | Ý nghĩa   | Xử lý khi thiếu/sai |
| --------------- | --------------- | ------------ | ----------- | ---------------------- |
| `paper_id` | string | Có | Định danh bài báo (DOI) | Chuẩn hóa regex; bỏ record nếu null |
| `title` | string | Có | Tiêu đề bài báo | Strip khoảng trắng, lọc ký tự lỗi |
| `summary` | string | Có | Tóm tắt bài báo | Bóc sạch thẻ XML JATS (`<jats:p>`) |
| `authors` | list[str] | Có | Danh sách tác giả | Gộp thành chuỗi `authors_joined` |
| `published` | string (YYYY-MM-DD) | Có | Ngày xuất bản | Chuẩn hóa định dạng chuẩn ISO |
| `categories` | list[str] | Không | Phân loại chủ đề | Điền rỗng nếu thiếu, gộp `categories_joined` |
| `age_days` | integer | Có | Tuổi của bài báo tính từ ngày chạy | `(now_utc - published_date).days` |
| `text_for_embedding` | string | Có | Toàn văn kết hợp 5 trường để embedding | Ghép Title, Authors, Categories, Date, Summary |

### Quy tắc cleaning

| Quy tắc                                 | Quality dimension liên quan | Số record bị tác động | Cách xác minh      |
| ---------------------------------------- | ---------------------------- | -------------------------: | -------------------- |
| Bóc tách thẻ XML `<jats:p>`, `</jats:p>` | Validity / Consistency | 24 | Regex clean abstract, kiểm tra không còn thẻ `<` |
| Khử trùng lặp theo `paper_id` | Uniqueness | 0 (trên dữ liệu gốc) | Great Expectations `ExpectColumnValuesToBeUnique` |
| Tính toán `age_days` | Freshness / Timeliness | 24 | Kiểm tra `age_days >= 0` |
| Ghép chuỗi `text_for_embedding` 5 phần | Completeness | 24 | Kiểm tra độ dài ký tự `summary_chars > 0` |

---

## 6. Evaluation setup

| Thành phần                             | Cấu hình thực tế          |
| ---------------------------------------- | ----------------------------- |
| Số câu hỏi                            | 10 câu hỏi benchmark chuẩn |
| Các `question_type`                    | `summary` (4 câu), `authors` (2 câu), `date` (2 câu), `categories` (2 câu) |
| Ground-truth document ID                 | Khớp chính xác với trường `paper_id` của tài liệu gốc tương ứng |
| Embedding model                          | `sentence-transformers/all-MiniLM-L6-v2` (384 dimensions) |
| Vector store/collection                  | ChromaDB: `papers-baseline`, `papers-corrupted`, `papers-repaired` |
| Retrieval `top_k`                       | `3` |
| LLM provider/model                       | `gemini-2.5-flash` / Local evaluation matcher |
| Test set dùng chung cho ba trạng thái | `data/eval/test_set.json` |

Test set được giữ nguyên cố định (Frozen Benchmark) xuyên suốt 3 pha nhằm đảm bảo tính công bằng và kiểm chứng khách quan tác động của lỗi dữ liệu (Data Corruption) cũng như năng lực phục hồi của pipeline mà không bị ảnh hưởng bởi sự thay đổi của bộ đề thi.

---

## 7. Kết quả baseline

### Artifact checklist

| Artifact                 | Đường dẫn thực tế                | Trạng thái | Ghi chú   |
| ------------------------ | -------------------------------------- | ------------ | ---------- |
| Raw response/records     | `data/raw/crossref_records.json` | Đầy đủ | 24 bài báo nguyên bản |
| Cleaned dataset          | `data/clean/papers_clean.csv`, `.json` | Đầy đủ | Đã clean và sinh embedding text |
| Embedding manifest/index | `data/embeddings/papers_embeddings.json` | Đầy đủ | 24 vector embeddings |
| Evaluation set           | `data/eval/test_set.json` | Đầy đủ | 10 câu hỏi benchmark chuẩn |
| Baseline metrics         | `data/results/baseline_metrics.json` | Đầy đủ | Hit Rate 100%, F1 1.0000 |
| Quality/freshness        | `data/quality/baseline_quality_report.json` | Đầy đủ | GX Passed, Freshness SLA đạt |
| Baseline report          | `data/reports/phase1_report.md` | Đầy đủ | Báo cáo Markdown Phase 1 |

### Baseline metrics

| Metric                 |       Giá trị | Diễn giải                             |
| ---------------------- | --------------: | --------------------------------------- |
| `retrieval_hit_rate` |     **100.0%** | Toàn bộ 10/10 câu hỏi đều truy vấn trúng bài báo chứa ground truth |
| `mean_token_f1`      |     **1.0000** | Độ trùng khớp từ ngữ tuyệt đối giữa câu trả lời và ground truth |
| `judge_accuracy`     |     **100.0%** | Ngữ nghĩa câu trả lời chính xác hoàn toàn |
| `mean_judge_score`   |     **5.00/5.0** | Điểm số tuyệt đối |

---

## 8. Data quality và freshness

### Quality checks (Great Expectations 1.x)

| Check | Quality dimension | Ngưỡng/kỳ vọng | Kết quả baseline | Bằng chứng |
| ----- | ----------------- | -------------- | ---------------- | ---------- |
| `ExpectTableRowCountToBeBetween` | Completeness | Min=10, Max=1000 | PASS (24 dòng) | `baseline_quality_report.json` |
| `ExpectColumnValuesToNotBeNull` | Completeness | Column `paper_id` not null | PASS (0% null) | `baseline_quality_report.json` |
| `ExpectColumnValuesToBeUnique` | Uniqueness | Column `paper_id` unique | PASS (100% unique) | `baseline_quality_report.json` |
| `ExpectColumnValueLengthsToBeBetween` | Validity | Column `title` length >= 8 | PASS (độ dài hợp lệ) | `baseline_quality_report.json` |

### Freshness

| Thuộc tính               | Giá trị                           |
| -------------------------- | ----------------------------------- |
| Freshness được đo tại | Clean DataFrame (`data/clean/papers_clean.csv`) |
| Timestamp mới nhất       | Ngày xuất bản gần nhất trong dataset (2026) |
| Ngưỡng freshness         | `age_days <= 180` (tối đa 25% bài được phép vượt ngưỡng) |
| Trạng thái baseline      | **FRESH (True)** |
| Lý do                     | 0/24 bài báo quá hạn 180 ngày (tỷ lệ vi phạm 0.0% < 25.0%) |

---

## 9. Corruption scenarios và repair

| Corruption | Cách tạo | Record bị tác động | Quality signal kỳ vọng | Tác động thực tế | Cách repair |
| ---------- | -------- | -----------------: | ---------------------- | ---------------- | ----------- |
| 1. Drop latest | Bỏ 20% bài có ngày xuất bản mới nhất | 4 bài | Giảm số dòng dữ liệu | Không tìm thấy bài báo mới | Tải lại từ raw snapshot |
| 2. Blank summary | Gán `summary = ""` cho 25% số dòng | 6 bài | Expect null / rỗng | Mất ngữ cảnh tóm tắt | Đọc lại abstract gốc từ snapshot |
| 3. Inject noise | Thêm chuỗi rác `@@#$! CORRUPTED DATA !$%#@@` | 6 bài | Vector drift | Vector bị sai lệch không gian | Làm sạch lại từ bản ghi gốc |
| 4. Truncate title | Cắt tiêu đề còn dưới 5 ký tự | 6 bài | GX Length Check FAILED | Truy vấn theo title thất bại | Khôi phục tiêu đề đầy đủ |
| 5. Stale date | Lùi published về `2016-01-01` (`age_days=3800`) | 7 bài | Freshness SLA STALE | Cảnh báo quá hạn dữ liệu | Tính lại `age_days` từ ngày gốc |
| 6. Duplicate rows | Nhân bản 3 dòng dữ liệu | 3 dòng | GX Unique Check FAILED | Dữ liệu trùng lặp | Khử trùng lặp qua `paper_id` |

**Corruption log:**
- Đường dẫn: `data/results/corruption_log.json`
- Trạng thái: Đầy đủ 6 kịch bản, chi tiết số lượng bản ghi và danh sách `paper_id` bị tác động.

**Cơ chế Idempotent Repair:**
Dữ liệu được sửa chữa bằng cách đọc trực tiếp từ bản chụp nguyên trạng ban đầu (`data/raw/crossref_records.json`), chạy lại toàn bộ quy trình tiền xử lý, chuẩn hóa và kiểm dịch chất lượng để nạp vào collection mới `papers-repaired`. Quá trình này không phụ thuộc vào trạng thái lỗi và có tính tất định (chạy nhiều lần vẫn ra cùng kết quả chuẩn).

---

## 10. So sánh baseline, corrupted và repaired

| Metric/signal            | Baseline | Corrupted | Repaired | Thay đổi do corruption | Mức phục hồi | Nhận xét   |
| ------------------------ | -------: | --------: | -------: | -----------------------: | --------------: | ------------ |
| `retrieval_hit_rate`   |   100.0% |     60.0% |   100.0% |                   -40.0% |          +40.0% | Phục hồi hoàn hảo |
| `mean_token_f1`        |   1.0000 |    0.5000 |   1.0000 |                  -0.5000 |         +0.5000 | Trùng khớp từ vựng tối đa |
| `judge_accuracy`       |   100.0% |     50.0% |   100.0% |                   -50.0% |          +50.0% | Đúng ngữ nghĩa 100% |
| `mean_judge_score`     |     5.00 |      3.00 |     5.00 |                    -2.00 |           +2.00 | Điểm đánh giá trở lại 5/5 |
| Quality checks pass/fail |     PASS |      FAIL |     PASS | Bắt trúng vi phạm schema |     Đạt kiểm duyệt | GX 1.x cảnh báo chính xác |
| Freshness status         |    FRESH |     STALE |    FRESH |     Vi phạm SLA 180 ngày |       Hết vi phạm | Bắt trúng 30% bài cũ |

### Hai kết luận nhân quả hỗ trợ bởi artifacts:
1. **[Data Corruption (Drop + Truncate + Noise)] ➔ [GX Gate FAILED & Freshness STALE] ➔ [Hit Rate giảm từ 100% xuống 60%, F1 giảm từ 1.0 xuống 0.5]**: Chứng minh hiện tượng Silent Failure—hệ thống RAG vẫn sinh câu trả lời nhưng câu trả lời sai lệch hoặc thiếu thông tin nghiêm trọng.
2. **[Idempotent Repair từ Raw Snapshot] ➔ [GX Gate & Freshness PASSED trở lại] ➔ [Hit Rate và Token F1 phục hồi 100% về mức Baseline ban đầu]**: Chứng minh tầm quan trọng của kiến trúc Raw Data Preservation giúp khắc phục triệt để lỗi mà không cần gọi lại API ngoài.

---

## 11. Vấn đề tích hợp quan trọng

- **Triệu chứng:** Khi chạy script `run_corruption_flow.py` trên môi trường Windows PowerShell, chương trình xuất báo cáo Markdown thành công nhưng gặp lỗi `UnicodeEncodeError: 'charmap' codec can't encode character '\u1ea2'` tại bước in bảng đối chiếu ra terminal console.
- **Nguyên nhân:** Bảng console sử dụng các ký tự tiếng Việt có dấu (`BẢNG ĐỐI CHIẾU...`), trong khi bảng mã mặc định của Windows console là CP1252.
- **Cách xử lý:** Cấu hình tự động `sys.stdout.reconfigure(encoding='utf-8')` ngay đầu luồng thực thi trong `src/pipelines/corruption_flow.py`.
- **Cách xác minh:** Chạy lại `python script/run_corruption_flow.py`, toàn bộ pipeline hoàn thành trơn tru với **Exit Code 0** và hiển thị đẹp mắt bảng so sánh 3 trạng thái.

---

## 12. Giới hạn và hướng cải thiện

| Giới hạn hiện tại | Ảnh hưởng   | Hướng cải thiện có thể kiểm chứng |
| --------------------- | -------------- | ----------------------------------------- |
| Quy mô dữ liệu demo 24 bài báo | Chưa bao quát tải lớn hàng chục nghìn bài | Thêm module Streaming Ingestion và batching ChromaDB |
| Đánh giá Judge LLM phụ thuộc API mạng | Có thể gặp độ trễ hoặc quota limit | Tích hợp local LLM (Ollama / Qwen) làm Judge dự phòng |

---

## 13. Checklist trước khi nộp

- [x] Thông tin nhóm và repository chính xác.
- [x] Phân công khớp với module, artifact và kết quả thực tế.
- [x] Lệnh tái hiện đã được chạy lại trên phiên bản dùng để nộp (Exit Code 0).
- [x] Baseline, corrupted và repaired dùng cùng evaluation set (`data/eval/test_set.json`).
- [x] Bảng metrics khớp với các file trong `data/results/`.
- [x] Quality/freshness conclusions khớp với `data/quality/`.
- [x] Các đường dẫn báo cáo và artifact truy cập được.
- [x] Mỗi thành viên đã hoàn thành báo cáo vai trò riêng.
- [x] Không có `.env`, API key, token hoặc secret trong source, report, log hay ảnh.

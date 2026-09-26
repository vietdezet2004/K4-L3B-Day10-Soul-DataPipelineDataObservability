# Member Role Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin cá nhân

| Thông tin | Nội dung |
| ------------------ | -------------------------- |
| Họ và tên | Phan Hoàng Vũ |
| MSSV | 2A202602450 |
| Khóa/Lớp | K4/L3B |
| Tên nhóm | Soul |
| Vai trò chính | TV2: Data Observability & Benchmark Evaluation |
| Repository | <https://github.com/vietdezet2004/K4-L3B-Day10-Soul-DataPipelineDataObservability.git> |
| Ngày hoàn thành | 2026-09-26 |

---

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

| Module/deliverable | File/hàm phụ trách | Input nhận vào | Output bàn giao | Trạng thái |
| ------------------ | --------------------- | ---------------- | ----------------- | ---------- |
| **Data Quality Gate (CP1)** | `src/observability/quality.py` (`run_data_quality_checks`) | `df: pd.DataFrame` sau làm sạch của TV1 | `data/quality/*_quality_report.json` | Hoàn thành |
| **Freshness SLA Monitoring (CP1)** | `src/observability/quality.py` (`build_freshness_report`) | DataFrame và ngưỡng `age_days > 180` | `data/quality/*_freshness_report.json` | Hoàn thành |
| **Benchmark Test Set (CP2)** | `src/evaluation/testset.py` (`build_synthetic_test_set`) | Cleaned dataset 24 bài báo | `data/eval/test_set.json` (10 câu hỏi ground-truth) | Hoàn thành |
| **Báo Cáo Đối Chiếu 3 Trạng Thái (CP5)** | `src/observability/reporting.py` (`generate_corruption_report`) | Metrics & quality reports của Baseline, Corrupted, Repaired | `data/reports/corruption_report.md` | Hoàn thành |

### Việc hỗ trợ ngoài phạm vi chính

| Hoạt động | Thành viên/module được hỗ trợ | Kết quả |
| --------- | ----------------------------- | ------- |
| Tích hợp chốt kiểm dịch vào luồng Phase 1 | TV3 (`src/pipelines/phase1.py`) | Đảm bảo pipeline dừng lại hoặc cảnh báo nếu Quality Gate không đạt |
| Sửa lỗi encoding terminal trên Windows | TV3 (`src/pipelines/corruption_flow.py`) | Khắc phục `UnicodeEncodeError` khi in bảng đối chiếu UTF-8 ra console |

---

## 3. Kết quả theo vai trò

| Nhiệm vụ đã thực hiện | File/hàm/artifact liên quan | Kết quả bàn giao | Cách xác minh |
| --------------------- | --------------------------- | ---------------- | ------------- |
| Thiết lập GX 1.x Quality Gate | `src/observability/quality.py` | 4 Expectations kiểm tra completeness, uniqueness, validity | `baseline_quality_report.json` (`success=True`) |
| Giám sát Freshness SLA | `src/observability/quality.py` | Cảnh báo vi phạm khi >25% bài báo có `age_days > 180` | `freshness_report.json` (`is_fresh=True`) |
| Sinh Benchmark Testset | `src/evaluation/testset.py` | Bộ đề thi chuẩn 10 câu hỏi bao phủ 4 nhóm nghiệp vụ | `data/eval/test_set.json` |
| Báo cáo so sánh định lượng | `src/observability/reporting.py` | Bảng so sánh đối chiếu định lượng 3 trạng thái | `data/reports/corruption_report.md` |

**Output cụ thể tiêu biểu:**
Báo cáo đối chiếu định lượng [data/reports/corruption_report.md](file:///d:/Hoc/VinAI/K4-L3B-Day10-Soul-DataPipelineDataObservability/data/reports/corruption_report.md) thể hiện đầy đủ bức tranh:

- Baseline: Hit Rate 100%, F1 1.0000, Quality PASS, Freshness FRESH.
- Corrupted: Hit Rate giảm về 60%, F1 giảm về 0.5000, Quality FAIL, Freshness STALE.
- Repaired: Hit Rate phục hồi về 100%, F1 phục hồi về 1.0000, Quality PASS, Freshness FRESH.

---

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết

1. **Ngăn chặn Silent Failure:** Trong các hệ thống RAG truyền thống, dữ liệu bẩn (thiếu trường, text rác, tiêu đề cụt, dữ liệu quá hạn) không làm hệ thống sập nhưng khiến mô hình AI sinh ra câu trả lời sai lệch (hallucination). Cần một chốt kiểm dịch tự động (Quality Gate) chặn dữ liệu xấu trước khi đưa vào Vector Store.
2. **Đánh giá khách quan:** Cần một bộ đề thi Benchmark có ground-truth tài liệu cố định để đo lường chính xác tác động của lỗi và hiệu quả phục hồi.

### Cách triển khai

1. **Great Expectations 1.x Fluent API:**
   Sử dụng Ephemeral Data Context hiện đại của GX 1.x:

   ```python
   context = gx.get_context(mode="ephemeral")
   data_source = context.data_sources.add_pandas(name="papers_source")
   data_asset = data_source.add_dataframe_asset(name="papers_asset")
   batch_def = data_asset.add_batch_definition_whole_dataframe("papers_batch")
   batch = batch_def.get_batch(batch_parameters={"dataframe": df})
   ```

   Khai báo 4 Expectations cốt lõi:
   - `ExpectTableRowCountToBeBetween(min_value=10, max_value=1000)`: Đảm bảo số lượng bài nạp đủ ngưỡng.
   - `ExpectColumnValuesToNotBeNull(column="paper_id")`: Đảm bảo không mất ID tài liệu.
   - `ExpectColumnValuesToBeUnique(column="paper_id")`: Đảm bảo không bị duplicate bản ghi.
   - `ExpectColumnValueLengthsToBeBetween(column="title", min_value=8)`: Đảm bảo tiêu đề không bị cắt cụt.

2. **Freshness SLA Monitoring:**
   Dựa trên trường `age_days = (run_date - published).days`:
   - Xác định bài báo quá hạn: `age_days > 180`.
   - Tính tỷ lệ vi phạm: `stale_ratio = stale_rows / total_rows`.
   - Kết luận `is_fresh = stale_ratio <= 0.25` (cho phép tối đa 25% bài quá 180 ngày).

3. **Sinh Benchmark Testset (`testset.py`):**
   Sinh 10 câu hỏi bao phủ 4 nhóm nghiệp vụ chính:
   - `summary`: Kiểm tra tóm tắt nội dung bài báo.
   - `authors`: Kiểm tra tác giả bài báo.
   - `date`: Kiểm tra ngày xuất bản.
   - `categories`: Kiểm tra phân loại chủ đề.
   Mỗi câu hỏi liên kết với `ground_truth_doc_ids` chứa chính xác `paper_id` của bài báo trong dataset sạch.

4. **Báo Cáo Đối Chiếu 3 Trạng Thái (`reporting.py`):**
   Triển khai hàm `generate_corruption_report` tự động đọc kết quả metrics của 3 pha, tính toán độ suy giảm $\Delta$ và tỷ lệ phục hồi, tạo báo cáo Markdown chuẩn cho nhóm thuyết trình.

### Input, output và contract

| Thành phần | Mô tả |
| ---------- | ----- |
| Input | `df: pd.DataFrame` chứa 24 bài báo với các cột `paper_id`, `title`, `summary`, `published`, `age_days` |
| Output | `test_quality_report.json`, `freshness_report.json`, `test_set.json`, `corruption_report.md` |
| Module phụ thuộc | Nhận dữ liệu sạch từ TV1 (`cleaning.py`) |
| Module sử dụng output | TV3 sử dụng `test_set.json` để chạy `evaluate_pipeline`, và sử dụng kết quả kiểm dịch để quyết định nạp ChromaDB |
| Điều kiện lỗi xử lý | Xử lý DataFrame rỗng, trường hợp thiếu cột, trường hợp không có bài báo nào hợp lệ |

### Cách xác minh

```bash
python -c "from core.config import load_settings; from observability.quality import run_data_quality_checks; import pandas as pd; s=load_settings(); df=pd.read_json(s.paths.clean_json); res=run_data_quality_checks(df, s, 'test'); print(f'Tín hiệu hoàn thành: Quality check status = {res[\"success\"]}')"
```

- **Kết quả mong đợi:** `Quality check status = True`
- **Kết quả thực tế:** Đúng như mong đợi (`success=True`), toàn bộ 4 Expectation đều Pass.

---

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** Great Expectations có sự thay đổi rất lớn về kiến trúc giữa phiên bản cũ (GX 0.18 legacy với `DataContext` cấu hình YAML phức tạp) và phiên bản mới (GX 1.x với Fluent Datasources và Ephemeral Context).
- **Các phương án đã cân nhắc:**
  - *Phương án 1:* Dùng `gx.get_context()` mặc định lưu thư mục `great_expectations/` trên ổ đĩa.
  - *Phương án 2:* Dùng Ephemeral Context trực tiếp trong RAM thông qua `gx.get_context(mode="ephemeral")` kết hợp `add_pandas` và `add_batch_definition_whole_dataframe`.
- **Phương án đã chọn:** Phương án 2 (GX 1.x Ephemeral Mode).
- **Lý do:** Giúp pipeline chạy hoàn toàn tự động, nhẹ, độc lập (stateless), không tạo rác thư mục cấu hình trong git repository, đồng thời phù hợp hoàn hảo với chuẩn CI/CD và kiến trúc Microservices hiện đại.

---

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng/lỗi nguyên văn:**

  ```text
  UnicodeEncodeError: 'charmap' codec can't encode character '\u1ea2' in position 12: character maps to <undefined>
  ```

- **Lệnh hoặc bước tái hiện:** Chạy `python script/run_corruption_flow.py` trên Windows PowerShell khi in bảng kết quả terminal.
- **Nguyên nhân gốc:** Bảng đối chiếu console chứa tiêu đề tiếng Việt có dấu (`BẢNG ĐỐI CHIẾU...`), trong khi stdout trên Windows mặc định mã hóa bằng cp1252 không hỗ trợ các ký tự Unicode tiếng Việt đặc thù.
- **Cách xử lý:** Thêm xử lý `sys.stdout.reconfigure(encoding="utf-8")` an toàn vào luồng pipeline để ép stream stdout sang UTF-8.
- **Cách xác minh sau khi sửa:** Chạy lại `python script/run_corruption_flow.py`, pipeline chạy thông suốt từ đầu đến cuối và in bảng số liệu hoàn hảo với Exit code 0.
- **Điều học được:** Khi phát triển pipeline dữ liệu đa nền tảng, luôn chủ động quản lý bảng mã (encoding) của I/O stream, đặc biệt trên hệ điều hành Windows.

---

## 7. Hiểu biết về luồng end-to-end

1. **Dữ liệu đi từ Crossref đến vector index như thế nào?**
   Dữ liệu thô JSON từ Crossref API được tải về và lưu vào snapshot `crossref_records.json`. Sau đó, module cleaning khử các thẻ XML JATS rác, chuẩn hóa ngày tháng, tính `age_days` và ghép chuỗi đại diện ngữ nghĩa `text_for_embedding` gồm 5 thành phần (Title, Authors, Categories, Date, Summary). Chuỗi này được mô hình `all-MiniLM-L6-v2` mã hóa thành vector 384 chiều và lưu vào ChromaDB.

2. **Evaluation set và ground-truth document IDs dùng để đo retrieval/answer quality ra sao?**
   Mỗi câu hỏi benchmark có một `ground_truth_doc_ids` (ID bài báo gốc) và `ground_truth` (câu trả lời chuẩn). Khi truy vấn, nếu bài báo ID nằm trong Top-3 kết quả trả về của ChromaDB thì được tính là Hit (`retrieval_hit_rate`). Sau đó, câu trả lời do LLM/Agent sinh ra được so sánh với `ground_truth` thông qua Token F1 Score và LLM Judge để chấm điểm độ trung thực ngữ nghĩa.

3. **Quality checks khác freshness monitoring ở điểm nào trong bài lab?**
   - **Quality checks (GX 1.x):** Đo lường tính toàn vẹn về cấu trúc và giá trị của dữ liệu (Completeness, Uniqueness, Validity, Schema constraint) như không null, không trùng ID, tiêu đề đủ dài.
   - **Freshness monitoring:** Đo lường tính thời sự và tính hợp lệ về mặt thời gian (Timeliness), bảo đảm dữ liệu nạp vào hệ thống AI không bị lỗi thời (Stale data) làm suy giảm chất lượng tri thức thời gian thực.

4. **Vì sao phải dùng cùng test set cho baseline, corrupted và repaired?**
   Để đảm bảo tính khoa học và biến kiểm soát (Control Variable). Nếu thay đổi bộ test set giữa các pha, sự thay đổi của các chỉ số hiệu năng (Hit Rate, F1) sẽ bị nhiễu do độ khó của câu hỏi khác nhau chứ không phản ánh đúng tác động thực sự của lỗi dữ liệu và hiệu quả phục hồi.

5. **Repair được xem là thành công dựa trên artifact và metric nào?**
   - Dựa trên artifact: `repaired_quality_report.json` đạt `success=True` (vượt qua 4/4 expectations) và `repaired_freshness_report.json` đạt `is_fresh=True`.
   - Dựa trên metric: `repaired_metrics.json` có `retrieval_hit_rate` phục hồi từ 60.0% trở lại **100.0%** và `mean_token_f1` phục hồi từ 0.5000 trở lại **1.0000**.

---

## 8. Phân tích kết quả

### Metrics chính

| Metric/signal          | Baseline | Corrupted | Repaired | Nhận xét của cá nhân |
| ---------------------- | -------: | --------: | -------: | --------------------- |
| `retrieval_hit_rate` |   100.0% |     60.0% |   100.0% | Suy giảm nặng 40% do drop và truncate title; hồi phục trọn vẹn |
| `mean_token_f1`      |   1.0000 |    0.5000 |   1.0000 | Giảm một nửa do summary bị xóa rỗng hoặc inject noise |
| `judge_accuracy`     |   100.0% |     50.0% |   100.0% | Đánh giá ngữ nghĩa rơi tự do trong pha corrupted |
| `mean_judge_score`   |     5.00 |      3.00 |     5.00 | Điểm chất lượng giảm từ 5 xuống 3 |
| Quality checks         |     PASS |      FAIL |     PASS | Bắt trúng vi phạm uniqueness và độ dài title |
| Freshness status       |    FRESH |     STALE |    FRESH | Bắt trúng kịch bản lùi ngày quá hạn 180 ngày |

### Kết luận từ số liệu

1. **[Data Corruption] ➔ [Quality FAIL & Freshness STALE] ➔ [Hit Rate giảm từ 100% xuống 60%, F1 giảm từ 1.0 xuống 0.5]**:
   Minh chứng sống động cho hiện tượng **Silent Failure**. Không có lỗi runtime nào xảy ra, hệ thống vẫn trả lời nhưng chất lượng câu trả lời đã suy sụp nếu không có Data Observability Gate cảnh báo trước.
2. **[Idempotent Repair] ➔ [Quality PASS & Freshness FRESH] ➔ [Hit Rate đạt 100%, F1 đạt 1.0000]**:
   Việc bảo tồn snapshot thô nguyên bản (`crossref_records.json`) cho phép tái tạo lại toàn bộ serving layer một cách hoàn hảo mà không cần phụ thuộc vào external API hay dữ liệu đã hỏng.

---

## 9. Điều học được và hướng cải thiện

### Ba điều quan trọng nhất

1. **Data Observability không phải là kiểm thử tĩnh:** Cần kết hợp cả Data Quality Gate (cấu trúc) và Freshness SLA (thời gian) để giám sát dữ liệu sống liên tục trong luồng pipeline.
2. **Nguy cơ Silent Failure trong hệ thống AI/RAG:** Một pipeline không có cảnh báo dữ liệu thì mọi lỗi chất lượng dữ liệu ở tầng dưới đều âm thầm lan truyền lên tầng AI, gây sai lệch nghiêm trọng tới người dùng cuối.
3. **Giá trị của Idempotency và Raw Data Preservation:** Giữ gìn nguyên vẹn dữ liệu gốc trước khi biến đổi là nền tảng cốt tử để hệ sinh thái dữ liệu có năng lực tự phục hồi (Self-healing).

### Nếu có thêm thời gian

Tôi muốn tích hợp thêm **Continuous Drift Monitoring** (giám sát độ trôi dạt phân phối vector embedding) để tự động kích hoạt cảnh báo khi phân bố khoảng cách cosine giữa các vector bài báo mới có dấu hiệu bất thường so với phân bố chuẩn.

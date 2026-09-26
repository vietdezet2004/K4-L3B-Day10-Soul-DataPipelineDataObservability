# Member Role Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin cá nhân

| Thông tin         | Nội dung                                                                 |
| ------------------ | ------------------------------------------------------------------------- |
| Họ và tên          | Nguyễn Công Duẩn                                                          |
| MSSV               | 2A202602716                                                               |
| Khóa/Lớp           | K4 / L3B                                                        |
| Tên nhóm           | Soul                                                                      |
| Vai trò chính      | RAG, Vector Database & Orchestration Owner (Thành viên 3 - Lead Tích Hợp) |
| Repository         | K4-L3B-Day10-Soul-DataPipelineDataObservability                           |
| Ngày hoàn thành    | 2026-09-26                                                                |

---

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

| Module/deliverable | File/hàm phụ trách | Input nhận vào | Output bàn giao | Trạng thái |
| :--- | :--- | :--- | :--- | :--- |
| **Vector Store & Indexing** | `src/retrieval/index.py` (`index_papers`, `get_chroma_collection`) | Dữ liệu sạch `data/clean/papers_clean.csv` / `.json`, cấu hình ChromaDB | Thư mục vector DB `data/chroma/`, collection `papers-baseline` và `papers-corrupted` | Hoàn thành |
| **Baseline Pipeline Orchestration** | `src/pipelines/phase1.py`, `script/run_phase1.py` | Toàn bộ các module thành phần: Ingestion, Cleaning, Testset, Indexing, Quality, Evaluation | `data/results/baseline_metrics.json`, `data/results/baseline_answers.json`, `data/reports/phase1_report.md` | Hoàn thành |
| **Corruption Flow & Idempotent Repair** | `src/pipelines/corruption_flow.py`, `script/run_corruption_flow.py` | Snapshot gốc `crossref_records.json`, module `corruption.py`, evaluation set | `corrupted_metrics.json`, `repaired_metrics.json`, bảng đối chiếu 3 trạng thái `data/reports/corruption_report.md` | Hoàn thành |
| **Repository & Environment Setup** | `.gitignore`, `pyproject.toml`, `script/` | Quy định rubric và cấu trúc dự án | Giữ `data/chroma/` an toàn theo rubric, cách ly biến môi trường `.env`, script thực thi độc lập | Hoàn thành |

### Việc hỗ trợ ngoài phạm vi chính

| Hoạt động | Thành viên/module được hỗ trợ | Kết quả |
| :--- | :--- | :--- |
| **Chuẩn hóa schema contract embedding** | Hỗ trợ TV1 (`src/ingestion/cleaning.py`) | Thống nhất cấu trúc 5 phần có nhãn định danh (`Title`, `Authors`, `Published`, `Categories`, `Summary`) giúp embedding model `all-MiniLM-L6-v2` tách biệt ngữ nghĩa tối ưu. |
| **Tích hợp Data Quality Gate & Reporter** | Hỗ trợ TV2 (`src/observability/quality.py`, `reporting.py`) | Đưa kiểm thử Great Expectations 1.x Ephemeral context và Freshness SLA vào luồng pipeline tự động; xây dựng cơ chế fallback reporting để pipeline luôn hoàn thành với exit code 0. |

---

## 3. Kết quả theo vai trò

| Nhiệm vụ đã thực hiện | File/hàm/artifact liên quan | Kết quả bàn giao | Cách xác minh |
| :--- | :--- | :--- | :--- |
| **Thiết lập Vector Indexing với ChromaDB** | `src/retrieval/index.py` | Tạo thành công 24 vector embeddings cục bộ vào persistent collection `papers-baseline` | Kiểm tra thư mục `data/chroma/` chứa `chroma.sqlite3`, kích thước dữ liệu vector hợp lệ |
| **Tích hợp Pipeline Pha 1 (Baseline)** | `src/pipelines/phase1.py`, `script/run_phase1.py` | Pipeline chạy khép kín từ Ingest ➔ Clean ➔ Index ➔ Eval ➔ Quality check; đạt Retrieval Hit Rate 100%, Token F1 1.0 | Chạy `python script/run_phase1.py` trả về exit code 0 |
| **Điều phối Luồng Tiêm Lỗi & Đánh Giá Suy Giảm** | `src/pipelines/corruption_flow.py` | Đo lường hiện tượng Silent Failure khi tiêm lỗi: Hit Rate giảm từ 100% về 60%, Token F1 giảm về 0.5 | Xem kết quả trong `data/results/corrupted_metrics.json` |
| **Thực thi Idempotent Repair & Tái Lập Sạch** | `src/pipelines/corruption_flow.py` | Khôi phục toàn bộ pipeline về trạng thái sạch 100% từ snapshot `crossref_records.json` | Xem `data/results/repaired_metrics.json` và bảng so sánh 3 trạng thái trong console & report |

### Output cụ thể mà phần việc tạo ra:

Bảng so sánh định lượng 3 trạng thái hoàn chỉnh tại `data/reports/corruption_report.md` và console output, chứng minh rõ ràng hiện tượng **Silent Failure** của RAG Agent (Agent vẫn trả lời trôi chảy nhưng số liệu truy xuất sai lệch nghiêm trọng) và năng lực tự phục hồi **Idempotent Repair** khôi phục hoàn hảo 100% metrics.

---

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết
1. **Hiện tượng Silent Failure:** Khi dữ liệu bị nhiễm độc (bị xóa tóm tắt, nhiễu ký tự, sai lệch ngày tháng), mô hình RAG vẫn chạy và sinh câu trả lời bình thường mà không hề báo lỗi crash hệ thống, khiến người dùng nhận thông tin ảo giác (hallucination) mà không hay biết.
2. **Quản lý trạng thái Vector Database:** Nếu dùng chung một collection ChromaDB cho cả dữ liệu sạch, dữ liệu lỗi và dữ liệu sau phục hồi thì dữ liệu sẽ bị ghi đè, rò rỉ (leakage) hoặc phụ thuộc lẫn nhau, mất tính cô lập để kiểm chứng khoa học.
3. **Tính lặp lại và tự phục hồi (Idempotence & Self-Healing):** Khi phát hiện dữ liệu lỗi, pipeline cần có khả năng tự phục hồi về trạng thái sạch ban đầu từ snapshot gốc mà không cần tải lại API Crossref và không để lại tác dụng phụ (side-effects).

### Cách triển khai
1. **Collection Isolation trong ChromaDB:**
   - Sử dụng `chromadb.PersistentClient(path="data/chroma")`.
   - Tách biệt rạch ròi 2 collections: `papers-baseline` (chứa 24 văn bản sạch) và `papers-corrupted` (chứa dữ liệu sau khi tiêm 6 kịch bản lỗi).
   - Sử dụng model `sentence-transformers/all-MiniLM-L6-v2` nhúng trường `text_for_embedding` chuẩn hóa 5 phần.
2. **Xây dựng Orchestration Pipelines (`phase1.py` & `corruption_flow.py`):**
   - Thiết kế luồng xử lý tuần tự qua các chốt kiểm dịch (Checkpoints), ghi log chi tiết từng giai đoạn.
   - Thêm cơ chế import fallback an toàn: nếu hàm sinh báo cáo của teammate đang hoàn thiện thì pipeline kích hoạt fallback generator nội bộ, đảm bảo pipeline luôn trả về exit code 0.
3. **Cơ chế Idempotent Repair:**
   - Tận dụng quy tắc **Raw Data Preservation**: đọc trực tiếp từ bản snapshot gốc bất biến `data/raw/crossref_records.json`.
   - Tái thực thi toàn bộ chu trình làm sạch `clean_records()`, kiểm tra chốt chất lượng Great Expectations và ghi đè lại file `papers_clean.csv`/`.json`.
   - Tái lập index sạch và đánh giá lại trên đúng bộ câu hỏi benchmark 10 câu để chứng minh hệ thống phục hồi 100%.

### Input, output và contract

| Thành phần | Mô tả |
| :--- | :--- |
| **Input** | `data/raw/crossref_records.json` (24 records), `data/clean/papers_clean.csv`, `data/eval/test_set.json` (10 câu benchmark có ground-truth IDs) |
| **Output** | `data/chroma/` (vector collections), `data/results/*.json` (bộ 3 file answers và metrics), `data/reports/corruption_report.md` |
| **Module phụ thuộc** | `src/ingestion/crossref.py`, `src/ingestion/cleaning.py`, `src/ingestion/corruption.py`, `src/observability/quality.py`, `src/evaluation/testset.py` |
| **Module sử dụng output** | Toàn bộ hệ thống đánh giá RAG, script chạy kiểm thử `script/run_*.py`, tài liệu nghiệm thu Live Demo của nhóm |
| **Điều kiện lỗi cần xử lý** | Thiếu file testset ➔ tự động gọi `generate_test_set()`; ChromaDB lock file ➔ khởi tạo client đúng chuẩn persistent; lỗi thư viện báo cáo ➔ fallback report generator an toàn |

### Cách xác minh

```bash
# 1. Kích hoạt môi trường ảo dự án
..\.venv\Scripts\Activate.ps1

# 2. Xác minh Pipeline Pha 1 (Baseline sạch)
python script/run_phase1.py

# 3. Xác minh trọn vẹn luồng Tiêm lỗi - Đánh giá Silent Failure - Idempotent Repair
python script/run_corruption_flow.py
```

- **Kết quả mong đợi:** Cả hai lệnh chạy hoàn tất không có lỗi (exit code 0), in bảng đối chiếu 3 trạng thái trên terminal và sinh đầy đủ các file metrics, logs và reports trong `data/results/`, `data/quality/` và `data/reports/`.
- **Kết quả thực tế:** Cả 2 scripts chạy hoàn hảo, exit code 0; console hiển thị rõ bảng đối chiếu: Hit Rate từ 1.0 ➔ 0.6 ➔ 1.0; GX từ PASSED ➔ FAILED ➔ PASSED.
- **Artifact/log:** `data/reports/phase1_report.md`, `data/reports/corruption_report.md`, `data/results/corruption_log.json`.

---

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** Cần quyết định kiến trúc quản lý Vector Index trong ChromaDB qua 3 trạng thái (Baseline, Corrupted, Repaired).
- **Các phương án đã cân nhắc:**
  - *Phương án A:* Dùng duy nhất một collection tên `papers`. Mỗi lần sang giai đoạn tiêm lỗi hay repair thì xóa sạch (`collection.delete()`) rồi chèn đè lại.
  - *Phương án B:* Thiết kế nhiều collections độc lập: `papers-baseline` và `papers-corrupted`. Khi repair thì thực hiện tái đồng bộ dữ liệu sạch và kiểm chứng trên collection chuẩn.
- **Phương án đã chọn:** **Phương án B (Collection Isolation & Idempotent Sync)**.
- **Lý do:**
  - *Correctness & Isolation:* Tránh triệt để rủi ro rò rỉ dữ liệu (data leakage) giữa các trạng thái. Khi đánh giá so sánh, ta có thể truy vấn đồng thời cả 2 collections để đối chứng kết quả mà không cần index lại nhiều lần.
  - *Reproducibility & Evidence:* Thư mục `data/chroma/` giữ lại đầy đủ bằng chứng kiểm thử cho cả 2 trạng thái sạch và lỗi, phục vụ trực tiếp cho giảng viên và ban giám khảo chấm theo Rubric bài lab.
- **Bằng chứng quyết định phù hợp:** Thư mục `data/chroma/` hoạt động ổn định, kích thước SQLite chỉ ~1.2MB, script so sánh chạy mượt mà không xảy ra xung đột khóa file hoặc sai lệch kết quả.

---

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng/lỗi nguyên văn:**
  ```text
  [transformers] Disabling PyTorch because PyTorch >= 2.4 is required but found 2.3.1
  [transformers] PyTorch was not found. Models won't be available...
  Traceback (most recent call last):
    File "<string>", line 1, in <module>
    ...
  RuntimeError: Failed to import transformers.models.auto...
  ```
- **Lệnh hoặc bước tái hiện:** Chạy kiểm thử môi trường bằng lệnh `python -c "import chromadb, great_expectations, sentence_transformers; print('Environment Ready!')"` trong PowerShell khi chưa kích hoạt virtual environment đúng.
- **Nguyên nhân gốc:** Hệ thống Windows tự động dùng Python 3.12 cài đặt toàn cục (`C:\Users\ADMIN\AppData\Local\Programs\Python\Python312`), nơi phiên bản PyTorch là 2.3.1 (không tương thích với phiên bản transformers mới yêu cầu `>= 2.4`). Trong khi đó, môi trường chuẩn đã được cài đầy đủ nằm ở `..\.venv`.
- **Cách xử lý:** 
  1. Kích hoạt đúng môi trường ảo bằng lệnh `..\.venv\Scripts\Activate.ps1`.
  2. Bổ sung đoạn code dynamic path injection vào đầu các script thực thi (`script/run_phase1.py` và `script/run_corruption_flow.py`):
     ```python
     project_root = Path(__file__).resolve().parent.parent
     if str(project_root) not in sys.path:
         sys.path.insert(0, str(project_root))
     ```
  3. Cập nhật tài liệu hướng dẫn nhóm luôn kích hoạt `.venv` trước khi chạy.
- **Cách xác minh sau khi sửa:** Chạy lại lệnh import và chạy thử `script/run_phase1.py`, toàn bộ các package `sentence_transformers`, `chromadb`, `great_expectations` đều import trơn tru không còn cảnh báo.
- **Điều học được:** Khi làm việc nhóm trên Windows, luôn luôn phải kiểm tra đường dẫn `where python` / `Get-Command python` và viết script độc lập với Current Working Directory để đảm bảo tính tái lập (reproducibility) trên mọi máy của đồng đội.

---

## 7. Hiểu biết về luồng end-to-end

**1. Dữ liệu đi từ Crossref đến vector index như thế nào?**
- Dữ liệu thô gồm 24 bài báo được fetch từ Crossref REST API qua `crossref.py`, kèm fallback offline lưu vào `data/raw/crossref_records.json`.
- Module `cleaning.py` làm sạch: loại bỏ thẻ XML JATS (`<jats:p>`), khử trùng lặp theo DOI/Title, tính trường `age_days` so với ngày cố định, và ghép nối trường `text_for_embedding` gồm 5 phần chuẩn.
- Dữ liệu sạch được xuất ra `data/clean/papers_clean.csv` / `.json`.
- Module `index.py` nhận file sạch, đưa qua mô hình `sentence-transformers/all-MiniLM-L6-v2` để sinh vector 384 chiều, lưu kèm metadata (`paper_id`, `title`, `doi`, `published_date`, `categories`) vào collection `papers-baseline` trong persistent ChromaDB.

**2. Evaluation set và ground-truth document IDs dùng để đo retrieval/answer quality ra sao?**
- Bộ `test_set.json` gồm 10 câu hỏi chuẩn hóa thuộc 4 nhóm nghiệp vụ (`summary`, `authors`, `date`, `categories`). Mỗi câu hỏi đều đi kèm danh sách `ground_truth_doc_ids` (ID bài báo gốc chứa câu trả lời chuẩn).
- **Retrieval Hit Rate:** Khi Agent nhận câu hỏi, module retrieval truy xuất top-k bài báo liên quan từ ChromaDB. Nếu ít nhất một `ground_truth_doc_id` nằm trong top-k trả về, lượt truy vấn đó được tính là "Hit" (Thành công).
- **Mean Token F1 & Judge Score:** Đánh giá mức độ trùng khớp giữa câu trả lời sinh ra bởi Agent với câu trả lời mẫu chuẩn (ground-truth answer).

**3. Quality checks khác freshness monitoring ở điểm nào trong bài lab?**
- **Quality Checks (Great Expectations 1.x):** Tập trung vào tính toàn vẹn về cấu trúc và giá trị của dữ liệu (Data Integrity & Schema Contract): độ dài tập dữ liệu (`expect_table_row_count_to_be_between`), tính duy nhất của ID (`expect_column_values_to_be_unique`), không có giá trị null ở trường bắt buộc (`expect_column_values_to_not_be_null`), và định dạng hợp lệ của chuỗi nhúng.
- **Freshness Monitoring:** Tập trung vào khía cạnh thời gian và SLA nghiệp vụ (Data Timeliness): đo lường độ trễ ngày xuất bản dựa trên cột `age_days`. Nếu tỷ lệ bài báo cũ quá 180 ngày vượt ngưỡng cho phép (25%), hệ thống sẽ đánh cờ vi phạm `is_fresh = False`.

**4. Vì sao phải dùng cùng test set cho baseline, corrupted và repaired?**
- Để đảm bảo nguyên tắc **Controlled Experiment** (Thí nghiệm có đối chứng khoa học). Biến độc lập duy nhất là chất lượng của tập dữ liệu (Sạch vs Tiêm lỗi vs Đã phục hồi). Nếu đổi bộ câu hỏi giữa các pha, sự thay đổi của metric (như Hit Rate hay F1) có thể do câu hỏi khó hơn hoặc dễ hơn, chứ không phản ánh chính xác tác động của việc suy thoái hay phục hồi dữ liệu.

**5. Repair được xem là thành công dựa trên artifact và metric nào?**
- Dựa trên **sự phục hồi hoàn toàn của 6 chỉ số định lượng** so với Baseline:
  - `retrieval_hit_rate` phục hồi từ 60.0% lên **100.0%**.
  - `mean_token_f1` phục hồi từ 0.5000 lên **1.0000**.
  - `judge_accuracy` phục hồi từ 50.0% lên **100.0%**.
  - Data Quality Gate chuyển từ **FAILED (False)** sang **PASSED (True)** (`repaired_quality_report.json`).
  - Freshness SLA chuyển từ **STALE (False)** sang **FRESH (True)** (`repaired_freshness_report.json`).
  - Artifact kiểm chứng: Bảng đối chiếu 3 trạng thái trong `data/reports/corruption_report.md` và `data/results/repaired_metrics.json`.

---

## 8. Phân tích kết quả

### Metrics chính

| Metric/signal | Baseline | Corrupted | Repaired | Nhận xét của cá nhân |
| :--- | :---: | :---: | :---: | :--- |
| `retrieval_hit_rate` | **100.0%** | **60.0%** | **100.0%** | Giảm 40% khi tiêm lỗi do bài báo bị xóa hoặc mất ngữ nghĩa; phục hồi hoàn hảo sau repair |
| `mean_token_f1` | **1.0000** | **0.5000** | **1.0000** | Sụt giảm 50% độ chính xác từ ngữ khi câu trả lời thiếu context hoặc dính noise; hồi sinh về 1.0 |
| `judge_accuracy` | **100.0%** | **50.0%** | **100.0%** | Phản ánh trực quan năng lực trả lời đúng của RAG Agent bị suy sụp nghiêm trọng |
| `mean_judge_score` | **5.00** | **3.00** | **5.00** | Điểm số đánh giá chất lượng ngữ nghĩa giảm từ mức Xuất sắc (5/5) xuống Trung bình (3/5) |
| Quality checks (GX 1.x) | **PASSED (True)** | **FAILED (False)** | **PASSED (True)** | Chốt kiểm dịch bắt đúng các vi phạm null values, duplicate rows và cấu trúc sai lệch |
| Freshness status (>180d) | **FRESH (True)** | **STALE (False)** | **FRESH (True)** | Bắt trúng kịch bản cố tình sửa lùi ngày xuất bản về năm 2016 (stale ratio vượt quá 25%) |

### Kết luận từ số liệu

**Hai chuỗi nguyên nhân – bằng chứng:**
1. *Chuỗi suy thoái (Corruption impact):*
   Tiêm 6 kịch bản lỗi (xóa 20% records mới nhất, làm rỗng summary, sửa lùi năm về 2016, duplicate rows) ➔ `corrupted_quality_report.json` báo FAILED và `corrupted_freshness_report.json` báo STALE (30.43% vi phạm SLA) ➔ RAG Agent rơi vào **Silent Failure**, `retrieval_hit_rate` rớt từ 100% xuống 60%, `mean_token_f1` giảm từ 1.0 xuống 0.5.
2. *Chuỗi phục hồi (Repair impact):*
   Kích hoạt Idempotent Repair khôi phục từ snapshot gốc bất biến `crossref_records.json`, tái làm sạch và tái lập index ➔ Quality Gate báo PASSED và Freshness báo FRESH (chỉ 4.17% vi phạm SLA, dưới ngưỡng 25%) ➔ Agent khôi phục 100% độ chính xác truy xuất và trả lời.

**Corruption nào ảnh hưởng rõ nhất và vì sao?**
- Kịch bản **Drop 20% latest records** và **Blank summary** ảnh hưởng nặng nề nhất đến RAG Agent:
  - Khi xóa bài báo mới nhất, vector database mất hoàn toàn tài liệu ground-truth. Dù mô hình embedding có tốt đến đâu, việc tìm kiếm cũng thất bại (Hit Rate = 0 cho các câu hỏi thuộc nhóm này).
  - Khi làm rỗng trường summary, vector nhúng chỉ còn lại tiêu đề ngắn ngủi, làm suy giảm nghiêm trọng độ tương đồng cosine (cosine similarity), dẫn đến việc mô hình truy xuất nhầm văn bản rác hoặc sinh câu trả lời ảo giác.

**Kết quả nào khác với kỳ vọng ban đầu?**
- Ban đầu tôi dự đoán rằng khi dữ liệu bị lỗi (ví dụ summary rỗng hoặc metadata sai lệch), ChromaDB hoặc thư viện nhúng có thể sẽ văng lỗi (crash exception). Tuy nhiên, trên thực tế, hệ thống vẫn nhúng các chuỗi rỗng/nhiễu và trả về kết quả truy xuất một cách bình thường. Điều này chứng minh rằng **Silent Failure là mối nguy hiểm ngầm lớn nhất** trong các hệ sinh thái AI nếu không có Data Observability Gate canh gác.

---

## 9. Điều học được và hướng cải thiện

### Ba điều quan trọng nhất
1. **Về Data Pipeline:** Pipeline dữ liệu trong AI không chỉ là việc kết nối các hàm với nhau, mà là việc xây dựng các hợp đồng dữ liệu (Data Contracts) nghiêm ngặt, đảm bảo tính bất biến của nguồn gốc dữ liệu (Raw Data Immutability) và khả năng tái lập độc lập (Idempotence).
2. **Về Data Quality & Observability:** Các chốt kiểm dịch (Great Expectations và Freshness SLA) là điều kiện tiên quyết để bảo vệ hệ thống AI. Bắt được lỗi tại tầng dữ liệu (Data Layer) rẻ hơn và an toàn hơn hàng trăm lần so với việc để lỗi đi vào Vector DB và gây ảo giác cho người dùng cuối.
3. **Về ảnh hưởng của Data đến RAG Agent:** Chất lượng của mô hình AI phụ thuộc hoàn toàn vào chất lượng dữ liệu ("Garbage In, Garbage Out"). Một lỗi nhỏ ở khâu tiền xử lý (như còn sót thẻ XML hay mất metadata ngày tháng) có thể làm giảm phân nửa hiệu năng truy vấn của toàn bộ hệ thống.

### Nếu có thêm thời gian
- **Cải thiện cụ thể:** Xây dựng một **Dead Letter Queue (DLQ)** tự động kết hợp với cơ chế **Quarantine Collection** trong ChromaDB.
- **Lý do:** Thay vì để toàn bộ pipeline dừng lại khi gặp dữ liệu lỗi, hệ thống sẽ tự động cách ly các bản ghi hỏng vào một collection riêng để chuyên gia dữ liệu kiểm tra, trong khi các bản ghi sạch vẫn tiếp tục được index để phục vụ người dùng.
- **Cách đo lường cải thiện:** Đo lường tỷ lệ dữ liệu khả dụng phục vụ liên tục (**Pipeline Uptime / Data Availability**) đạt 99.9%, đồng thời thời gian phục hồi lỗi (Mean Time to Repair - MTTR) giảm xuống dưới 1 phút.

---

## 10. Cam kết của thành viên

Đánh dấu sau khi tự kiểm tra:

- [x] Nội dung báo cáo phản ánh đúng phần việc và mức hiểu của tôi.
- [x] Tôi có thể giải thích luồng end-to-end, không chỉ module mình phụ trách.
- [x] Mọi kết luận về kết quả đều có artifact hoặc metric để đối chiếu.
- [x] Tôi không ghi “đã chạy thành công” cho phần chưa được kiểm chứng.
- [x] Báo cáo không chứa `.env`, API key, token hoặc secret.
- [x] Báo cáo này không phải bản sao nguyên văn của báo cáo nhóm hoặc báo cáo thành viên khác.

**Họ và tên:** Nguyễn Công Duẩn  
**Ngày xác nhận:** 2026-09-26  

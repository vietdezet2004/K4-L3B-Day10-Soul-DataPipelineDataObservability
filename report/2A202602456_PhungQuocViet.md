# Member Role Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin cá nhân

| Thông tin         | Nội dung                                                                  |
| ------------------ | -------------------------------------------------------------------------- |
| Họ và tên          | Phùng Quốc Việt                                                           |
| MSSV               | 2A202602456                                                               |
| Khóa/Lớp           | K4 / L3B                                                                  |
| Tên nhóm           | Soul                                                                      |
| Vai trò chính      | TV1: Data Ingestion, Data Cleaning & Data Corruption Owner (Tầng Dữ Liệu) |
| Repository         | https://github.com/vietdezet2004/K4-L3B-Day10-Soul-DataPipelineDataObservability.git |
| Ngày hoàn thành    | 2026-09-26                                                                |

---

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

| Module/deliverable | File/hàm phụ trách | Input nhận vào | Output bàn giao | Trạng thái |
| ------------------ | --------------------- | ---------------- | ----------------- | ---------- |
| **Raw Data Ingestion (CP0)** | `src/ingestion/crossref.py` (`fetch_source_records`, `parse_crossref_payload`, `load_raw_records`) | Crossref REST API query hoặc snapshot local | `data/raw/crossref_records.json` (24 bài), `crossref_response.json` | Hoàn thành |
| **Data Cleaning & Modeling (CP1)** | `src/ingestion/cleaning.py` (`build_clean_dataframe`, `_clean_text`) | `list[PaperRecord]` từ raw snapshot | `data/clean/papers_clean.csv` và `papers_clean.json` (24 dòng sạch) | Hoàn thành |
| **Baseline Reporting (CP3)** | `src/observability/reporting.py` (`generate_phase1_report`) | Source summary, metrics, GX report, Freshness report | `data/reports/phase1_report.md` | Hoàn thành |
| **Synthetic Corruption Suite (CP4)** | `src/ingestion/corruption.py` (6 kịch bản lỗi, `corrupt_dataset`, `rebuild_text_for_embedding`) | `papers_clean.json` (dữ liệu sạch 24 bài) | `data/results/corruption_log.json`, `data/clean/papers_clean_corrupted.*` | Hoàn thành |
| **Idempotent Repair Foundation (CP5)** | `src/ingestion/cleaning.py` & snapshot gốc | Snapshot bất biến `crossref_records.json` | Khôi phục `papers_clean_repaired.json` sạch 100% đồng nhất | Hoàn thành |

### Việc hỗ trợ ngoài phạm vi chính

| Hoạt động | Thành viên/module được hỗ trợ | Kết quả |
| --------- | ----------------------------- | ------- |
| Thống nhất Schema dữ liệu và Data Contracts | TV2 (`src/observability/quality.py`) | Cung cấp đủ các trường `paper_id`, `title`, `summary`, `age_days` cho chốt kiểm dịch Great Expectations 1.x |
| Thống nhất định dạng nhúng cho ChromaDB | TV3 (`src/retrieval/index.py`) | Chuẩn hóa cấu trúc trường `text_for_embedding` đúng 5 phần giúp mô hình `all-MiniLM-L6-v2` đạt độ tương đồng tối đa |
| Khắc phục lỗi UTF-8 trên Windows PowerShell | Cả nhóm | Hướng dẫn cấu hình `$env:PYTHONIOENCODING="utf-8"` để các lệnh kiểm thử và in bảng đối chiếu không bị lỗi font |

---

## 3. Kết quả theo vai trò

| Nhiệm vụ đã thực hiện | File/hàm/artifact liên quan | Kết quả bàn giao | Cách xác minh |
| --------------------- | --------------------------- | ---------------- | ------------- |
| Thu thập 24 bản ghi thô có Offline Fallback | `src/ingestion/crossref.py` | 24 records JSON chuẩn hóa không mất mát trường dữ liệu | `data/raw/crossref_records.json` (24 bài) |
| Làm sạch thẻ JATS XML, tính `age_days`, tạo embedding text | `src/ingestion/cleaning.py` | DataFrame 24 dòng sạch không còn thẻ HTML/XML, có trường `text_for_embedding` 5 phần | `data/clean/papers_clean.json`, lệnh test clean thành công 24 dòng |
| Lập trình báo cáo Pha 1 chuẩn Markdown | `src/observability/reporting.py` | Báo cáo chi tiết về Ingestion, Quality Gate và Metrics Baseline | `data/reports/phase1_report.md` |
| Triển khai trọn vẹn 6 kịch bản tiêm lỗi dữ liệu | `src/ingestion/corruption.py` | Bộ dữ liệu bẩn kèm log chi tiết ghi lại danh sách ID bị biến đổi | `data/results/corruption_log.json` ghi đủ 6 kịch bản |
| Đảm bảo năng lực phục hồi Idempotent | `src/ingestion/cleaning.py` + raw snapshot | Tái tạo lại dữ liệu sạch 100% từ snapshot thô gốc ban đầu | `data/clean/papers_clean_repaired.json` khớp hoàn toàn baseline |

**Output cụ thể tiêu biểu:**
File nhật ký lỗi [data/results/corruption_log.json](file:///c:/Users/Phung%20Quoc%20Viet/Desktop/AI_in_Action/LAB_10/K4-L3B-Day10-Soul-DataPipelineDataObservability/data/results/corruption_log.json) và tập dữ liệu lỗi [data/clean/papers_clean_corrupted.json](file:///c:/Users/Phung%20Quoc%20Viet/Desktop/AI_in_Action/LAB_10/K4-L3B-Day10-Soul-DataPipelineDataObservability/data/clean/papers_clean_corrupted.json) là mắt xích cốt lõi để kích hoạt toàn bộ kịch bản đo lường Silent Failure của AI:
- Ghi nhận chính xác 4 bài báo mới nhất bị loại bỏ (`drop_latest_records`).
- 5 bài báo bị xóa trắng phần tóm tắt (`blank_summary`).
- 5 bài báo bị chèn ký tự nhiễu rác `@@#$! CORRUPTED DATA !$%#@@` (`inject_noise`).
- 5 bài báo bị cắt ngắn tiêu đề xuống dưới 8 ký tự (`truncate_title`).
- 6 bài báo bị lùi ngày xuất bản về năm 2016 (`age_days = 3800`) khiến tỷ lệ bài cũ tăng lên 30.43% vi phạm Freshness SLA (`stale_date`).
- 3 bài báo bị nhân bản trùng lặp ID (`duplicate_rows`).

---

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết
1. **Tính bất định của API ngoài (External API Brittleness):** Crossref REST API thường xuyên gặp lỗi mạng, rate-limit `429 Too Many Requests` hoặc trả về payload rác chứa nhiều thẻ định dạng XML nội bộ của nhà xuất bản (`<jats:p>`, `<jats:sec>`, `<b>`, `<i>`). Nếu không có tầng bóc tách và fallback, pipeline dữ liệu sẽ bị gián đoạn ngay từ đầu nguồn.
2. **Cấu trúc hóa dữ liệu ngữ nghĩa (Semantic Data Modeling for Vector Search):** Để mô hình nhúng (`all-MiniLM-L6-v2`) hiểu rõ ngữ cảnh khoa học, không thể chỉ nhúng tiêu đề hoặc tóm tắt rời rạc, mà cần một cấu trúc 5 phần chuẩn mực kết hợp đầy đủ: Tiêu đề, Tác giả, Chuyên ngành, Ngày xuất bản và Tóm tắt nội dung.
3. **Mô phỏng sự cố thực tế (Synthetic Data Corruption):** Để chứng minh hiện tượng **Silent Failure** của hệ thống AI (khi dữ liệu lỗi nhưng AI vẫn sinh câu trả lời mà không crash), cần thiết kế các dạng lỗi có chủ đích nhằm phá vỡ các giả định về Data Quality (Tính đầy đủ, Tính duy nhất, Tính hợp lệ, và Tính kịp thời/Freshness).
4. **Bảo toàn nguồn gốc và phục hồi sạch (Data Lineage & Idempotent Repair):** Khi hệ thống phát hiện dữ liệu bẩn, không được phép "chắp vá" trực tiếp trên dữ liệu lỗi (state mutation), mà phải có khả năng tái tạo lại trạng thái sạch từ nguồn dữ liệu gốc bất biến (Single Source of Truth).

### Cách triển khai

1. **Cơ chế Ingestion với Offline Fallback (`crossref.py`):**
   - Khi `settings.refresh_source = True`, pipeline gọi API Crossref. Nếu gặp lỗi HTTP hoặc Timeout, hệ thống tự động bắt ngoại lệ và kích hoạt đọc từ snapshot local `data/raw/crossref_response.json`.
   - Hàm `parse_crossref_payload()` bóc tách an toàn các trường JSON phức tạp, xử lý trường hợp mảng tác giả rỗng, ngày xuất bản ở nhiều định dạng date-parts khác nhau.

2. **Làm sạch và Modeling Text (`cleaning.py`):**
   - Hàm `_clean_text()` sử dụng Regex `re.sub(r"<[^>]+>", " ", text)` để bóc sạch mọi thẻ XML/HTML JATS và chuẩn hóa khoảng trắng thừa.
   - Tính toán tuổi thọ bài báo: `age_days = (run_date_utc - published_datetime).days`.
   - Xây dựng cấu trúc trường `text_for_embedding` 5 phần chuẩn hóa:
     ```text
     Title: <title>
     Authors: <authors_joined>
     Categories: <categories_joined>
     Published Date: <published_str>
     Summary: <summary>
     ```
   - Khử trùng lặp theo `paper_id` và lọc bỏ các bản ghi không hợp lệ trước khi ghi ra CSV/JSON.

3. **Bộ 6 kịch bản tiêm lỗi dữ liệu (`corruption.py`):**
   - `drop_latest_records(df, ratio=0.2)`: Sắp xếp theo ngày xuất bản giảm dần, cắt bỏ 20% bài mới nhất (4 bản ghi), làm mất ngữ cảnh các câu hỏi thời sự.
   - `blank_summary(df, sample_ratio=0.25)`: Xóa rỗng `summary = ""` ở 25% dòng (5 bài), làm mất hoàn toàn ngữ nghĩa cốt lõi.
   - `inject_noise(df, sample_ratio=0.25)`: Chèn chuỗi ký tự rác `@@#$! CORRUPTED DATA !$%#@@` làm xáo trộn vector embedding trong không gian ngữ nghĩa.
   - `truncate_title(df, sample_ratio=0.25)`: Cắt ngắn `title[:5]` vi phạm Expectation độ dài tối thiểu $\ge 8$ ký tự của Great Expectations.
   - `stale_date(df, sample_ratio=0.3)`: Gán ngày xuất bản về `2016-01-01` (`age_days = 3800`) ở 30% bài báo, đẩy tỷ lệ quá hạn lên >25% để kích hoạt cảnh báo vi phạm Freshness SLA.
   - `duplicate_rows(df, num_dups=3)`: Nhân bản 3 dòng dữ liệu giữ nguyên `paper_id` nhằm kích hoạt lỗi Uniqueness.
   - `rebuild_text_for_embedding(df)`: Tái cấu trúc lại toàn bộ trường nhúng phản ánh chính xác các nội dung đã bị tiêm lỗi, đảm bảo lỗi thực sự tác động đến vector database.

4. **Nền tảng Idempotent Repair:**
   - Đảm bảo tính chất hàm thuần khiết (Pure Function): `build_clean_dataframe(load_raw_records(...))` luôn cho ra cùng một kết quả DataFrame bất biến nếu snapshot đầu vào không đổi.

### Input, output và contract

| Thành phần | Mô tả |
| ---------- | ----- |
| **Input** | Snapshot JSON từ Crossref API (`crossref_records.json` chứa 24 bài báo thô) |
| **Output** | `papers_clean.csv/.json`, `corruption_log.json`, `papers_clean_corrupted.csv/.json`, `papers_clean_repaired.csv/.json` |
| **Module phụ thuộc** | `core/config.py` (quản lý đường dẫn `Paths`), `core/utils.py` (hàm I/O) |
| **Module sử dụng output** | TV2 dùng để chạy Data Quality Gate (GX 1.x) & Freshness SLA; TV3 dùng để nạp vào ChromaDB và chạy Agent RAG |
| **Điều kiện lỗi cần xử lý** | Mất mạng/Rate-limit API; thẻ XML JATS lồng ghép; tác giả bị thiếu; định dạng ngày không chuẩn; chuỗi rỗng |

### Cách xác minh

```powershell
$env:PYTHONIOENCODING="utf-8"
# 1. Xác minh Ingestion & Clean data (CP0 & CP1)
.venv\Scripts\python.exe -c "from datetime import datetime, timezone; from core.config import load_settings; from ingestion.crossref import load_raw_records; from ingestion.cleaning import build_clean_dataframe; s=load_settings(); df=build_clean_dataframe(load_raw_records(s.paths.raw_records_json), datetime.now(timezone.utc)); print(f'Tín hiệu hoàn thành: Clean thành công {len(df)} dòng')"

# 2. Xác minh 6 kịch bản tiêm lỗi Corruption (CP4)
.venv\Scripts\python.exe -c "from core.config import load_settings; from ingestion.corruption import corrupt_dataset; import pandas as pd; s=load_settings(); df=pd.read_json(s.paths.clean_json); corrupted_df, log = corrupt_dataset(df, s); print('Tiêm thành công các lỗi:', list(log.keys()))"
```

- **Kết quả mong đợi:** In ra `Clean thành công 24 dòng` và danh sách đầy đủ 6 kịch bản lỗi trong `log.keys()`.
- **Kết quả thực tế:** 
  - `Tín hiệu hoàn thành: Clean thành công 24 dòng`.
  - `Tiêm thành công các lỗi: ['drop_latest_records', 'blank_summary', 'inject_noise', 'truncate_title', 'stale_date', 'duplicate_rows', 'records_before_corruption', 'records_after_corruption']`.
- **Artifact/log:** [data/raw/crossref_records.json](file:///c:/Users/Phung%20Quoc%20Viet/Desktop/AI_in_Action/LAB_10/K4-L3B-Day10-Soul-DataPipelineDataObservability/data/raw/crossref_records.json), [data/clean/papers_clean.json](file:///c:/Users/Phung%20Quoc%20Viet/Desktop/AI_in_Action/LAB_10/K4-L3B-Day10-Soul-DataPipelineDataObservability/data/clean/papers_clean.json), [data/results/corruption_log.json](file:///c:/Users/Phung%20Quoc%20Viet/Desktop/AI_in_Action/LAB_10/K4-L3B-Day10-Soul-DataPipelineDataObservability/data/results/corruption_log.json).

---

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** Khi thiết kế cơ chế phục hồi dữ liệu (Repair Mechanism) cho Checkpoint 5, cần lựa chọn cách khôi phục bộ dữ liệu sau khi đã bị tiêm 6 kịch bản lỗi.
- **Các phương án đã cân nhắc:**
  - *Phương án 1 (In-place Patching):* Viết hàm dò tìm các bản ghi bị lỗi (ví dụ tìm summary rỗng rồi fill text mặc định, tìm title ngắn rồi ghép thêm chuỗi, lọc bỏ duplicate rows trực tiếp trên file `papers_clean_corrupted.json`).
  - *Phương án 2 (Raw Snapshot Preservation & Idempotent Re-execution):* Giữ nguyên bản snapshot thô ban đầu `data/raw/crossref_records.json` như một **Single Source of Truth** bất biến. Khi có sự cố, pipeline kích hoạt quy trình tái thực thi (Re-execution) từ raw records chạy qua toàn bộ logic làm sạch chuẩn hóa để tạo ra dataset repaired.
- **Phương án đã chọn:** **Phương án 2 (Raw Snapshot Preservation & Idempotent Re-execution)**.
- **Lý do:**
  - *Tính đúng đắn và bất biến (Correctness & Idempotency):* Phương án "vá lỗi tại chỗ" (In-place patching) rất dễ dẫn đến hiện tượng tích lũy sai số (error accumulation), không thể khôi phục lại các bài báo đã bị xóa mất (`drop_latest_records`) hoặc nội dung summary đã bị xóa trắng. 
  - *Reproducibility:* Nguyên tắc cốt lõi của Data Engineering hiện đại là: *Dữ liệu thô phải bất biến, dữ liệu sạch là kết quả của hàm chuyển đổi thuần khiết: $\text{Clean} = f(\text{Raw})$*. Nếu $f$ là hàm xác định, ta luôn có thể tái lập trạng thái sạch 100% tại bất kỳ thời điểm nào mà không phụ thuộc vào trạng thái lỗi trước đó.
- **Bằng chứng quyết định phù hợp:** Dữ liệu sau phục hồi [data/clean/papers_clean_repaired.json](file:///c:/Users/Phung%20Quoc%20Viet/Desktop/AI_in_Action/LAB_10/K4-L3B-Day10-Soul-DataPipelineDataObservability/data/clean/papers_clean_repaired.json) khôi phục chính xác 24 bài báo, giúp RAG Agent phục hồi toàn bộ từ 60.0% lên 100.0% Hit Rate và 1.0000 Token F1.

---

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng/lỗi nguyên văn:**
  ```text
  Traceback (most recent call last):
    File "<string>", line 1, in <module>
    File "C:\Users\Phung Quoc Viet\AppData\Local\Programs\Python\Python312\Lib\encodings\cp1252.py", line 19, in encode
      return codecs.charmap_encode(input,self.errors,encoding_table)[0]
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  UnicodeEncodeError: 'charmap' codec can't encode character '\u1ed7' in position 21: character maps to <undefined>
  ```
- **Lệnh hoặc bước tái hiện:** Khi chạy lệnh kiểm chứng hàm tiêm lỗi `corrupt_dataset` trên Windows PowerShell:
  `.venv\Scripts\python.exe -c "... print('Tiêm thành công các lỗi:', list(log.keys()))"`
- **Nguyên nhân gốc:** Môi trường PowerShell mặc định trên Windows sử dụng bảng mã ký tự ANSI (`cp1252` hoặc `cp437`) cho luồng stdout/stderr tiêu chuẩn. Khi hàm `print()` in chuỗi tiếng Việt có dấu (`Tiêm thành công các lỗi`), bộ mã hóa mặc định của Python không thể ánh xạ ký tự Unicode tiếng Việt (`\u1ed7` - chữ "ỗ") dẫn đến văng lỗi `UnicodeEncodeError`.
- **Cách xử lý:** 
  1. Cấu hình biến môi trường toàn cục cho session PowerShell trước khi gọi Python:
     ```powershell
     $env:PYTHONIOENCODING="utf-8"
     ```
  2. Trong các hàm đọc/ghi file (`write_json`, `write_text` tại `src/core/utils.py`), luôn chỉ định tường minh tham số `encoding="utf-8"`.
  3. Trong các đoạn log/print kiểm chứng, chuẩn hóa sang chuỗi không dấu hoặc thông điệp chuẩn ASCII để đảm bảo an toàn tuyệt đối trên mọi nền tảng OS.
- **Cách xác minh sau khi sửa:** Chạy lại lệnh kiểm thử có kèm biến môi trường, lệnh thực thi trơn tru với exit code 0 và in kết quả đầy đủ ra console.
- **Điều học được:** Khi phát triển Data Pipeline trên môi trường đa nền tảng (đặc biệt là Windows), luôn luôn phải kiểm soát chặt chẽ Encoding ở cả 3 tầng: File I/O, Python Process Environment và Terminal Console.

---

## 7. Hiểu biết về luồng end-to-end

**1. Dữ liệu đi từ Crossref đến vector index như thế nào?**
- Bắt đầu từ Crossref REST API: `crossref.py` gửi truy vấn lấy metadata của 24 bài báo khoa học thuộc lĩnh vực AI/Data. Nếu có sự cố kết nối, cơ chế fallback lập tức đọc bản snapshot local có sẵn. Dữ liệu thô được lưu vào `data/raw/crossref_records.json`.
- Tầng Cleaning: `cleaning.py` đọc 24 bài báo thô, loại bỏ triệt để các thẻ HTML/JATS XML rác, tính toán số ngày tuổi `age_days`, và format trường `text_for_embedding` gồm 5 phần chuẩn. Dữ liệu sạch được lưu vào `data/clean/papers_clean.json`.
- Tầng Vector Store: `index.py` nhận file JSON sạch, nạp vào mô hình embedding `sentence-transformers/all-MiniLM-L6-v2` để tính toán các vector biểu diễn 384 chiều, sau đó lưu kèm metadata vào ChromaDB collection `papers-baseline`.

**2. Evaluation set và ground-truth document IDs dùng để đo retrieval/answer quality ra sao?**
- Bộ `test_set.json` gồm 10 câu hỏi chuẩn hóa thuộc 4 nhóm nghiệp vụ (`summary`, `authors`, `date`, `categories`). Mỗi câu hỏi đều đi kèm danh sách `ground_truth_doc_ids` (chứa chính xác `paper_id` của bài báo mang thông tin câu trả lời).
- Khi đánh giá Retrieval Hit Rate: Hệ thống lấy câu hỏi, truy vấn ChromaDB lấy top-k bài báo tương đồng nhất. Nếu top-k trả về có chứa ít nhất một ID nằm trong `ground_truth_doc_ids`, truy vấn được tính là thành công (Hit).
- Khi đánh giá Answer Quality: Câu trả lời được Agent sinh ra từ context truy xuất sẽ được so sánh với câu trả lời ground-truth để tính điểm trùng khớp từ ngữ (`mean_token_f1`) và chấm điểm ngữ nghĩa (`mean_judge_score`).

**3. Quality checks khác freshness monitoring ở điểm nào trong bài lab?**
- **Quality Checks (Great Expectations 1.x):** Đóng vai trò là chốt kiểm dịch cấu trúc dữ liệu (Data Structural & Schema Integrity). Nó kiểm tra xem tập dữ liệu có đủ số dòng (20 đến 30 dòng), các trường quan trọng có bị rỗng (null/blank) hay không, `paper_id` có bị trùng lặp (uniqueness) hay không, và độ dài tiêu đề có đạt chuẩn ($\ge 8$ ký tự) hay không.
- **Freshness Monitoring:** Đóng vai trò là chốt giám sát thời gian và SLA nghiệp vụ (Data Timeliness). Nó kiểm tra độ tươi mới của dữ liệu thông qua cột `age_days`. Nếu tỷ lệ bài báo quá 180 ngày vượt quá 25%, hệ thống sẽ đánh cờ vi phạm `is_fresh = False` để cảnh báo dữ liệu đã bị lỗi thời.

**4. Vì sao phải dùng cùng test set cho baseline, corrupted và repaired?**
- Để đảm bảo tính khách quan và khoa học theo nguyên tắc **Thí nghiệm đối chứng (Controlled Experiment)**. Trong thí nghiệm này, chất lượng dữ liệu là biến độc lập duy nhất thay đổi qua 3 pha (Sạch ➔ Bị tiêm lỗi ➔ Phục hồi). Việc giữ cố định 10 câu hỏi kiểm thử và ground-truth đảm bảo rằng sự sụt giảm hay phục hồi của Hit Rate và F1 phản ánh chính xác 100% chất lượng của dữ liệu, loại trừ hoàn toàn yếu tố chủ quan do câu hỏi thay đổi độ khó.

**5. Repair được xem là thành công dựa trên artifact và metric nào?**
- Quá trình Repair được xem là thành công hoàn toàn khi đáp ứng đồng thời cả tiêu chuẩn kiểm dịch và phục hồi chỉ số AI:
  1. **Data Quality Gate:** Great Expectations chuyển từ `FAILED (False)` sang `PASSED (True)` tại `repaired_quality_report.json` (đạt 8/8 expectations).
  2. **Freshness SLA:** Chuyển từ `STALE (False)` sang `FRESH (True)` tại `repaired_freshness_report.json` (chỉ 4.17% bài quá 180 ngày, dưới ngưỡng 25%).
  3. **Retrieval Hit Rate:** Phục hồi từ 60.0% lên **100.0%**.
  4. **Mean Token F1:** Phục hồi từ 0.5000 lên **1.0000**.
  5. **Judge Accuracy & Score:** Phục hồi từ 50.0% lên 100.0% và điểm số đạt tuyệt đối 5.00/5.0.
  6. **Artifact kiểm chứng:** Báo cáo đối chiếu [data/reports/corruption_report.md](file:///c:/Users/Phung%20Quoc%20Viet/Desktop/AI_in_Action/LAB_10/K4-L3B-Day10-Soul-DataPipelineDataObservability/data/reports/corruption_report.md) hiển thị đầy đủ bảng so sánh 3 cột.

---

## 8. Phân tích kết quả

### Metrics chính

| Metric/signal | Baseline | Corrupted | Repaired | Nhận xét của cá nhân |
| :--- | :---: | :---: | :---: | :--- |
| `retrieval_hit_rate` | **100.0%** | **60.0%** | **100.0%** | Giảm 40% do 4 bài mới bị xóa và các bài bị tiêm noise; phục hồi hoàn hảo sau repair |
| `mean_token_f1` | **1.0000** | **0.5000** | **1.0000** | Sụt giảm nghiêm trọng khi câu trả lời thiếu ngữ cảnh hoặc dính chuỗi rác; hồi sinh về 1.0 |
| `judge_accuracy` | **100.0%** | **50.0%** | **100.0%** | Năng lực trả lời đúng của RAG Agent bị suy giảm một nửa khi dữ liệu bị lỗi |
| `mean_judge_score` | **5.00/5.0** | **3.00/5.0** | **5.00/5.0** | Điểm số chất lượng ngữ nghĩa giảm từ Xuất sắc xuống Trung bình |
| Quality checks (GX 1.x) | **PASSED (True)** | **FAILED (False)** | **PASSED (True)** | Bắt chính xác 3 vi phạm: null summary, tiêu đề ngắn và trùng lặp ID |
| Freshness status (>180d) | **FRESH (True)** | **STALE (False)** | **FRESH (True)** | Bắt đúng kịch bản sửa lùi ngày về 2016 (30.43% quá hạn, vượt ngưỡng 25%) |

### Kết luận từ số liệu

**Hai chuỗi nguyên nhân – bằng chứng:**
1. *Chuỗi suy thoái (Corruption Impact):*
   Khi tôi tiêm 6 kịch bản lỗi vào dữ liệu sạch ➔ `corrupted_quality_report.json` ghi nhận 3 expectations thất bại và `corrupted_freshness_report.json` ghi nhận tỷ lệ quá hạn 30.43% ➔ Tuy nhiên hệ thống RAG không hề bị crash (hiện tượng **Silent Failure**) mà vẫn tiếp tục tạo embedding và truy vấn ➔ Dẫn đến hệ quả `retrieval_hit_rate` sụt giảm mạnh từ 100% xuống 60%, `mean_token_f1` giảm từ 1.0 xuống 0.5.
2. *Chuỗi phục hồi (Repair Impact):*
   Khi kích hoạt cơ chế Idempotent Repair đọc lại từ snapshot bất biến `crossref_records.json` ➔ Dữ liệu được làm sạch lại hoàn toàn, tạo ra `papers_clean_repaired.json` ➔ Quality Gate và Freshness SLA lập tức chuyển sang màu xanh (PASSED/FRESH) ➔ Toàn bộ chỉ số hiệu năng của AI quay trở lại mốc hoàn hảo 100% Hit Rate và 1.0 Token F1.

**Corruption nào ảnh hưởng rõ nhất và vì sao?**
- Hai kịch bản gây tác hại lớn nhất là: **`drop_latest_records` (Xóa bài mới nhất)** và **`blank_summary` (Xóa rỗng tóm tắt)**:
  - Khi xóa bài báo mới nhất, tài liệu chứa ground-truth biến mất hoàn toàn khỏi cơ sở dữ liệu vector. Dù câu hỏi rất đơn giản và thuật toán tìm kiếm rất mạnh, Agent vẫn không thể tìm thấy (Hit Rate = 0 cho các câu hỏi tương ứng).
  - Khi xóa rỗng summary, trường `text_for_embedding` mất đi hơn 80% dung lượng ngữ nghĩa, vector embedding sinh ra bị co cụm hoặc sai lệch khoảng cách cosine, dẫn đến việc bộ tìm kiếm trả về các tài liệu hoàn toàn không liên quan.

**Kết quả nào khác với kỳ vọng ban đầu?**
- Ban đầu tôi nghĩ rằng khi tiêm chuỗi ký tự rác (`inject_noise`) hoặc cắt ngắn tiêu đề (`truncate_title`), mô hình sentence-transformers hoặc ChromaDB có thể sẽ ném ra lỗi ngoại lệ (exception) do định dạng không hợp lệ. Nhưng thực tế mô hình AI vẫn "nuốt chửng" dữ liệu rác và âm thầm sinh ra câu trả lời sai lệch. Điều này cho thấy tầm quan trọng sống còn của Data Observability: nếu không có chốt kiểm dịch Great Expectations chặn từ tầng dữ liệu, người dùng cuối sẽ phải gánh chịu toàn bộ sự ảo giác (hallucination) của AI mà đội ngũ kỹ thuật không hề hay biết.

---

## 9. Điều học được và hướng cải thiện

### Ba điều quan trọng nhất
1. **Nguyên tắc bất biến của dữ liệu thô (Raw Data Immutability):** Dữ liệu thô thu thập từ API phải được bảo toàn nguyên vẹn trong một snapshot bất biến. Đây chính là chiếc "phao cứu sinh" duy nhất cho phép hệ thống tự phục hồi Idempotent khi xảy ra sự cố dữ liệu.
2. **Nguy cơ tiềm ẩn của Silent Failure trong AI:** Mô hình ngôn ngữ lớn (LLM) và Vector Database không có khả năng tự phát hiện dữ liệu bẩn. Chúng luôn cố gắng trả lời ngay cả khi dữ liệu đầu vào là rác. Do đó, việc xây dựng chốt kiểm dịch (Data Quality Gate) ở tầng Ingestion là bắt buộc đối với mọi hệ thống RAG thực tế.
3. **Data Contracts và Pair Programming trong đội ngũ kỹ thuật:** Việc phân chia ranh giới rõ ràng về Input/Output và Data Contracts giữa các thành viên (TV1 lo dữ liệu, TV2 lo kiểm dịch, TV3 lo AI) giúp cả nhóm làm việc độc lập song song mà không bị xung đột, đồng thời dễ dàng tích hợp ở khâu cuối cùng.

### Nếu có thêm thời gian
- **Cải thiện cụ thể:** Xây dựng cơ chế **Dead Letter Queue (DLQ)** tự động kết hợp với **Data Quarantine Buffer**.
- **Lý do:** Hiện tại, khi phát hiện dữ liệu bẩn, toàn bộ pipeline phải dừng lại hoặc cần kích hoạt sửa chữa toàn bộ dataset. Với DLQ, các bản ghi lỗi sẽ được tự động tách riêng vào vùng cách ly để chuyên viên kiểm tra, trong khi 80% các bản ghi sạch còn lại vẫn được chuyển tiếp mượt mà vào Vector Store để phục vụ người dùng.
- **Cách đo lường cải thiện:** Đo lường tỷ lệ dữ liệu khả dụng liên tục (**Data Pipeline Availability**) đạt $\ge 99.5\%$ và thời gian tự động xử lý phục hồi bản ghi lỗi (**MTTR**) dưới 30 giây.

---

## 10. Cam kết của thành viên

Đánh dấu sau khi tự kiểm tra:

- [x] Nội dung báo cáo phản ánh đúng phần việc và mức hiểu của tôi.
- [x] Tôi có thể giải thích luồng end-to-end, không chỉ module mình phụ trách.
- [x] Mọi kết luận về kết quả đều có artifact hoặc metric để đối chiếu.
- [x] Tôi không ghi “đã chạy thành công” cho phần chưa được kiểm chứng.
- [x] Báo cáo không chứa `.env`, API key, token hoặc secret.
- [x] Báo cáo này không phải bản sao nguyên văn của báo cáo nhóm hoặc báo cáo thành viên khác.

**Họ và tên:** Phùng Quốc Việt  
**Ngày xác nhận:** 2026-09-26  

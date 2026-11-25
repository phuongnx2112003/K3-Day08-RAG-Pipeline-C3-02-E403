# Báo cáo code review — Nhóm trưởng

**Thời điểm review:** 2026-08-04  
**Phiên bản đã review:** `main` / `3681900` (`merge: resolve merge conflict and integrate full RAG A/B benchmark app`)  
**Phạm vi:** RAG pipeline, hai giao diện (Streamlit/React-FastAPI), đánh giá A/B, khả năng chạy lại và quản trị repository.

## Kết luận điều hành

Pipeline có nền tảng tốt: tách được dense retrieval, BM25, RRF và PageIndex fallback; BM25/RRF có kiểm tra đầu vào tương đối kỹ. Tuy nhiên, **chưa nên demo/phát hành ở trạng thái hiện tại**. Có một lỗi giao diện Streamlit chắn toàn bộ luồng chat, kết quả đánh giá A/B không phải số đo thực nghiệm, và repository đang chứa dependency/artifact sinh tự động. Đề nghị ưu tiên khắc phục P0/P1 trước khi freeze bản demo.

| Mức độ | Số lượng | Quy tắc xử lý |
|---|---:|---|
| P0 — chặn demo | 1 | Sửa và kiểm thử lại ngay |
| P1 — sai lệch kết quả / rủi ro cao | 3 | Hoàn thành trước freeze |
| P2 — chất lượng, bảo mật, vận hành | 4 | Lên kế hoạch trong sprint kế tiếp |

## Phát hiện chi tiết

### P0-01 — Streamlit gọi sai hợp đồng hàm generation

**Bằng chứng:** [app.py](../app.py) dòng 139–149 gọi `generate_with_citation` với `conversation` và `score_threshold`, rồi kỳ vọng `model`/`citations_valid`. Trong khi [src/task10_generation.py](../src/task10_generation.py) dòng 162–168 chỉ nhận `query`, `top_k`, `retrieval_mode`, `use_reranking`, `use_pageindex_fallback`, và trả ba trường ở dòng 234–238.

**Tác động:** mọi query từ Streamlit phát sinh `TypeError`; UI bắt exception và chỉ hiện thông báo lỗi. Thanh điều chỉnh fallback và trạng thái citation/model cũng không có tác dụng.

**Khuyến nghị:** thống nhất một response schema và chữ ký hàm. Hoặc bổ sung hai tham số (và logic) cho Task 10, hoặc sửa `app.py` chỉ truyền các tham số hiện hỗ trợ. Thêm integration test gọi chính xác luồng Streamlit → Task 10 với mock retrieval/LLM.

### P1-01 — Báo cáo metric RAG/A-B là heuristic và Config B bị suy diễn

**Bằng chứng:** [group_project/evaluation/eval_pipeline.py](evaluation/eval_pipeline.py) dòng 72–80 gán Faithfulness/Relevancy/Recall theo điều kiện chuỗi/số nguồn; dòng 120 chỉ chạy 5 mẫu cho Config A; dòng 122–129 tạo Config B bằng cách trừ cố định điểm của A. Dù tiêu đề tài liệu ghi RAGAS, mã không gọi RAGAS.

**Tác động:** bảng kết quả và kết luận “Hybrid vượt trội” không chứng minh được bằng thực nghiệm; có thể gây hiểu lầm trong demo/báo cáo học thuật.

**Khuyến nghị:** chạy cả hai cấu hình thật trên cùng toàn bộ golden dataset; lưu answer, contexts, latency và source IDs theo từng case. Nếu không có judge/LLM cho RAGAS, đổi tên metric thành heuristic và không ghi là RAGAS. Tính recall/precision bằng `expected_sources` thay vì chuỗi `expected_context`.

### P1-02 — Fallback PageIndex không tuân theo mô tả và dễ trả evidence yếu

**Bằng chứng:** docstring [src/task9_retrieval_pipeline.py](../src/task9_retrieval_pipeline.py) dòng 74–78 nói PageIndex không có evidence thì trả `[]`, nhưng dòng 119 để lại kết quả dense yếu. Điều này trái với ý nghĩa threshold; Task 10 sau đó còn tự fallback sang keyword-local ở [src/task10_generation.py](../src/task10_generation.py) dòng 190–193.

**Tác động:** câu hỏi ngoài phạm vi có thể vẫn nhận context/câu trả lời thay vì từ chối; benchmark fallback thiếu tính tin cậy.

**Khuyến nghị:** quyết định rõ chính sách: khi dense dưới ngưỡng và PageIndex không có kết quả thì trả `[]` (khuyến nghị), không chuyển sang local retrieval; hoặc chỉ local fallback khi lỗi hạ tầng và phải gắn cờ `degraded`. Viết test out-of-domain cho cả hai nhánh.

### P1-03 — Lỗi retrieval bị nuốt, làm mất khả năng chẩn đoán

**Bằng chứng:** [src/task10_generation.py](../src/task10_generation.py) dòng 178–188 bắt toàn bộ exception từ `retrieve` rồi coi là không có chunk; tiếp tục local fallback ở dòng 190–193.

**Tác động:** thiếu JINA key, lỗi Chroma, lỗi schema hoặc lỗi mã đều bị che thành kết quả local, làm API/UI báo sai `retrieval_source` và khó điều tra.

**Khuyến nghị:** bắt các lỗi dự kiến riêng biệt, log stack trace có cấu trúc, và trả trạng thái `error`/`degraded` trong response. Không được dùng local fallback để che lỗi pipeline chính.

### P2-01 — Repository commit 6.854 file `node_modules` và Chroma DB

**Bằng chứng:** `git ls-files 'frontend/node_modules/**'` trả 6.854 tệp; `chroma_db` cũng đang tracked, dù [.gitignore](../.gitignore) đã ignore hai đường dẫn. Kiểm tra build thực tế thất bại: `npm run build` → `sh: 1: vite: Permission denied`; file `frontend/node_modules/.bin/vite` không có executable bit.

**Tác động:** clone/pull nặng, diff nhiễu, không tái lập đa nền tảng và làm hỏng build trên môi trường hiện tại.

**Khuyến nghị:** git-rm cache `frontend/node_modules/` và `chroma_db/` trong một PR dọn dẹp riêng (sau khi team xác nhận artifact không cần lịch sử); giữ `package-lock.json`, thêm hướng dẫn `npm ci`, tạo script index dữ liệu thay cho commit DB. CI phải chạy `npm ci && npm run build`.

### P2-02 — API mở CORS toàn bộ trong khi bật credentials

**Bằng chứng:** [src/server.py](../src/server.py) dòng 39–45 đặt `allow_origins=["*"]` cùng `allow_credentials=True`.

**Tác động:** cấu hình không phù hợp cho production và có thể bị browser/framework xử lý không như mong muốn khi dùng credential.

**Khuyến nghị:** production dùng allowlist origin từ biến môi trường; nếu public không dùng credential thì tắt `allow_credentials`.

### P2-03 — Thiếu kiểm thử end-to-end và môi trường test chưa sẵn sàng

**Bằng chứng:** `python3 -m compileall -q app.py src group_project/evaluation tests` thành công (chỉ có cảnh báo escape sequence trong docstring `app.py`), nhưng `pytest -q` thất bại vì `pytest: command not found`. Test hiện có là unit-oriented, chưa có test contract Streamlit/Task 10 hoặc FastAPI endpoints.

**Tác động:** lỗi P0 đã không được phát hiện trước merge.

**Khuyến nghị:** tạo môi trường từ `requirements.txt`, thêm lockfile/CI, và bắt buộc chạy: `pytest -q`, smoke test `/api/health`, `/api/chat`, `/api/compare`, `npm ci && npm run build` trước merge.

### P2-04 — Chủ đề và tài liệu còn không nhất quán

**Bằng chứng:** [README.md](../README.md) mô tả “University Services/RMIT”; [app.py](../app.py), [src/server.py](../src/server.py) và dữ liệu lại là Book Insights/BookMind; [src/supervisor.py](../src/supervisor.py) vẫn ghi University Services. README hướng dẫn Streamlit trong khi `src/server.py` phục vụ FastAPI/static UI.

**Tác động:** người demo/cài đặt dễ chạy nhầm giao diện, corpus và câu hỏi; giảm độ tin cậy của bàn giao.

**Khuyến nghị:** chọn một product name/corpus chính thức, cập nhật README, `.env.example`, runbook và docstrings đồng bộ; nêu rõ giao diện nào là bản demo được hỗ trợ.

## Điểm tích cực

- `text_normalization.py` xử lý Unicode, zero-width và dehyphenation hợp lý cho tiếng Việt/Anh.
- BM25/TF-IDF và RRF có validation, thứ tự ổn định và loại trùng chunk rõ ràng.
- Task 9 dùng cosine dense gốc, không dùng RRF score để quyết định fallback—đúng về mặt thiết kế.
- Có golden dataset, benchmark API và tài liệu demo, là nền tảng tốt để chuyển sang đánh giá thực nghiệm.

## Kế hoạch hành động đề xuất

1. **Owner UI + Task 10 (P0):** thống nhất contract generation, sửa Streamlit, thêm integration test và xác nhận một query thành công.
2. **Owner Retrieval (P1):** chốt chính sách fallback/error; không che lỗi pipeline bằng local fallback; thêm 3 test: out-of-domain, thiếu Jina key, PageIndex không có doc ID.
3. **Owner Evaluation (P1):** thay evaluation mô phỏng bằng chạy A/B thực tế trên toàn bộ dataset, xuất raw results và mô tả trung thực phương pháp đo.
4. **Owner DevOps/repo (P2):** bỏ tracked artifacts, tái cài dependency sạch, bổ sung CI và README/runbook thống nhất.

## Tiêu chí cho bản demo đạt (exit criteria)

- Streamlit hoặc FastAPI/React được chọn làm UI chính và một câu hỏi mẫu trả answer + nguồn không lỗi.
- Hai config A/B chạy thật trên cùng dataset; kết quả truy vết được theo từng câu hỏi.
- Câu hỏi Bitcoin/ngoài corpus bị từ chối nhất quán khi không có evidence.
- `pytest -q` và `npm ci && npm run build` chạy xanh trên môi trường sạch.
- `git status` sạch ngoài dữ liệu local được chủ động giữ; không còn `node_modules`/database sinh tự động trong Git.

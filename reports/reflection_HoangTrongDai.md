# Reflection Cá Nhân — Mapping Bài Giảng & Kế Hoạch Đồ Án

**Học viên:** Hoàng Trọng Đại · **Khóa học:** AICB-K34 · Track 3: GraphRAG · **Ngày:** 20/08/2026

> File này tách riêng theo đúng tên gọi trong `RUBRIC.md` (mục 4.3).

---

## 1. Mapping Bài giảng vào Code

| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|--------------------------|------------------|------------------------|-----------------------------|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()`, `run_coref()` | Cơ chế fallback đúng thiết kế (không suy diễn khi lỗi) nhưng 97% batch bị lỗi API trong lần chạy thực tế → coref gần như không có tác dụng, cần retry/backoff tốt hơn thay vì catch-all exception. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` trong `extract_batch()` | Chặn hiệu quả quan hệ/loại node ngoài schema; không chặn được lỗi *gán sai type trong phạm vi allowlist* (case Aaritya Technologies → Technology thay vì Company). |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` (UNWIND + MERGE, batch 1000) | Chạy ổn định qua nhiều lần re-run nhờ MERGE idempotent — không tạo trùng lặp dù pipeline phải chạy lại nhiều lần để khắc phục lỗi model/quota. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF`, `merge_guard()` | Ngưỡng mặc định 0.90 quá cao cho tập dữ liệu nhỏ/đa dạng (audit rỗng); điều chỉnh xuống 0.40 để bộc lộ rõ cả case merge đúng lẫn case rủi ro false-merge thật. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()`, `SUPER_NODE_DEGREE=100` | Chưa thực sự kích hoạt trong đồ thị 103-node hiện tại (max degree=5); cơ chế chạy không lỗi qua `test_supernode_policy()` nhưng cần đồ thị lớn hơn để quan sát tác động thật. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()`, `judge_json()` | Judge (OpenAI gpt-4o-mini) cho điểm nhất quán, rationale rõ ràng, trung thực khi cả 2 hệ thống thiếu evidence (không cho điểm cao giả tạo). |

---

## 2. Quá trình Debugging & Bài học

**Lỗi kỹ thuật phức tạp nhất gặp phải:** Model `llama-3.3-70b-versatile` (mặc định trong `.env.example`) đã bị Groq **deprecate hoàn toàn** (`404 model_not_found`). Lỗi này không lộ ra ngay mà bị "nuốt âm thầm": coreference "chạy xong" không báo lỗi (nhờ cơ chế fallback conservative), nhưng bước NER+RE Extraction trả về `raw_triples_df` rỗng 0 cột, gây `AttributeError: 'DataFrame' object has no attribute 'source_raw'` ở bước Entity Resolution — cách xa vị trí lỗi gốc nhiều cell, rất khó truy vết nếu không kiểm tra kỹ log gốc.

**Cách xử lý:** Truy vết ngược từ traceback lên tận cell đầu tiên gọi Groq API, viết script test độc lập gọi thử model để xác nhận lỗi 404, liệt kê danh sách model khả dụng qua `client.models.list()`, thay bằng model còn hoạt động. Sau đó tiếp tục gặp **rate-limit Tokens-Per-Day (200.000/ngày)** dùng chung hạn mức cho model mới — giải quyết bằng cách xây cơ chế **checkpoint/resume** (lưu kết quả coref/extraction/entity-resolution ra CSV, tự động load lại nếu đã tồn tại) để không phải trả token 2 lần cho cùng một bước mỗi khi phải đổi model.

**Bài học lớn nhất:** Trong pipeline nhiều bước phụ thuộc LLM, lỗi ở bước đầu (extraction rỗng) có thể **không throw exception ngay** mà chỉ biểu hiện ở bước xa hơn dưới dạng lỗi khó hiểu (AttributeError trên DataFrame rỗng) — cần validate output rỗng ngay tại nguồn (đã thêm `if raw_triples_df.empty: raise RuntimeError(...)` để lỗi lộ ra đúng chỗ) thay vì để nó lan truyền.

---

## 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)

**Tên đồ án:** P-168 — AI Agent phát hiện tin giả và link độc (URL/domain risk scoring: rule engine + blacklist + reputation intelligence + LLM claim verification).

**Đặc thù bài toán & lý do chọn giải pháp:** Có cần GraphRAG. Các URL/domain độc hại thường liên kết nhau qua hạ tầng dùng chung (cùng WHOIS registrant, cùng IP/hosting, cùng brand bị giả mạo, cùng chiến dịch lừa đảo) — bài toán multi-hop/cross-doc điển hình ("domain này liên hệ gì với 5 domain đã bị blacklist trước đó?") mà Flat vector search trên corpus fact-check hiện tại không trả lời được. GraphRAG chỉ nên bổ sung cho `retrieve_evidence`/`verify_claim` ở case rủi ro trung bình-cao cần giải thích, không thay thế rule engine tốc độ cao ở đầu pipeline.

**Cấu trúc Node & Relation dự kiến:**
- Nodes: `Domain`, `URL`, `Registrant` (WHOIS), `IP/Hosting`, `Brand`, `Campaign`, `BlacklistEntry`
- Relations: `REGISTERED_BY`, `RESOLVES_TO`, `IMPERSONATES` (brand), `SHARES_INFRA_WITH`, `PART_OF_CAMPAIGN`, `FLAGGED_BY` (nguồn: VirusTotal/Safe Browsing/security-admin)

**Chiến lược Super-node & Entity Resolution:** Domain/hosting dùng chung (Cloudflare, URL shortener) sẽ thành super-node cực lớn — bắt buộc áp `SUPER_NODE_DEGREE` cap như lab này để tránh 1 IP shared-hosting kéo hàng nghìn domain không liên quan vào context. Entity Resolution cho `Registrant`/`Brand` nên dùng ngưỡng **cao** (gần 0.90, khác với 0.40 của lab) vì đây là domain an ninh — gộp nhầm 2 registrant khác nhau (False Merge) nguy hiểm hơn nhiều so với bỏ sót liên kết.

---

## 🎯 Tự đánh giá
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | 3 | - |
| Khả năng kiểm soát AI Coding Agent | 3 | - |
| Chất lượng đồ thị tri thức xây dựng | 3 | - |
| Khả năng phân tích và debug hệ thống | 3 | - |

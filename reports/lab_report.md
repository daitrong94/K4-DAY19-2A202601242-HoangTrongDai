# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Hoàng Trọng Đại 
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 20/08/2026

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
> **Tình huống thực tế:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Ví dụ từ dữ liệu:** Trên 400 chunk đưa vào bước Coreference (`EXTRACTION_MAX_CHUNKS=400`), có **388/400 chunk (97%)** trả về `unresolved_mentions = ["COREF_BATCH_FAILED"]` — ví dụ cụ thể: `chunk_id=331537d8f978a369b442::c0000` ("Ryan Specialty... acquire Socius Insurance Services") và `chunk_id=55a09cbc43c41ffb2dd9::c0000` ("Talkiatry... partnership with Transact").
- **Hiện tượng:** Đây **không phải** là lỗi phân giải sai đại từ theo nghĩa ngữ nghĩa (nhầm antecedent), mà là lỗi ở tầng hạ tầng: hàm `resolve_coref_batch()` gọi Groq API theo batch 5 chunk/lần, và phần lớn các lệnh gọi bị exception (rate-limit/JSON không hợp lệ khi dùng các model thay thế `llama-3.3-70b-versatile` đã bị Groq deprecate — xem mục 5). Khi đó, khối `except Exception` trong `run_coref()` kích hoạt cơ chế fallback: giữ nguyên `text` gốc và gắn nhãn `COREF_BATCH_FAILED` thay vì suy diễn.
- **Hậu quả đối với Graph:** Về mặt an toàn dữ liệu, cơ chế "conservative fallback" hoạt động đúng tinh thần thiết kế — **không có False Edge nào được tạo ra do suy diễn sai đại từ**, vì hệ thống không cố "đoán" khi gặp lỗi. Tuy nhiên, hậu quả thực tế là **coreference gần như không có tác dụng** trong lần chạy này: các câu có đại từ tham chiếu xuyên câu (ví dụ "the company", "it") không được thay bằng tên thực thể đầy đủ trước khi đưa vào bước NER+RE, khiến một số quan hệ có thể bị bỏ sót (Recall thấp) thay vì bị sai (Precision vẫn giữ được). Đây là đánh đổi giữa "thà bỏ sót còn hơn tạo cạnh sai" — đúng chủ đích của "Conservative rule" trong đề bài, nhưng lộ ra một điểm yếu vận hành: cần retry/backoff và logging lỗi rõ ràng hơn thay vì catch-all exception im lặng.

---

### 2. Entity Resolution Threshold & Lexical Guard
> **Ngưỡng & Cơ chế Guard:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao nhưng bị Lexical Guard chặn không cho gộp và giải thích lý do.

*Trả lời:*
- **Ngưỡng cosine similarity đã chọn:** `threshold = 0.40` (giảm từ mặc định gợi ý 0.90 trong đề). **Lý do:** với `EXTRACTION_MAX_CHUNKS=400` trên một tập dữ liệu tin tức đa dạng (đa số công ty nhỏ/độc lập, không lặp lại), số lượng thực thể trùng lặp thật sự (ví dụ "Microsoft" vs "Microsoft Corp") gần như bằng 0 — ở ngưỡng 0.90 mặc định, bảng audit **hoàn toàn rỗng (0 dòng)**, không thỏa yêu cầu tối thiểu 10 dòng audit minh bạch. Hạ ngưỡng xuống 0.40 (kèm `top_k=6`) giúp hệ thống xét nhiều cặp ứng viên hơn và để **Lexical Guard** (không phải ngưỡng vector) đóng vai trò quyết định cuối cùng — đúng tinh thần "Vector là đề xuất, Lexical Guard là người gác cổng".
- **Cặp thực thể similarity cao nhưng bị Guard chặn:** `AI` vs `Artificial Intelligence` (Technology), **cosine similarity = 0.791**. Guard tính `SequenceMatcher(strip_suffix("ai"), strip_suffix("artificial intelligence")).ratio() < 0.72` nên bị `REJECT_GUARD`.
- **Lý do chặn (và vì sao đây là điểm cần lưu ý):** Về mặt ngữ nghĩa, "AI" và "Artificial Intelligence" **là cùng một khái niệm** — merge lẽ ra là đúng. Nhưng Lexical Guard hiện tại chỉ so khớp chuỗi ký tự (SequenceMatcher ratio) sau khi loại corporate suffix, nên **không nhận diện được quan hệ viết tắt/từ đầy đủ**. Đây là một **False Negative** của Guard (chặn nhầm một merge hợp lệ) — khác với kịch bản kinh điển trong đề (Guard chặn đúng một merge nguy hiểm như `Sam Altman` vs `Steve Altman`), nhưng cho thấy rõ giới hạn thực sự của thuật toán: Guard tối ưu để tránh **False Merge** (an toàn hơn) bằng cái giá là có thể bỏ sót một số **True Merge** hợp lệ dạng viết tắt.
- **Trường hợp ngược lại (rủi ro False Merge thật sự phát hiện được):** `Annealing quantum computing` vs `Gate model quantum computing` (Technology, cùng do D-Wave Quantum Inc. phát triển), similarity = 0.625, nhưng `SequenceMatcher` ratio = 0.764 ≥ 0.72 nên bị **`MERGE_VECTOR`** (được gộp!). Đây là **2 công nghệ lượng tử khác nhau về bản chất** (ủ nhiệt vs mô hình cổng) chỉ trùng cụm từ đuôi "quantum computing" — một ví dụ thật về rủi ro False Merge nguy hiểm y hệt kịch bản `Apple` vs `Apple Watch` mà đề bài mô tả.

---

### 3. Đồ thị & Super-node Mitigation
> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy N cạnh mới nhất mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

*Trả lời:*
- **Đồ thị cuối cùng:** 103 node, 62 cạnh (edge), **0 cạnh thiếu `source_chunk_id`/`published_date`** (kiểm tra `graph_checks()` pass).
- **Top 3 Super-nodes:**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|------|--------------|---------------------|----------------------|
| 1 | Railergy | Company | 5 |
| 1 | Apple | Company | 5 |
| 3 | Unacademy | Company | 3 |

  (Railergy và Apple đồng hạng 1. Cả hai đều **thấp hơn nhiều** ngưỡng `SUPER_NODE_DEGREE=100`, nên trong lần chạy thực tế chưa node nào thực sự kích hoạt cơ chế cắt tỉa 50-cạnh — `test_supernode_policy()` chạy qua nhưng không có `supernode_events` nào được ghi nhận, do quy mô đồ thị 400-chunk còn nhỏ.)

- **Ưu điểm & Rủi ro của Temporal Mitigation (ưu tiên 50 cạnh mới nhất khi degree > 100):**
  - *Ưu điểm:* Chặn được hiện tượng bùng nổ context/token khi một thực thể lớn (ví dụ một Big Tech thật) có hàng nghìn cạnh — giữ độ trễ và chi phí truy vấn ổn định, đồng thời ưu tiên thông tin **cập nhật nhất** — phù hợp với các câu hỏi dạng "tình hình hiện tại của X".
  - *Rủi ro:* Nếu câu hỏi liên quan đến **sự kiện lịch sử xa** (ví dụ "X mua lại công ty nào đầu tiên") hoặc cần **tổng hợp toàn bộ lịch sử quan hệ** của một super-node, việc cắt cứng theo `published_date DESC LIMIT 50` sẽ loại bỏ các cạnh cũ có giá trị — tạo ảo giác "context đầy đủ" trong khi thực chất bị thiên lệch thời gian (recency bias).

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge, trung bình theo nhóm câu hỏi — trích từ `outputs/graphrag_vs_flatrag_summary.csv`):

| Nhóm | Tiêu chí đánh giá | Flat RAG | GraphRAG | Δ | Nhận xét |
|------|-------------------|----------|----------|---|----------|
| cross-doc | Comprehensiveness (1–5) | 1.50 | **3.00** | +1.50 | GraphRAG cải thiện rõ rệt |
| cross-doc | Faithfulness (1–5) | 3.00 | 3.00 | 0 | Ngang nhau |
| cross-doc | Multi-hop reasoning (1–5) | 2.00 | **3.00** | +1.00 | GraphRAG cải thiện rõ rệt |
| cross-doc | Latency (s) | 8.99 | 6.30 | -2.68 | GraphRAG nhanh hơn ở sample này |
| cross-doc | Token usage | 2468 | 2530 | +62 | Xấp xỉ nhau |
| factoid | Comprehensiveness/Faithfulness/Multi-hop | 1.00 | 1.00 | 0 | Cả hai đều thấp — câu hỏi ngoài phạm vi dữ liệu (xem phân tích bên dưới) |
| multi-hop | Comprehensiveness/Faithfulness/Multi-hop | 3.00 | 3.00 | 0 | Ngang nhau |
| multi-hop | Latency (s) | 3.27 | 12.56 | +9.29 | GraphRAG chậm hơn đáng kể (graph traversal + 2 lượt generate) |

**Nhận xét tổng quát:** GraphRAG thể hiện lợi thế rõ nhất ở nhóm **cross-doc** (tổng hợp thông tin từ nhiều bài báo theo dòng thời gian) — đúng như kỳ vọng thiết kế. Ở nhóm multi-hop đơn giản (2 thực thể trong cùng 1 chunk), hai phương pháp cho kết quả tương đương nhưng GraphRAG tốn thêm độ trễ do phải traversal + generate 2 lần (Flat + Graph context).

#### Phân tích 2 Ca lỗi Điển hình:

1. **Ca lỗi Flat RAG thất bại một phần, GraphRAG thắng rõ (G05):**
   - *Question ID & Câu hỏi:* `G05` — "Based on multiple articles from 2023, how did Apple's technology activities evolve over the year?"
   - *Kết quả:* Flat RAG: Comprehensiveness=2, Multi-hop=3. GraphRAG: Comprehensiveness=5, Multi-hop=5.
   - *Tại sao Flat RAG yếu hơn?* Top-k=6 similarity search không đảm bảo lấy đủ **cả 3 chunk liên quan đến Apple trải dài từ tháng 5 đến tháng 9/2023** (Final Cut Pro/Logic Pro, M3/A17 Bionic, đối tác Arm) trong cùng một lần truy vấn — vector search tối ưu theo độ tương đồng ngữ nghĩa tức thời, không có khái niệm "cùng 1 entity nhưng khác thời điểm".
   - *GraphRAG đã giải quyết như thế nào?* Seed extraction nhận diện entity "Apple" → BFS traversal thu thập **toàn bộ cạnh liên quan đến node Apple** (degree=5) bất kể chunk nguồn, rồi `textualize()` sắp xếp theo `published_date DESC` — cho phép model tổng hợp đúng trình tự thời gian: phần mềm sáng tạo (05/2023) → chip M3/A17 (08/2023) → hợp tác Arm (09/2023). Judge rationale xác nhận: *"provides a detailed chronological account... including... advancements in silicon technology, and a partnership..."*.

2. **Ca lỗi cả hai cùng thất bại (G03 — Aaritya Technologies bị "vô hình"):**
   - *Question ID & Câu hỏi:* `G03` — So sánh 2 sự kiện gọi vốn tháng 9/2023: Meeno và Aaritya Technologies.
   - *Nguyên nhân:* Cả Flat RAG và GraphRAG đều trả lời đúng phần **Meeno** (Sequoia, $3.9M, chunk `212476770ac78703d3b7`) nhưng **đều không tìm thấy thông tin về Aaritya Technologies** dù dữ liệu thực sự tồn tại trong Neo4j (`Accel -INVESTED_IN-> Aaritya Technologies`, `Elevation Capital -INVESTED_IN-> Aaritya Technologies`, chunk `6ef23bfa65272a9d88a4`). Root cause: bước NER+RE đã **gắn sai `entity_type` cho "Aaritya Technologies" là `Technology` thay vì `Company`** (Aaritya Technologies thực chất là tên startup — sản phẩm mới của cựu CTO Swiggy, Dale Vaz). Vì seed matcher và graph traversal lọc theo type khi cần, và Flat RAG's top-k similarity cho câu hỏi so sánh 2 startup không đủ ưu tiên đúng chunk chứa "Aaritya" khi ở cạnh chunk "Meeno" mạnh hơn về mặt embedding — cả 2 hệ thống đều báo "insufficient evidence" cho phần này (Judge chấm thấp vì thiếu Comprehensiveness).
   - *Đề xuất khắc phục:* (1) Thêm bước validation/second-pass để phát hiện entity type nghi ngờ sai (ví dụ: entity có object là "chủ ngữ của INVESTED_IN" nhưng gắn type Technology thay vì Company/Person nên được flag review). (2) Với Flat RAG, tăng `k` hoặc dùng MMR (Maximal Marginal Relevance) để tránh 2 câu hỏi con bị "che" lẫn nhau trong cùng 1 lần retrieve. (3) Với GraphRAG, mở rộng seed extraction để nhận diện thực thể dù seed ban đầu gắn sai type (fallback fuzzy match không lọc type).

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
> Trade-offs, Agent Control & Scale 350MB

*Trả lời:*
- **Đánh đổi Quality vs Cost vs Latency:** Số liệu thực tế cho thấy GraphRAG **không phải lúc nào cũng đắt hơn về token** (ở nhóm factoid và multi-hop, GraphRAG dùng token tương đương hoặc ít hơn Flat RAG nhờ context đồ thị cô đọng hơn văn bản thô), nhưng **luôn tốn latency cao hơn** ở các trường hợp cần traversal nhiều hop (multi-hop: 12.56s vs 3.27s) vì phải: (1) gọi LLM trích seed entity, (2) truy vấn Neo4j nhiều round-trip theo BFS, (3) generate câu trả lời trên cả GRAPH + VECTOR context gộp. Đây là đánh đổi kinh điển: **GraphRAG trả nhiều latency hơn để đổi lấy Comprehensiveness/Multi-hop reasoning tốt hơn ở các câu hỏi cross-doc thực sự cần liên kết nhiều nguồn** — với câu hỏi đơn giản (factoid, multi-hop 1 chunk), chi phí traversal không mang lại lợi ích tương xứng.

- **Quyết định từ chối/điều chỉnh so với đề xuất mặc định của AI Coding Agent (Claude):** Trong quá trình debug, agent đã đề xuất và mình (người vận hành) đã **điều chỉnh thay vì áp dụng nguyên trạng** một số điểm:
  1. **Từ chối chạy toàn bộ notebook 300MB dataset lại từ đầu mỗi lần fix lỗi.** Vì mỗi vòng coreference+extraction trên 400 chunk tốn ngay ~199.000/200.000 token TPD của Groq (theo model), việc lặp lại toàn bộ pipeline mỗi lần sửa 1 dòng code là không khả thi trong thời gian giới hạn — thay vào đó đã bổ sung **cơ chế checkpoint/resume** (lưu kết quả coref, extraction, entity-resolution ra CSV, tự động load lại nếu đã tồn tại) để chỉ tính toán lại phần thực sự thay đổi.
  2. **Không giữ nguyên ngưỡng entity-resolution mặc định 0.90** dù đó là con số đề bài gợi ý — vì với quy mô 400-chunk sample, ngưỡng này khiến audit table rỗng hoàn toàn (vi phạm yêu cầu ≥10 dòng). Quyết định hạ threshold xuống 0.40 kèm giải thích rõ lý do (mục 2), thay vì fake dữ liệu audit.
  3. **Từ chối gán cứng danh sách MANUAL_ALIASES cho Microsoft/Google/Meta/Apple** khi phát hiện dataset mẫu (400 chunk từ ~1500 bài báo, seed=42) hầu như không nhắc đến các hãng này — thay vì sửa dữ liệu để khớp câu hỏi mẫu, đã **viết lại 4/5 câu hỏi Golden (G02–G05) dựa trên thực thể có thật trong đồ thị đã trích xuất** (Meeno/Sequoia, Aaritya Technologies, DoNotPay/Robot Attorney, Apple) để đảm bảo golden answers có căn cứ kiểm chứng được, thay vì để trống hoặc bịa đáp án.

- **Giải pháp scale lên 350MB (~100.000 bài báo):**
  - **Bottleneck đầu tiên:** LLM API rate-limit theo ngày (Tokens-Per-Day) — với chỉ 400 chunk đã chiếm gần hết 200.000 token/ngày/model của Groq free tier, mở rộng lên hàng chục nghìn chunk sẽ cần hàng triệu token/ngày, vượt xa hạn mức free tier của bất kỳ model đơn lẻ nào.
  - **Giải pháp:**
    1. **Async batch extraction với worker queue + nhiều API key/provider luân phiên** (round-robin qua nhiều model Groq/OpenAI có quota riêng, hoặc nâng cấp Dev Tier) thay vì gọi tuần tự.
    2. **Checkpoint/resume ở mức chunk** (không chỉ ở mức toàn cục) để có thể dừng/tiếp tục qua nhiều ngày mà không mất tiến độ — đã áp dụng một phần trong lab này.
    3. **Entity Resolution dùng ANN index (HNSW/FAISS-IVF) thay vì brute-force FlatIP** khi số lượng entity vượt quá vài chục nghìn, tránh so khớp toàn bộ O(N²).
    4. **Community Partitioning** (xem Bonus) để chia đồ thị lớn thành các cụm xử lý độc lập, giảm kích thước context khi truy vấn.
    5. Cân nhắc **downsize/quantize** sang model local (ví dụ Llama nhỏ chạy on-prem qua vLLM) cho các bước có khối lượng lớn nhưng độ khó thấp (dedup, coref cơ bản), chỉ dùng model lớn/API trả phí cho bước NER+RE cần độ chính xác cao.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code
| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|--------------------------|------------------|------------------------|-----------------------------|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()`, `run_coref()` | Cơ chế fallback hoạt động đúng thiết kế (không suy diễn khi lỗi) nhưng 97% batch bị lỗi API trong lần chạy thực tế → coref gần như không có tác dụng, cần cải thiện retry/backoff. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` trong `extract_batch()` | Hiệu quả trong việc chặn quan hệ/loại node ngoài schema; tuy nhiên không chặn được lỗi *gán sai type* trong phạm vi allowlist (case Aaritya Technologies bị gắn Technology thay vì Company). |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` (UNWIND + MERGE, batch 1000) | Chạy ổn định qua nhiều lần re-run nhờ `MERGE` idempotent — không tạo trùng lặp khi pipeline được chạy lại nhiều lần để khắc phục lỗi model/quota. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF`, `merge_guard()` | Ngưỡng mặc định 0.90 quá cao cho tập dữ liệu nhỏ/đa dạng; đã điều chỉnh xuống 0.40 để bộc lộ rõ cơ chế Guard (cả case merge đúng lẫn case rủi ro false-merge thật). |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()`, `SUPER_NODE_DEGREE=100` | Chưa thực sự kích hoạt trong đồ thị 103-node hiện tại (max degree=5); cơ chế đã kiểm chứng không lỗi qua `test_supernode_policy()` nhưng cần đồ thị lớn hơn để quan sát tác động thật. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()`, `judge_json()` | Judge (OpenAI gpt-4o-mini) cho điểm nhất quán, rationale rõ ràng và trung thực khi cả 2 hệ thống đều thiếu evidence (không cho điểm cao giả tạo). |

---

### 2. Quá trình Debugging & Bài học
- **Lỗi kỹ thuật phức tạp nhất gặp phải:** Model `llama-3.3-70b-versatile` (mặc định trong `.env.example`) đã bị Groq **deprecate hoàn toàn** (`404 model_not_found`) — lỗi này không lộ ra ngay mà bị **nuốt âm thầm**: coreference "chạy xong" không báo lỗi (nhờ cơ chế fallback), nhưng bước NER+RE Extraction trả về `raw_triples_df` rỗng 0 cột, gây `AttributeError: 'DataFrame' object has no attribute 'source_raw'` ở bước Entity Resolution — cách xa vị trí lỗi gốc nhiều cell, rất khó truy vết nếu không kiểm tra log gốc.
- **Cách xử lý:** Truy vết ngược từ traceback lên tận cell đầu tiên gọi Groq API, viết script test độc lập gọi thử model để xác nhận lỗi 404, liệt kê danh sách model khả dụng qua `client.models.list()`, rồi thay bằng model còn hoạt động (`openai/gpt-oss-120b`). Sau đó tiếp tục gặp **rate-limit Tokens-Per-Day (200.000/ngày)** dùng chung hạn mức cho mọi model mới thử — giải quyết bằng cách xây **cơ chế checkpoint/resume** để không phải trả token 2 lần cho cùng một bước khi phải đổi model nhiều lần.

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)
- **Tên đồ án / Dự án:** P-168 — AI Agent phát hiện tin giả và link độc (URL/domain risk scoring: rule + blacklist + reputation intel + LLM claim verification).
- **Đặc thù bài toán & Lý do chọn giải pháp:** Có cần GraphRAG. Các URL/domain độc hại thường **liên kết nhau qua hạ tầng dùng chung** (cùng WHOIS registrant, cùng IP/hosting, cùng brand bị giả mạo, cùng chiến dịch lừa đảo) — đây là bài toán multi-hop/cross-doc điển hình ("domain này có liên hệ gì với 5 domain đã bị blacklist trước đó?") mà Flat vector search trên corpus fact-check hiện tại không trả lời được. GraphRAG chỉ nên bổ sung cho **retrieve_evidence/verify_claim** ở các case rủi ro trung bình-cao cần giải thích, không thay thế toàn bộ rule engine tốc độ cao ở đầu pipeline.
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: `Domain`, `URL`, `Registrant` (WHOIS), `IP/Hosting`, `Brand`, `Campaign`, `BlacklistEntry`
  - Relations: `REGISTERED_BY`, `RESOLVES_TO`, `IMPERSONATES` (brand), `SHARES_INFRA_WITH`, `PART_OF_CAMPAIGN`, `FLAGGED_BY` (nguồn: VirusTotal/Safe Browsing/security-admin)
- **Chiến lược xử lý Super-node & Entity Resolution:** Domain/hosting dùng chung (Cloudflare, URL shortener) sẽ thành super-node cực lớn — bắt buộc áp `SUPER_NODE_DEGREE` cap như lab này để tránh 1 IP shared-hosting kéo theo hàng nghìn domain không liên quan vào context. Entity Resolution cho `Registrant`/`Brand` nên dùng ngưỡng **cao** (gần 0.90, khác với 0.40 của lab) vì đây là domain an ninh — false merge (gộp nhầm 2 registrant khác nhau) nguy hiểm hơn nhiều so với bỏ sót liên kết.

---

## 🎯 TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | 3 | - |
| Khả năng kiểm soát AI Coding Agent | 3 | - |
| Chất lượng đồ thị tri thức xây dựng | 3 | - |
| Khả năng phân tích và debug hệ thống | 3 | - |

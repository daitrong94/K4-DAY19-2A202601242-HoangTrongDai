# Thuyết Minh Kỹ Thuật — 10 Câu Hỏi Bảo Vệ Kiến Trúc

**Học viên:** Hoàng Trọng Đại · **Khóa học:** AICB-K34 · Track 3: GraphRAG · **Ngày:** 20/08/2026

> File này tách riêng theo đúng tên gọi trong `RUBRIC.md` (mục 4.1). Nội dung đầy đủ hơn (kèm bảng benchmark, mapping bài giảng...) nằm trong `reports/lab_report.md`.

---

### 1. Coreference sai ở tình huống nào?
Trên 400 chunk đưa vào Coreference, **388/400 (97%)** trả về `unresolved_mentions=["COREF_BATCH_FAILED"]` — ví dụ `chunk_id=331537d8f978a369b442::c0000` ("Ryan Specialty... acquire Socius Insurance Services"). Đây không phải lỗi *suy diễn sai antecedent* mà là lỗi hạ tầng: `resolve_coref_batch()` gọi Groq API bị exception hàng loạt (model bị deprecate / rate-limit — xem câu 9), khiến khối `except Exception` trong `run_coref()` kích hoạt fallback giữ nguyên text gốc thay vì suy diễn. Hậu quả: không tạo False Edge do đoán sai đại từ (đúng tinh thần Conservative rule), nhưng coreference gần như mất tác dụng thực tế → một số quan hệ có thể bị bỏ sót (Recall thấp) thay vì bị sai (Precision vẫn giữ).

### 2. Entity threshold bao nhiêu, vì sao?
`threshold = 0.40` (hạ từ gợi ý 0.90). Với 400-chunk sample gồm phần lớn công ty nhỏ/độc lập, ngưỡng 0.90 khiến audit table **rỗng hoàn toàn (0 dòng)** — không đạt yêu cầu ≥10 dòng. Hạ threshold + `top_k=6` để có đủ ứng viên cho Lexical Guard thể hiện vai trò "người gác cổng cuối cùng" thay vì để vector similarity tự quyết.

### 3. Candidate nào similarity cao nhưng không nên merge?
`Annealing quantum computing` vs `Gate model quantum computing` (Technology, cùng do D-Wave Quantum Inc. phát triển) — similarity=0.625, nhưng `SequenceMatcher` ratio=0.764 ≥ 0.72 nên bị **`MERGE_VECTOR`** (gộp nhầm!). Đây là 2 công nghệ lượng tử khác bản chất (ủ nhiệt vs mô hình cổng), chỉ trùng cụm từ đuôi "quantum computing" — ví dụ thật về rủi ro False Merge, tương tự kịch bản `Apple` vs `Apple Watch` trong đề bài.

### 4. Top 3 super-node và degree?
| Hạng | Thực thể | Type | Degree |
|---|---|---|---|
| 1 | Railergy | Company | 5 |
| 1 | Apple | Company | 5 |
| 3 | Unacademy | Company | 3 |

Đồ thị cuối: 103 node / 62 edge, 0 cạnh thiếu provenance.

### 5. Vì sao ưu tiên edge mới nhất có thể đúng/sai?
*Đúng:* tránh bùng nổ context/token khi 1 entity có hàng nghìn cạnh, ưu tiên thông tin cập nhật — phù hợp câu hỏi "tình hình hiện tại của X". *Sai/rủi ro:* câu hỏi cần lịch sử xa (ví dụ "công ty đầu tiên X mua lại") sẽ bị cắt mất bởi `ORDER BY published_date DESC LIMIT 50`, tạo ảo giác context đầy đủ trong khi bị thiên lệch thời gian (recency bias). Lưu ý: trong đồ thị 103-node hiện tại chưa node nào vượt `SUPER_NODE_DEGREE=100` nên cơ chế cap 50-cạnh chưa thực sự được kích hoạt bằng dữ liệu thật (chỉ verify được code chạy không lỗi qua `test_supernode_policy()`).

### 6. Flat RAG thắng nhóm nào?
Không có nhóm nào Flat RAG thắng rõ rệt trong 5 câu Golden; ở nhóm cross-doc Flat RAG **thua** GraphRAG về Comprehensiveness (1.50 vs 3.00) và Multi-hop reasoning (2.00 vs 3.00). Flat RAG chỉ có lợi thế về **latency** ở câu multi-hop đơn giản (3.27s vs 12.56s) vì không cần traversal.

### 7. GraphRAG thắng nhóm nào?
**cross-doc** — ví dụ G05 (Apple 2023): GraphRAG đạt Comprehensiveness=5/Multi-hop=5 nhờ BFS thu thập toàn bộ cạnh của node Apple (degree=5) trải dài 3 tháng, `textualize()` sắp theo thời gian, giúp model tổng hợp đúng trình tự (Final Cut Pro/Logic Pro → M3/A17 Bionic → đối tác Arm) — điều Flat RAG's top-k=6 similarity không đảm bảo gom đủ cả 3 chunk trong 1 lần truy vấn.

### 8. Latency/token trade-off?
GraphRAG **không phải lúc nào cũng tốn token hơn** (ở factoid/multi-hop, token tương đương hoặc ít hơn nhờ context đồ thị cô đọng), nhưng **luôn tốn latency cao hơn** khi cần traversal nhiều hop (multi-hop: 12.56s vs 3.27s) do phải: gọi LLM trích seed → nhiều round-trip Neo4j theo BFS → generate trên cả GRAPH+VECTOR context gộp. Trade-off hợp lý khi câu hỏi thực sự cross-doc; lãng phí khi câu hỏi đơn giản.

### 9. AI Coding Agent đề xuất gì mà bạn không dùng, vì sao?
1. **Không chạy lại toàn bộ pipeline 300MB mỗi lần fix lỗi** — vì mỗi vòng coref+extraction tốn ngay ~199k/200k token TPD của Groq; thay vào đó xây **checkpoint/resume theo bước** (coref, extraction, entity-resolution lưu CSV, tự load lại nếu đã có).
2. **Không giữ nguyên threshold 0.90 mặc định** dù đó là số đề bài gợi ý — vì gây audit table rỗng; quyết định hạ xuống 0.40 và giải thích rõ lý do thay vì fake dữ liệu.
3. **Không gán cứng MANUAL_ALIASES cho Microsoft/Google/Meta/Apple** khi phát hiện dataset mẫu thực tế hầu như không nhắc các hãng này — thay vào đó viết lại 4/5 câu Golden (G02–G05) dựa trên thực thể *có thật* trong đồ thị đã trích xuất, để golden answers có căn cứ kiểm chứng được.

### 10. Scale 350MB: bottleneck đầu tiên là gì?
**LLM API rate-limit theo ngày (Tokens-Per-Day)** — chỉ 400 chunk đã chiếm gần hết 200.000 token/ngày/model (Groq free tier); mở rộng lên hàng chục nghìn chunk cần hàng triệu token/ngày. Giải pháp: async batch extraction + xoay vòng nhiều API key/provider, checkpoint/resume ở mức chunk, ANN index (HNSW) thay brute-force cho Entity Resolution khi entity > vài chục nghìn, Community Partitioning để chia nhỏ đồ thị, và cân nhắc model local/nhỏ cho các bước khối lượng lớn nhưng độ khó thấp.

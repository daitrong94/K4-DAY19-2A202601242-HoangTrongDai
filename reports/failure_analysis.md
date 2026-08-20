# Phân Tích Ca Lỗi — Flat RAG vs GraphRAG

**Học viên:** Hoàng Trọng Đại · **Khóa học:** AICB-K34 · Track 3: GraphRAG · **Ngày:** 20/08/2026

> File này tách riêng theo đúng tên gọi trong `RUBRIC.md` (mục 4.2). Dữ liệu gốc: `outputs/graphrag_eval_results.csv`.

---

## Ca lỗi 1: Flat RAG yếu, GraphRAG thắng rõ rệt (G05)

**Câu hỏi:** `G05` — "Based on multiple articles from 2023, how did Apple's technology activities evolve over the year?"

**Kết quả LLM Judge:**
| Metric | Flat RAG | GraphRAG |
|---|---|---|
| Comprehensiveness | 2 | 5 |
| Faithfulness | 5 | 5 |
| Multi-hop reasoning | 3 | 5 |
| Latency (s) | 14.5 | 4.2 |

**Root-cause vì sao Flat RAG yếu hơn:**
Dữ liệu về Apple nằm rải rác ở **3 chunk khác nhau, cách nhau 4 tháng** (Final Cut Pro/Logic Pro — 05/2023; M3/A17 Bionic — 08/2023; đối tác Arm — 09/2023). Flat RAG dùng vector similarity top-k=6 trên toàn bộ 3000 chunk — không có cơ chế nào đảm bảo cả 3 chunk "Apple" cùng lọt vào top-6 cho một câu hỏi tổng hợp thời gian, vì similarity được tính độc lập theo ngữ nghĩa tức thời, không có khái niệm "cùng 1 entity, khác thời điểm".

**GraphRAG giải quyết như thế nào:**
Seed extraction nhận diện entity "Apple" trong câu hỏi → BFS traversal (`retrieve_graph_context`) thu thập **toàn bộ cạnh gắn với node Apple** (degree=5) bất kể chunk nguồn → `textualize()` sắp xếp theo `published_date DESC` → model nhận đủ ngữ cảnh theo đúng trình tự thời gian. Judge rationale xác nhận: *"provides a detailed chronological account... including... advancements in silicon technology, and a partnership..."*.

**Kết luận:** Đây là minh chứng trực tiếp cho luận điểm cốt lõi của GraphRAG — khi thông tin về 1 entity phân mảnh qua nhiều tài liệu theo thời gian, graph traversal theo entity ID vượt trội vector similarity thuần túy.

---

## Ca lỗi 2: Cả hai cùng thất bại một phần (G03 — "Aaritya Technologies" vô hình)

**Câu hỏi:** `G03` — So sánh 2 sự kiện gọi vốn tháng 9/2023: Meeno và Aaritya Technologies, ai đầu tư và khác nhau ra sao.

**Kết quả:** Cả Flat RAG và GraphRAG đều trả lời đúng phần **Meeno** (Sequoia dẫn đầu vòng seed $3.9M, chunk `212476770ac78703d3b7`) nhưng **cả hai đều báo "insufficient evidence"** cho phần Aaritya Technologies, dù dữ liệu thực sự tồn tại trong Neo4j:
```
Accel -INVESTED_IN-> Aaritya Technologies | chunk=6ef23bfa65272a9d88a4
Elevation Capital -INVESTED_IN-> Aaritya Technologies | chunk=6ef23bfa65272a9d88a4
```

**Truy vết nguyên nhân gốc rễ:**
1. Kiểm tra Neo4j xác nhận triples tồn tại → loại trừ khả năng "extraction bỏ sót".
2. Kiểm tra `entity_type` của node "Aaritya Technologies" trong Neo4j → phát hiện bị gắn **`Technology`** thay vì **`Company`** (đây thực chất là tên startup mới của cựu CTO Swiggy, Dale Vaz — lỗi phân loại của bước NER+RE khi LLM nhầm tên riêng công ty với tên sản phẩm/công nghệ).
3. Vì GraphRAG's seed matcher/graph traversal có thể lọc theo type ở một số đường truy vấn, và Flat RAG's top-k similarity cho câu hỏi ghép 2 chủ thể (Meeno + Aaritya) bị chunk "Meeno" có embedding mạnh hơn "che" mất chunk "Aaritya" trong cùng lần retrieve top-k=6 → cả 2 hệ thống đều không đưa được evidence về Aaritya vào context.

**Đề xuất khắc phục:**
1. Thêm bước validation/second-pass sau NER+RE: entity đóng vai trò *chủ ngữ của INVESTED_IN/FOUNDED* nhưng được gắn type `Technology` nên bị flag để review lại (heuristic: các relation này về logic luôn có source/target là Company hoặc Person).
2. Với Flat RAG: tăng `k` hoặc dùng MMR (Maximal Marginal Relevance) khi câu hỏi có nhiều chủ thể để tránh 1 chủ thể "che" chủ thể còn lại.
3. Với GraphRAG: nới lỏng seed matching để không phụ thuộc hoàn toàn vào entity_type đã gắn (fallback fuzzy match không lọc type khi exact-match theo type thất bại).

**Kết luận:** Đây không phải lỗi retrieval mà là lỗi **lan truyền từ bước Extraction** (entity type sai) xuống toàn bộ pipeline downstream — minh chứng cho tầm quan trọng của validation ở tầng Schema Guard, không chỉ chặn loại quan hệ/node ngoài allowlist mà còn cần phát hiện gán sai type *trong phạm vi* allowlist.

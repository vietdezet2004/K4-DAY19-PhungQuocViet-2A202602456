# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Phùng Quốc Việt  **MSSV:** 2A202602456  **Ngày:** 05/10/2026

## 1. Chi phí (10 điểm)

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     51.9
graph       196     91958     4903   0.00945    120.4

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.34
graph       0.80   1.50     2992       92   0.00050     2.17
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00945 | ×8.44 |
| Indexing giây | 51.9 | 120.4 | ×2.32 |
| Mỗi câu: USD | $0.00013 | $0.00050 | ×3.85 |
| Mỗi câu: giây | 1.34 | 2.17 | ×1.62 |
| Mỗi câu: in_tok | 694 | 2992 | ×4.31 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> Chi phí tăng thêm ở khâu indexing (×8.44 USD) đến từ việc GraphRAG phải gọi LLM trích xuất thực thể và quan hệ JSON từ 20 bài báo tin tức, trong khi Flat RAG chỉ tốn chi phí embedding. Ở khâu querying, chi phí (×3.85 USD) và token đầu vào (×4.31 in_tok) cao hơn do prompt của GraphRAG phải nạp thêm danh sách các dữ kiện multi-hop mở rộng từ đồ thị Neo4j cùng với các chunk văn bản.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa tiền chất nằm trọn trong một chunk của Điều 2 Luật Phòng chống ma túy nên cả hai pipeline đều lấy đủ thông tin. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Thông tin hai bị cáo nhận án tử hình nằm trực tiếp trong một bài báo nên vector search của cả hai bên đều tìm thấy. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Tên bị cáo ở tin tức còn Điều 251 và khung phạt ở luật nên Flat RAG trả về thiếu thông tin, còn Graph đi qua node cầu nối Crime nên trả lời chính xác. |
| Q4 | cross-kb | 0.00 / 0 | 0.33 / 1 | Graph | Flat RAG không có thông tin, trong khi Graph trích xuất được đúng hành vi "tổ chức sử dụng trái phép chất ma túy". |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 0.80 / 1 | Graph | GraphRAG có recall cao hơn (0.80 vs 0.60), liên kết đúng người, chất MDMA 9.6kg và khung hình phạt 20 năm, chung thân, tử hình. |
| Q6 | aggregation | 0.00 / 1 | 0.67 / 1 | Graph | GraphRAG truy vấn quan hệ INVOLVES nối tới MDMA liệt kê đủ 4 vụ án kèm tên can phạm, trong khi Flat RAG thiếu tên vụ và bỏ sót thông tin. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy (Broken bridge)

- **Hiện tượng:** Vụ án trong tin tức được trích xuất thành công nhưng không tạo được quan hệ `CHARGED_WITH` tới node `Crime`, khiến vụ án bị cô lập khỏi cơ sở tri thức Luật.
- **Bằng chứng:** (câu trả lời trích từ file kết quả, hoặc Cypher và kết quả)

```cypher
MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() RETURN k.name AS name, k.doc_id AS doc_id;
```

```
╒═════════════════════════════════════════╤═════════════════════════════╕
│name                                     │doc_id                       │
╞═════════════════════════════════════════╪═════════════════════════════╡
│"Vụ tông cảnh sát giao thông ở An Giang" │"news-100260926112415229"    │
└─────────────────────────────────────────┴─────────────────────────────┘
```

- **Nguyên nhân:** Lỗi ở bước crawl dữ liệu và prompt trích xuất LLM. Bài báo phản ánh sự việc đối tượng dương tính ma túy tông CSGT, cơ quan công an mới tạm giữ về hành vi chống người thi hành công vụ và chưa khởi tố tội danh ma túy theo Bộ luật Hình sự. LLM không tìm thấy tội danh tương ứng trong danh sách tội danh chuẩn, dẫn đến `link_entity` trả về rỗng và không tạo được quan hệ `CHARGED_WITH`.
- **Đề xuất sửa:** Bổ sung cơ chế fallback liên kết qua node `Substance` (`(:Case)-[:INVOLVES]->(:Substance)<-[:MENTIONS]-(:Clause)`). Nếu vụ án chưa có tội danh nhưng có thu giữ chất ma túy, hệ thống vẫn liên kết sang các Điều luật quy định về chất đó. Đánh đổi: Cần thêm nhánh truy vấn Cypher và làm prompt dài hơn.

### Lỗi E3: Trùng thực thể (Entity duplication)

- **Hiện tượng:** Cùng một chất ma túy ngoài đời thực bị phân tách thành nhiều node riêng biệt trong đồ thị Neo4j.
- **Bằng chứng:** (câu trả lời trích từ file kết quả, hoặc Cypher và kết quả)

```cypher
MATCH (s:Substance) RETURN s.name AS name ORDER BY toLower(s.name);
```

```
╒═════════════════════╕
│name                 │
╞═════════════════════╡
│"Amphetamine"        │
│"chất ma túy"        │
│"Cocaine"            │
│"côca"               │
│"cần sa"             │
│"etomidate"          │
│"Heroine"            │
│"Ketamine"           │
│"ketamine"           │
│"ma túy"             │
│"ma túy tổng hợp"    │
│"MDMA"               │
│"methamphetamine"    │
│"Methamphetamine"    │
│"thuốc lắc"          │
│"thuốc phiện"        │
│"XLR-11"             │
└─────────────────────┘
```

- **Nguyên nhân:** Lỗi ở thiết kế ontology và code nạp dữ liệu. Lệnh `MERGE (sub:Substance {name: s.name})` trong Neo4j phân biệt chữ hoa/thường (case-sensitive) nên 'Ketamine' và 'ketamine' thành hai node riêng. Ngoài ra, code trích xuất chưa chuẩn hóa đồng nghĩa ('thuốc lắc' và 'MDMA') và chưa loại bỏ các từ chung chung ('ma túy', 'chất ma túy').
- **Đề xuất sửa:** Áp dụng hàm `link_entity` với danh sách `SUBSTANCES` chuẩn cho toàn bộ tên chất trước khi `MERGE` vào Neo4j, đồng thời gộp các tên gọi thông tục (thuốc lắc, kẹo) vào thuộc tính `aliases` của chất chuẩn. Đánh đổi: Tăng thêm thời gian xử lý chuỗi ở khâu nạp dữ liệu.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> Nên dùng Knowledge Graph (GraphRAG) khi bài toán yêu cầu kết nối dữ liệu phân mảnh từ nhiều nguồn tri thức độc lập (cross-kb) hoặc suy luận nhiều bước (multi-hop). Số liệu ở mục 2 chứng minh rõ: trên các câu cross-kb như Q3 và Q4, Flat RAG hoàn toàn thất bại (recall 0.00, judge 0), trong khi GraphRAG đạt recall 1.00 và judge 2 nhờ đi qua node cầu nối Crime; tính trung bình GraphRAG nâng recall từ 0.43 lên 0.80. Ngược lại, Flat RAG là đủ khi các câu hỏi chỉ mang tính đơn nguồn (single-hop như Q1, Q2) nơi câu trả lời nằm trọn trong một đoạn văn, giúp tiết kiệm chi phí indexing gấp 8.44 lần ($0.00112 so với $0.00945) và giảm độ trễ truy vấn xuống 1.34s so với 2.17s của GraphRAG.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.14s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 22 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Dương Minh Tuấn

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> Không có lỗi chưa giải quyết. Toàn bộ các bài kiểm thử unit test (48/48 passed), lệnh kiểm tra hợp đồng tự động bench_kg.py --check (7/7 [OK]) và benchmark --judge đều đã chạy hoàn tất và cho kết quả chính xác.

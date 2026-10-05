# Ghi chép benchmark và khám phá graph — Day 19

Đây là ghi chép cho Bước 7–8, chưa phải `REPORT_KG.md`. Ontology dùng 7 label/7 quan hệ gợi ý, không đăng ký bonus. Ba ảnh Neo4j Browser đã được chụp và kiểm tra; phần Q-D dùng Giang Quốc Khánh.

## Lượt đo và tính hợp lệ

- Lệnh: `.venv/Scripts/python.exe bench_kg.py --judge`, mặc định `top_k=3`, `chunk_size=800`; chạy một lần, exit code `0`. Trước lượt đo graph là bản nhỏ sau `--check` (146 node/289 cạnh); sau lượt đo graph đầy đủ vẫn chạy trong container lab `neo4j-drug-kg` ở `bolt://localhost:7687`.
- Provider thực tế: chat `openai:gpt-4o-mini`, embedding `openai:text-embedding-3-small`. Không có file `ket_qua_benchmark_kg.txt` cũ trước lượt đo.
- Output kết thúc:

```text
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
Q1 flat  recall=1.00 1.81s $0.00012
Q1 graph recall=1.00 2.69s $0.00061
Q2 flat  recall=1.00 1.69s $0.00014
Q2 graph recall=1.00 2.66s $0.00073
Q3 flat  recall=0.00 1.72s $0.00008
Q3 graph recall=1.00 2.39s $0.00090
Q4 flat  recall=0.00 1.86s $0.00007
Q4 graph recall=1.00 3.19s $0.00116
Q5 flat  recall=0.60 2.23s $0.00016
Q5 graph recall=1.00 2.65s $0.00083
Q6 flat  recall=0.00 3.20s $0.00018
Q6 graph recall=0.33 3.57s $0.00123
Chat model: openai:gpt-4o-mini | Embedding: openai:text-embedding-3-small | top_k=3 | chunk_size=800 | chunks=176 | KG: 204 nodes / 381 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112    138.0
graph       196     91958     4627   0.00928    218.7

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     2.08
graph       0.89   1.83     5782       83   0.00091     2.86

Saved ket_qua_benchmark_kg.txt
```

File có đủ header, ba phần `Indexing`, `Querying`, `Per question` và đúng 12 mục Q1–Q6 × flat/graph; mọi mục có judge thuộc `0/1/2`. `KG: 204 nodes / 381 rels` khớp `graph.stats()` đọc sau benchmark. Không sửa tay file kết quả.

### Indexing (one-off) — chép nguyên số liệu

```text
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112    138.0
graph       196     91958     4627   0.00928    218.7
```

### Querying (mean per question) — chép nguyên số liệu

```text
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     2.08
graph       0.89   1.83     5782       83   0.00091     2.86
```

Theo số hiển thị đã làm tròn, GraphRAG tăng USD indexing `0.00928 - 0.00112 = 0.00816` (khoảng 8,29 lần mức Flat) và USD mỗi câu `0.00091 - 0.00013 = 0.00078` (khoảng 7 lần), thêm khoảng `0,78` giây/câu. `graph` indexing trong script bằng `flat_index + kg_build`, nên đã gồm chi phí embed chung. `metered(...)` chỉ bao quanh mỗi lần `agent.answer`; lệnh `llm.chat(JUDGE_PROMPT, json_mode=True)` ở `bench_kg.py` nằm **sau** lần đo đó. Vì vậy bảng Querying không tính chi phí judge và các USD trong bảng không phải tổng hóa đơn toàn lượt. Chênh lệch là quan sát của một lượt đo; chưa cô lập nguyên nhân hoặc biến thiên giữa các lượt.

## Đọc từng câu

| Câu / loại | Flat recall / judge | Graph recall / judge | Kết luận dựa trên câu trả lời |
|---|---:|---:|---|
| Q1 / single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa về nội dung tiền chất; Graph ghi thêm Điều 2 khoản 4, tốn thêm truy vấn. |
| Q2 / single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa; cả hai nêu Trần Thanh Tuấn và Trần Minh Tâm. |
| Q3 / cross-kb | 0.00 / 0 | 1.00 / 2 | Graph thắng; Flat trả “Không đủ thông tin.”, Graph có 36 tháng, Điều 251 và khung khoản 1. |
| Q4 / cross-kb | 0.00 / 0 | 1.00 / 2 | Graph thắng; nêu hành vi tổ chức sử dụng, Điều 255, khung tối đa 20 năm hoặc chung thân. Câu trả lời nói “bị bắt”, không khẳng định đã bị kết án. |
| Q5 / cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph thắng theo hai điểm số và xác định Điều 250 khoản 4; tuy nhiên câu Graph bỏ Ketamine dù nguồn có, nên judge 2 không đồng nghĩa trả đủ mọi vế. Flat ghi mơ hồ “khoản b)” và không có Điều 250. |
| Q6 / aggregation | 0.00 / 1 | 0.33 / 1 | Judge hòa, recall Graph cao hơn nhưng cả hai chưa đủ. Graph nêu thêm vụ hơn 36kg là có MDMA dù graph và bài nguồn không chứng minh; xem E5. |

Q3–Q5: GraphRAG **thắng cả ba theo recall và judge của lượt đo**, nhưng Q5 thiếu Ketamine. Recall là phép dò chuỗi `must_include`, còn judge là đánh giá LLM; phải đọc câu trả lời và nguồn trước khi coi điểm là đúng.

## Graph đầy đủ và truy vấn ảnh

Q-A đã chạy (7 dòng, tổng 204):

```cypher
MATCH (n) RETURN labels(n)[0] AS label, count(*) AS n ORDER BY n DESC;
```

| Label | n |
|---|---:|
| Clause | 99 |
| Person | 36 |
| Article | 18 |
| Substance | 17 |
| Case | 14 |
| Crime | 13 |
| Location | 7 |

Truy vấn cạnh đã chạy (7 dòng, tổng 381):

```cypher
MATCH ()-[r]->() RETURN type(r) AS rel, count(*) AS n ORDER BY n DESC;
```

| Quan hệ | n |
|---|---:|
| MENTIONS | 169 |
| HAS_CLAUSE | 99 |
| INVOLVED_IN | 43 |
| INVOLVES | 24 |
| CHARGED_WITH | 19 |
| LOCATED_IN | 14 |
| DEFINES | 13 |

Hai tập loại đúng 7 label và 7 quan hệ trong `report/ONTOLOGY.md`; số lượng node/cạnh phần tin phụ thuộc extraction của lượt này.

Q-B đã thử bằng driver, trả 25 path; path đầu qua `Nguyễn Thị Mai Anh → Vụ tổ chức sử dụng ma túy tại Sầm Sơn → tàng trữ trái phép chất ma túy → Điều 249 BLHS`. Dùng Cypher này để chụp Graph và Results overview; bộ lọc charge bảo đảm đường luật khớp người:

```cypher
MATCH p=(person:Person)-[r:INVOLVED_IN]->(:Case)-[:CHARGED_WITH]->(c:Crime)<-[:DEFINES]-(:Article)
WHERE r.charge = c.name
RETURN p LIMIT 25;
```

Trước khi chọn ảnh cuối, Q-D đã được thử bằng driver với ứng viên **Cái Quang Huy**, trả 3 cặp `p,q`; đó là truy vấn chuẩn bị, không phải người trong ảnh đã nộp. Đường người → vụ → tội → luật dẫn tới `Điều 250 BLHS`; các nhánh từ vụ là MDMA `9.6kg`, Ketamine `406g` và Hà Nội. Graph cũng có các ứng viên **Dương Minh Tuấn** (Điều 255) và **Trần Thanh Tuấn** (Điều 251). Người được chọn và xuất hiện trong ảnh `kg_my_case.png` thực tế là **Giang Quốc Khánh**. Truy vấn chỉ đọc xác nhận `r.charge = c.name`, tội mua bán trái phép chất ma túy, `Điều 251 BLHS`, Case `Vụ bắt giang hồ 'Hoàng Nato' và 126 người liên quan 8 đường dây ma túy`, `doc_id=news-100260920221957595`:

```cypher
MATCH (p:Person {name:'Giang Quốc Khánh'})-[r:INVOLVED_IN]->(k:Case)-[:CHARGED_WITH]->(c:Crime)<-[:DEFINES]-(a:Article)
WHERE r.charge = c.name
RETURN p.name, r.charge, k.name, k.doc_id, a.id;
```

```text
1 row: Giang Quốc Khánh | mua bán trái phép chất ma túy |
Vụ bắt giang hồ 'Hoàng Nato' và 126 người liên quan 8 đường dây ma túy |
news-100260920221957595 | Điều 251 BLHS
```

Cypher Cái Quang Huy đã chuẩn bị trước, không phải người trong ảnh đã chụp:

```cypher
MATCH p=(person:Person {name:'Cái Quang Huy'})-[r:INVOLVED_IN]->(k:Case)-[:CHARGED_WITH]->(c:Crime)<-[:DEFINES]-(a:Article)
WHERE r.charge = c.name
OPTIONAL MATCH q=(k)-[:INVOLVES|LOCATED_IN]->()
RETURN p, q LIMIT 25;
```

Trong Neo4j Browser, lưu ảnh cửa sổ thật, thấy ô truy vấn và Results overview; dùng `:clear` trước mỗi truy vấn:

| File cần lưu | Truy vấn | Trạng thái |
|---|---|---|
| `report/img/kg_count.png` | Q-A | Đã chụp; PNG không rỗng, thấy query và đủ 7 label trong bảng |
| `report/img/kg_cross_kb.png` | Q-B | Đã chụp; PNG không rỗng, thấy graph, Results overview và 4 loại node/3 loại cạnh của đường đi |
| `report/img/kg_my_case.png` | Q-D — Giang Quốc Khánh | Đã chụp; PNG không rỗng, thấy query, người/vụ/tội/Điều và Substance/Location trong Results overview |

Ba PNG đã được kiểm tra trực quan là ảnh Neo4j Browser thật. Các truy vấn được chuẩn bị/đối chiếu bằng driver; ảnh cho thấy truy vấn đã được chạy trong Browser. Không sửa, dựng bằng code hoặc cắt ghép ảnh.

## Bằng chứng lỗi cho báo cáo sau

### E3 — Một chất thành hai node vì khác chữ hoa/thường

1. **Hiện tượng:** query `MATCH (s:Substance) WHERE toLower(s.name) IN ['ketamine','methamphetamine'] RETURN s.name ORDER BY toLower(s.name),s.name` trả bốn node riêng: `Ketamine`, `ketamine`, `Methamphetamine`, `methamphetamine`.
2. **Bằng chứng:** `MATCH (k:Case)-[r:INVOLVES]->(s:Substance) WHERE toLower(s.name) IN ['ketamine','methamphetamine'] RETURN k.name,k.doc_id,s.name,r.amount ORDER BY s.name,k.name` cho `Ketamine` ở Case Cái Quang Huy (`news-100260917203001265`, `406g`) và `ketamine` ở Case Hoàng Nato (`news-100260920221957595`, amount extraction `khoảng 100g`) cùng Case Viện Pháp y (`news-100260924105118645`, amount rỗng). Nó cũng trả `Methamphetamine` ở Case 840kg tại Campuchia (`news-100260924145818945`) và `methamphetamine` ở Case Sầm Sơn (`news-100260930085028036`) cùng Case Viện Pháp y (`news-100260924105118645`). Nguồn Cái Quang Huy xác nhận gần 406g Ketamine; nguồn Viện Pháp y liệt kê “MDMA, ketamine, methamphetamine, cần sa”. Bài Hoàng Nato chỉ nói “khoảng 100g ma túy tổng hợp các loại”, không xác nhận lượng Ketamine cụ thể. Vì vậy giá trị 100g trên cạnh `ketamine` là lượng do extraction gắn vào, không phải lượng Ketamine được nguồn xác nhận.
3. **Nguyên nhân:** `Substance` MERGE theo `name`, phân biệt hoa/thường; phía tin không có bước chuẩn hóa/hậu kiểm whitelist tên chất. Điều này giải thích các node trùng theo cách viết. Việc gắn lượng tổng hợp 100g cho riêng Ketamine có khả năng là liên kết extraction quá rộng; chưa có JSON/prompt gốc của lần build để chứng minh chính xác bước gây ra.
4. **Đề xuất:** trước `add_news_case` trong `src/graph.py`, chuẩn hóa không phân biệt hoa/thường về tên chuẩn có căn cứ; chỉ gắn lượng khi nguồn chỉ rõ lượng thuộc chất nào, còn lượng “ma túy tổng hợp các loại” nên giữ là lượng tổng hợp hoặc để không rõ chất. Đánh đổi: cần rà alias để tránh gộp nhầm chất và sẽ tăng thời gian hậu kiểm, có thể giảm độ bao phủ extraction.

### E5 — Q6 GraphRAG nêu vụ không có cạnh MDMA

1. **Hiện tượng:** Q6 GraphRAG liệt kê 5 vụ, trong đó có `Vụ mua bán hơn 36kg ma túy tại TP.HCM` là liên quan MDMA.
2. **Bằng chứng:** nguyên văn Q6 graph: “5. Vụ 'Vụ mua bán hơn 36kg ma túy tại TP.HCM' - liên quan đến MDMA (khối lượng chưa rõ).” Truy vấn độc lập:

   ```cypher
   MATCH (k:Case)-[r:INVOLVES]->(s:Substance {name:'MDMA'})
   RETURN DISTINCT k.name, k.doc_id, r.amount ORDER BY k.name;
   ```

   chỉ trả **4 Case**: `Vụ góp tiền mua ma túy tại Hà Nội` (`news-100260918080821054`, `5 viên`), `Vụ tổ chức sử dụng ma túy tại Sầm Sơn` (`news-100260930085028036`, `0,686g`), `Vụ vận chuyển ma túy từ Đức về Việt Nam` (`news-100260917203001265`, `9.6kg`), `Vụ án tại Viện Pháp y tâm thần Trung ương` (`news-100260924105118645`, lượng rỗng). Truy vấn `MATCH (k:Case {name:'Vụ mua bán hơn 36kg ma túy tại TP.HCM'}) OPTIONAL MATCH (k)-[r:INVOLVES]->(s:Substance) RETURN k.doc_id,s.name,r.amount` cho `news-100260928173914514`, `ma túy`, `36kg` — không có MDMA. Bài nguồn này chỉ nói “ma túy các loại”, không xác định MDMA.
3. **Nguyên nhân:** câu trả lời do `GraphRAGAgent.answer` sinh từ cả facts và top-k chunks; trong ngữ cảnh đó LLM đã vượt quá tập Case có cạnh MDMA. Không thể kết luận riêng facts hay chunk nào gây ra nếu chưa lưu prompt của lượt đo. Bước liên quan là tổng hợp câu trả lời Q6, với prompt cho phép diễn giải chunk cùng graph; nguyên nhân chi tiết là giả thuyết cần kiểm tra thêm.
4. **Đề xuất:** ở `Neo4jGraph.context`/`GRAPH_PROMPT` trong `src/graph.py`, với câu liệt kê theo chất, trình bày tập Case truy vấn trực tiếp như danh sách được phép nêu và yêu cầu không thêm vụ ngoài tập; có thể hậu kiểm tên Case trên câu trả lời. Đánh đổi: query toàn graph và hậu kiểm tốn thời gian, có thể bỏ sót vụ nếu extraction thiếu hoặc chất có tên biến thể như E3.

### E4 — Điểm Q5 che phần chất bị bỏ sót

1. **Hiện tượng:** Q5 graph có `recall=1.00`, `judge=2`, nhưng không nêu Ketamine dù câu hỏi hỏi các loại ma túy và nguồn nêu cả MDMA lẫn Ketamine.
2. **Bằng chứng:** nguyên văn Q5 graph: “Cái Quang Huy bị truy tố về tội vận chuyển trái phép chất ma túy, với loại ma túy là MDMA. Với khối lượng MDMA là hơn 9,6kg, điều luật tương ứng được áp dụng là Điều 250 BLHS khoản 4. Khung hình phạt là 20 năm, tù chung thân hoặc tử hình.” Nguồn `news-100260917203001265` ghi “hơn 9,6kg MDMA và gần 406g Ketamine”; graph có `Case-[:INVOLVES {amount:'406g'}]->(:Substance {name:'Ketamine'})`. `data/benchmark_kg.json` đặt `must_include` Q5 là `["vận chuyển", "MDMA", "Điều 250", "khoản 4", "tử hình"]`, không có Ketamine.
3. **Nguyên nhân:** recall chỉ kiểm tra các chuỗi trong `must_include`; judge LLM chấm 2 cho câu có phần chính về MDMA và khung luật nhưng thiếu chất thứ hai. Đây là giới hạn của phép đo, không chứng minh graph thiếu Ketamine. Vì chỉ một lượt judge, không khẳng định judge luôn chấm vậy.
4. **Đề xuất:** ở `data/benchmark_kg.json`/rubric judge cho lần đánh giá sau, thêm kiểm tra Ketamine hoặc tách điểm theo các vế tội–chất–khối lượng–khoản–khung; đồng thời kiểm tra Q5 answer với danh sách `INVOLVES` của đúng Case. Đánh đổi: tiêu chí chặt hơn có thể giảm điểm dù phần trọng tâm MDMA đúng; thay benchmark sẽ mất khả năng so sánh trực tiếp với lượt này, nên **không sửa dữ liệu trong lượt đo hiện tại**.

### Trường hợp E1 không tính là lỗi

`MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() RETURN k.name,k.doc_id` trả một Case: vụ tông cảnh sát giao thông An Giang (`news-100260926112415229`). Bài nguồn nói Nguyễn Minh Nhân bị khởi tố về **chống người thi hành công vụ**; dùng rượu/ma túy là tình tiết trong tin, không phải cáo buộc một tội ma túy thuộc 13 `Crime` của KB luật. Không nên ép vụ này nối sang tội ma túy chỉ để đủ cạnh.

## Giới hạn và việc còn chờ

- Benchmark chỉ một lượt; extraction tin và judge có thể biến động. Chưa thử tính ngẫu nhiên bằng lượt thứ hai.
- `Case` MERGE theo tên với một `doc_id` có thể ghi đè provenance; số Case 14 không bằng 20 bài báo vì có bài không ra Case và/hoặc nhiều bài được gộp tên. Chưa điều tra hết từng trường hợp.
- Q5 `amount` là chuỗi, `Clause.text` giữ ngưỡng; không có suy luận số học tự động để xác nhận khoản áp dụng từ 9,6kg.
- Ba ảnh thật đã có và được kiểm tra: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`; Q-D dùng Giang Quốc Khánh. `REPORT_KG.md` đã được hoàn thiện; notes bổ sung bằng chứng, không thay thế báo cáo nộp.
- Giữ nguyên graph đầy đủ sau benchmark; trước khi chụp ảnh không chạy lại `--check`, `--build` hoặc reset.

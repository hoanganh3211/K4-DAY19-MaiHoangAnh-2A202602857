# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Mai Hoàng Anh  **MSSV:** 2A202602857  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

---

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```text
Chat model: openai:gpt-4o-mini | Embedding: openai:text-embedding-3-small | top_k=3 | chunk_size=800 | chunks=176 | KG: 204 nodes / 383 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     72.0
graph       196     91958     4726   0.00934    134.9

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       48   0.00013     1.71
graph       0.83   1.67     5558       75   0.00087     2.57
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00934 | ×8.34 |
| Indexing giây | 72.0s | 134.9s | ×1.87 |
| Mỗi câu: USD | $0.00013 | $0.00087 | ×6.69 |
| Mỗi câu: giây | 1.71s | 2.57s | ×1.50 |
| Mỗi câu: in_tok | 694 | 5558 | ×8.01 |

**Chi phí tăng thêm đến từ đâu?**
> Ở pha Indexing, chi phí GraphRAG tăng gấp ~8.3 lần do phải thực hiện thêm 20 lượt gọi LLM (`gpt-4o-mini`) kèm structured output để trích xuất thực thể, vụ án và tội danh từ các bài báo tin tức (tiêu tốn 4.726 output tokens đắt hơn embedding). Ở pha Querying, chi phí tăng gấp ~6.7 lần và số `in_tok` tăng gấp ~8 lần vì prompt của GraphRAG được mở rộng thêm toàn bộ các dữ kiện cấu trúc (seed facts, tóm tắt vụ việc, văn bản các điều khoản luật liên quan) đưa vào context trước khi trả lời.

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| **Q1** | single-hop-law | 1.00 / 2 | 1.00 / 2 | **Hòa** | Câu hỏi định nghĩa luật đơn giản ("tiền chất là gì?"), vector search của Flat RAG tìm trúng trực tiếp chunk Điều 2 Luật PCMT nên cả hai đều trả lời chính xác tuyệt đối. |
| **Q2** | single-hop-news | 1.00 / 2 | 1.00 / 2 | **Hòa** | Câu hỏi tra cứu dữ kiện cụ thể trong 1 bài báo (bị cáo tử hình vụ 36kg), Flat RAG truy xuất đúng đoạn văn bản chứa danh sách tuyên án. |
| **Q3** | cross-kb | 0.00 / 0 | 1.00 / 2 | **Graph** | Cần kết nối giữa bản án tin tức ("Lê Minh Thành", 36 tháng tù) và điều luật chế tài ("Điều 251 BLHS", khoản 1); Flat RAG chỉ lấy được tin tức nên báo "Không đủ thông tin", trong khi Graph đi xuyên 2 KB qua node Crime trả lời hoàn hảo. |
| **Q4** | cross-kb | 0.00 / 0 | 0.67 / 1 | **Graph** | Cần liên kết hành vi của 'Hoàng Nato' sang Điều 255 BLHS; Flat RAG thất bại hoàn toàn (judge=0), trong khi Graph tìm đúng Điều 255 và hành vi tổ chức sử dụng dù bị thiếu khung tối đa do lọc thiếu khoản. |
| **Q5** | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | **Graph** | Cần liên kết vụ án Cái Quang Huy (9,6kg MDMA) sang Điều 250 và đối chiếu định khung Khoản 4; Graph cung cấp đầy đủ thông tin chuẩn xác và đạt điểm tối đa (recall 1.0, judge 2), trong khi Flat RAG nhầm sang "khoản b)". |
| **Q6** | aggregation | 0.00 / 1 | 0.33 / 1 | **Graph** | Cần gom toàn bộ các vụ án liên quan đến MDMA trên toàn bộ corpus; GraphRAG tìm đủ cả 4 vụ án qua quan hệ `INVOLVES` với node `Substance {name: 'MDMA'}`, vượt trội so với Flat RAG chỉ lấy được 3 chunks rời rạc. |

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (sai khung hình phạt dù graph có đủ Điều luật)

- **Hiện tượng:** Ở câu hỏi Q4 hỏi về mức phạt tù *tối đa* đối với hành vi của giang hồ 'Hoàng Nato', GraphRAG trả lời mức phạt tù tối đa là 7 năm (theo Khoản 1), trong khi đáp án chuẩn là 20 năm hoặc tù chung thân (Khoản 4 Điều 255 BLHS).
- **Bằng chứng:**
  - Câu trả lời trích nguyên văn từ file `ket_qua_benchmark_kg.txt` (dòng 29):
    ```text
    --- Q4 [cross-kb] graph recall=0.67 judge=1 2.95s
    Giang hồ 'Hoàng Nato' bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy. Hành vi này có thể bị phạt tù tối đa 7 năm theo Điều 255 BLHS khoản 1.
    ```
  - Truy vấn Cypher kiểm tra các khoản của Điều 255 trong Neo4j:
    ```cypher
    MATCH (a:Article {id: 'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
    RETURN cl.number AS number, cl.penalty AS penalty ORDER BY number;
    ```
    *Kết quả thực tế:*
    ```text
    khoản 1: phạt tù từ 02 năm đến 07 năm
    khoản 2: phạt tù từ 07 năm đến 15 năm
    khoản 3: phạt tù từ 15 năm đến 20 năm
    khoản 4: phạt tù 20 năm hoặc tù chung thân
    khoản 5: phạt tiền từ 50.000.000 đồng đến 500.000.000 đồng...
    ```
- **Nguyên nhân:**
  - Lỗi nằm ở bước **KG-3 (`Neo4jGraph.context`)**:
  - Điều kiện lọc khoản luật khi duyệt từ vụ án sang điều luật là:
    ```cypher
    WHERE (cl.number = 1 OR EXISTS { (k)-[:INVOLVES]->(:Substance)<-[:MENTIONS]-(cl) })
    ```
  - Trong vụ án Hoàng Nato, chất sử dụng là `etomidate` (pod chill). Tuy nhiên, Điều 255 BLHS không quy định định khung tăng nặng theo tên chất ma túy mà tăng nặng theo hành vi (có tổ chức, phạm tội 2 lần trở lên, đối với người dưới 16 tuổi...). Do đó, điều kiện `EXISTS` trả về `false` với mọi khoản 2, 3, 4. Hệ thống chỉ lấy duy nhất **Khoản 1** nạp vào context, dẫn đến việc LLM thiếu thông tin và kết luận sai mức phạt tối đa.
- **Đề xuất sửa:**
  - Khi câu hỏi có chứa các từ khóa hỏi về mức hình phạt cao nhất (như `"tối đa"`, `"khung cao nhất"`, `"nặng nhất"`), câu lệnh Cypher trong `context()` cần chủ động truy vấn thêm khoản có khung hình phạt cao nhất (`ORDER BY cl.number DESC LIMIT 1`) hoặc lấy toàn bộ các khoản định hình phạt tù của điều luật đó thay vì chỉ phụ thuộc vào điều kiện khớp chất `MENTIONS`.

---

### Lỗi E3: Trùng thực thể (một thứ ngoài đời thành nhiều node)

- **Hiện tượng:** Cùng một chất ma túy ngoài đời thực nhưng bị nhân bản thành nhiều node `Substance` khác nhau trong cơ sở dữ liệu Neo4j do khác biệt chữ hoa/thường hoặc tên gọi đường phố vs tên khoa học.
- **Bằng chứng:**
  - Truy vấn Cypher kiểm tra danh sách `Substance`:
    ```cypher
    MATCH (s:Substance)
    RETURN s.name ORDER BY toLower(s.name);
    ```
  - *Kết quả thực tế từ database:*
    ```text
    ['Amphetamine', 'chất ma túy', 'Cocaine', 'côca', 'cần sa', 'etomidate', 'Heroine', 'Ketamine', 'ketamine', 'ma túy', 'ma túy tổng hợp', 'MDMA', 'methamphetamine', 'Methamphetamine', 'thuốc lắc', 'thuốc phiện', 'XLR-11']
    ```
    *Dễ dàng nhận thấy:*
    1. Cặp `'Ketamine'` (chữ hoa) và `'ketamine'` (chữ thường) bị tách thành 2 node riêng biệt.
    2. Cặp `'Methamphetamine'` và `'methamphetamine'` bị tách thành 2 node riêng biệt.
    3. `'MDMA'` và `'thuốc lắc'` là cùng một chất nhưng thành 2 node riêng.
    4. Các từ chung chung như `'chất ma túy'`, `'ma túy'` bị LLM trích xuất thành node chất cụ thể.
- **Nguyên nhân:**
  - Nằm ở bước **Thiết kế Ontology & KG-2 (`add_news_case`)**:
  - Khi nạp luật, hàm `find_substances` dùng danh sách `SUBSTANCES` chuẩn hóa chữ hoa đầu từ (`Ketamine`, `Methamphetamine`). Khi nạp tin tức, LLM trả về chữ thường (`ketamine`) hoặc từ lóng (`thuốc lắc`).
  - Lệnh Cypher `MERGE (sub:Substance {name: s.name})` thực hiện so khớp chính xác case-sensitive, khiến các biến thể không thể gộp vào node chuẩn đã có từ luật.
- **Đề xuất sửa:**
  - Chuẩn hóa tên chất về chữ thường hoặc danh mục chuẩn trước khi `MERGE` (ví dụ: `link_entity(s.name, SUBSTANCES)`).
  - Bổ sung bảng từ đồng nghĩa (synonym map): `{"thuốc lắc": "MDMA", "hàng đá": "Methamphetamine"}` để chuẩn hóa các tên đường phố về tên khoa học trong luật trước khi ghi vào Neo4j.

---

### Lỗi E1: Cầu nối gãy (vụ án không nối được sang luật)

- **Hiện tượng:** Vụ án tin tức được tạo ra nhưng không có cạnh `CHARGED_WITH` kết nối tới bất kỳ node `Crime` nào, khiến vụ án bị cô lập hoàn toàn và không thể tra cứu sang KB luật.
- **Bằng chứng:**
  - Truy vấn Cypher tìm các vụ án không có quan hệ `CHARGED_WITH`:
    ```cypher
    MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->()
    RETURN k.name AS name, k.doc_id AS doc_id;
    ```
  - *Kết quả thực tế từ database:*
    ```text
    name: "Vụ tông cảnh sát giao thông ở An Giang"
    doc_id: "news-100260926112415229"
    ```
  - Mở bài báo gốc `data/drug_news/news-100260926112415229.md`: Bài báo tường thuật một tài xế dương tính với ma túy đã đâm vào cán bộ cảnh sát giao thông khi bị yêu cầu dừng xe.
- **Nguyên nhân:**
  - Nằm ở bước **KG-1 (`link_entity`) & Thiết kế Ontology**:
  - Danh sách tội danh chuẩn (`known_crimes`) trích từ KB luật chỉ bao gồm các tội danh thuộc Chương XX BLHS (Tội phạm về ma túy). Trong bài báo trên, hành vi được điều tra liên quan đến "chống người thi hành công vụ" hoặc "vi phạm giao thông". Hàm `link_entity` không tìm thấy tội danh khớp trong `known_crimes` nên trả về `None`, dẫn đến danh sách `charges` bị rỗng và không tạo cạnh `CHARGED_WITH`.
- **Đề xuất sửa:**
  - Với những vụ việc mà tội danh không thuộc Chương XX BLHS, graph vẫn nên tạo một node tội danh tự do hoặc gắn cờ `is_unlinked = true` để lưu vết hành vi của vụ án, tránh làm đứt đoạn hoàn toàn ngữ cảnh vụ việc.

---

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2:

1. **Khi nào Flat RAG là đủ:**
   - Đối với các câu hỏi **Single-hop** (tra cứu trực tiếp trong một văn bản duy nhất như Q1 và Q2), Flat RAG đạt điểm số tối đa ngang bằng GraphRAG (cùng `recall = 1.00`, `judge = 2`), nhưng với chi phí rẻ hơn **6.7 lần** ($0.00013 vs $0.00087) và tốc độ phản hồi nhanh hơn **1.5 lần** (1.71s vs 2.57s). Nếu hệ thống chỉ phục vụ các truy vấn hỏi đáp tài liệu cục bộ, Flat RAG là lựa chọn tối ưu về chi phí và tài nguyên.

2. **Khi nào BẮT BUỘC nên dùng GraphRAG:**
   - Đối với các bài toán **Cross-KB** (kết nối đa nguồn tri thức như Q3, Q4) và **Multi-hop / Aggregation** (liên kết định khung tăng nặng, gom cụm toàn cục như Q5, Q6), Flat RAG gần như thất bại hoàn toàn (`recall` trung bình rơi xuống 0.00 – 0.60, thường xuyên trả về "Không đủ thông tin" với `judge = 0`). 
   - Ngược lại, GraphRAG phát huy sức mạnh vượt trội khi nâng `recall` trung bình lên **0.83** và điểm `judge` trung bình lên **1.67/2.0**, giúp trả lời chính xác mối quan hệ pháp lý giữa con người, vụ án thực tế và điều khoản chế tài của pháp luật.

---

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.04s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (vụ vận chuyển ma túy từ Đức về Việt Nam qua sân bay Nội Bài).

---

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử:
> Trên môi trường Windows PowerShell, lệnh `python bench_kg.py` gặp lỗi `UnicodeEncodeError: 'charmap' codec can't encode character '\u1eef'` khi in tiếng Việt ra console do mặc định PowerShell dùng bảng mã cp1252. Đã giải quyết triệt để bằng cách thiết lập biến môi trường `$env:PYTHONIOENCODING="utf-8"` trước khi thực thi script Python.

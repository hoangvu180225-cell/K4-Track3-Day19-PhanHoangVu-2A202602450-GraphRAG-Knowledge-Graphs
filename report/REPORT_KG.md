# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Phan Hoàng Vũ  **MSSV:** 2A202602450  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    115.9
graph       196     34619     5840   0.00580    173.2

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       70   0.00010     4.49
graph       0.94   1.83     5633      143   0.00062     2.03
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00000 | $0.00580 | N/A (+$0.00580) |
| Indexing giây | 115.9s | 173.2s | ×1.49 |
| Mỗi câu: USD | $0.00010 | $0.00062 | ×6.20 |
| Mỗi câu: giây | 4.49s | 2.03s | ×0.45 |
| Mỗi câu: in_tok | 696 | 5633 | ×8.09 |

**Chi phí tăng thêm đến từ đâu?**
1. **Ở giai đoạn Indexing (one-off)**: Chi phí tăng thêm $0.00580 và 57.3 giây đến từ 20 lượt gọi LLM (`gemini-3.5-flash-lite`) với `json_mode=True` để trích xuất có cấu trúc các thực thể (`Case`, `Person`, `Substance`, `Location`) từ 20 bài báo tin tức (tiêu tốn 34,619 input tokens và 5,840 output tokens). Ngược lại, phần luật được trích xuất bằng regex deterministic hoàn toàn miễn phí ($0).
2. **Ở giai đoạn Querying (mỗi câu hỏi)**: Input tokens của GraphRAG cao gấp ~8.09 lần (5,633 so với 696 tokens) khiến chi phí mỗi câu tăng gấp 6.2 lần. Lý do là prompt của GraphRAG ngoài top-3 chunks vector thông thường còn được nhồi thêm danh sách dữ kiện đa bước mở rộng từ đồ thị (`facts`). Tuy nhiên, do facts đã cấu trúc hóa sẵn thông tin Điều/Khoản luật, thời gian sinh câu trả lời của LLM lại nhanh hơn (2.03s so với 4.49s).

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa tiền chất nằm trọn trong 1 chunk của Luật PCMT 2021 nên Flat RAG tìm kiếm vector là đủ để trả lời chính xác. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Thông tin 2 bị cáo tử hình nằm gọn trong 1 bài báo xét xử đường dây 36kg ma túy, vector search lấy trúng chunk nên cả 2 đều đạt điểm tuyệt đối. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Flat RAG chỉ tìm thấy chunk tin tức về mức án 36 tháng mà thiếu chunk Điều 251 BLHS, trong khi GraphRAG đi qua node `Crime` nối trực tiếp sang Điều 251 khoản 1. |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | Graph | Flat RAG hoàn toàn không biết Điều luật và khung hình phạt, còn GraphRAG dẫn được Điều 255 khoản 1 (dù chưa lấy được khung tối đa ở khoản 4). |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | Câu hỏi đòi hỏi xâu chuỗi người (Huy) → chất (MDMA >9,6kg) → đối chiếu khung hình phạt khoản 4 Điều 250 (tử hình); chỉ có GraphRAG làm được trọn vẹn. |
| Q6 | aggregation | 0.00 / 1 | 1.00 / 2 | Graph | Câu hỏi tổng hợp đòi hỏi gom tất cả vụ án dính tới MDMA; Flat RAG chỉ lấy được 3 mẩu tin vụn không đủ tên người/vụ, còn GraphRAG gom đủ 4 vụ án nhờ quan hệ `INVOLVES` với node `MDMA`. |

**Quy luật tổng quát rút ra**:
- Với câu hỏi **đơn nguồn (single-hop)**: Flat RAG và GraphRAG có chất lượng tương đương nhau (100% recall, judge 2/2). Thêm đồ thị không giúp tăng độ chính xác nhưng làm tăng chi phí token.
- Với câu hỏi **xuyên nguồn (cross-kb) và tổng hợp (aggregation)**: Flat RAG thất bại nặng nề (recall chỉ từ 0.00 đến 0.40) vì không có chunk văn bản nào chứa đủ cả 2 đầu thông tin. GraphRAG vượt trội hoàn toàn (recall đạt 1.00 ở hầu hết các câu) nhờ khả năng duyệt đồ thị theo node cầu nối.

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật (Missing Legal Context on Maximum Penalty)

- **Hiện tượng:** Ở câu Q4 (hỏi về mức phạt tù tối đa của giang hồ "Hoàng Nato" về hành vi tổ chức sử dụng trái phép chất ma túy), GraphRAG chỉ trả lời được mức phạt tối đa của khoản 1 (đến 07 năm), không nêu được mức phạt cao nhất của Điều 255 BLHS (tù chung thân / 20 năm).
- **Bằng chứng:**
  - Trích nguyên văn câu trả lời Q4 của GraphRAG trong `ket_qua_benchmark_kg.txt`:
    > *"Theo Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy, hành vi này có mức phạt tù tối đa quy định tại khoản 1 là đến 07 năm (khung hình phạt là từ 02 năm đến 07 năm). Ngữ cảnh không cung cấp thông tin về các khoản nặng hơn của Điều 255."*
  - Kiểm tra bằng Cypher các khoản của Điều 255 BLHS trong Neo4j:
    ```cypher
    MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
    RETURN cl.number, cl.penalty ORDER BY cl.number;
    ```
  - Kết quả trả về thực tế:
    ```
    khoản 1: phạt tù từ 02 năm đến 07 năm
    khoản 2: phạt tù từ 07 năm đến 15 năm
    khoản 3: phạt tù từ 15 năm đến 20 năm
    khoản 4: phạt tù 20 năm hoặc tù chung thân
    ```
- **Nguyên nhân:** Lỗi nằm ở bước **Cypher retrieval trong KG-3 (`Neo4jGraph.context`)**: Quy tắc lọc khoản luật hiện tại được viết là chỉ lấy khoản 1 (`cl.number = 1`) và các khoản `MENTIONS` chất mà vụ án đó `INVOLVES`. Do vụ Hoàng Nato là tin bắt giữ về hành vi tổ chức sử dụng chứ chưa giám định khối lượng cụ thể gắn với khoản tăng nặng, nên Cypher chỉ trả về duy nhất khoản 1 của Điều 255. Do đó LLM bị thiếu ngữ cảnh về các khoản 2, 3, 4.
- **Đề xuất sửa:** Trong hàm `context()` tại `src/graph.py`, nếu câu hỏi có chứa các từ khóa hỏi về mức cao nhất ("tối đa", "cao nhất", "khung hình phạt cao nhất"), Cypher cần bổ sung truy vấn lấy thêm khoản có `cl.number` lớn nhất hoặc khoản có `cl.penalty CONTAINS 'chung thân' OR cl.penalty CONTAINS 'tử hình'`.
  - *Đánh đổi:* Làm câu truy vấn Cypher phức tạp hơn và prompt dài thêm ~100 tokens, nhưng giải quyết triệt để các câu hỏi về mức án kịch khung.

---

### Lỗi E4: Phép đo sai / Thiếu sót của metric so khớp từ khóa cơ học (Evaluation Flaw)

- **Hiện tượng:** Ở câu Q6 (aggregation), Flat RAG bị chấm `recall = 0.00` nhưng điểm `judge = 1`, trong khi đọc kỹ câu trả lời thì Flat RAG đã tìm đúng được cả 3 vụ việc liên quan đến MDMA.
- **Bằng chứng:**
  - Trích nguyên văn `ket_qua_benchmark_kg.txt`:
    ```
    --- Q6 [aggregation] flat recall=0.00 judge=1 1.99s
    Dựa trên ngữ cảnh, cả 3 vụ việc đều liên quan đến ma túy MDMA:
    * Vụ việc [1]: Lực lượng chức năng phát hiện bên trong thùng hàng có các viên nén màu xanh là MDMA (khối lượng gần 4,3kg).
    * Vụ việc [2]: Công an bắt quả tang Thành mang 5 viên nén màu trắng đi bán, kết quả giám định xác định đây là ma túy MDMA.
    * Vụ việc [3]: Kết quả giám định xác định số viên nén hình tam giác màu hồng - xám trong kiện hàng là MDMA (khối lượng hơn 5,3kg).
    ```
  - Từ khóa bắt buộc trong `data/benchmark_kg.json`:
    `"must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]`
- **Nguyên nhân:** Lỗi nằm ở **chính phép đo (`keyword_recall`)**: Hàm `keyword_recall` chỉ kiểm tra sự xuất hiện của chuỗi con chính xác. Các chunk vector tìm được chỉ trích đoạn ngắn nội dung bài báo, trong đó viết tắt là *"Thành mang 5 viên nén"* (thay vì "Lê Minh Thành") và mô tả hành vi thùng hàng qua Nội Bài (mà chưa nhắc tên đầy đủ "Cái Quang Huy"). Flat RAG trả lời đúng bản chất 3 vụ việc nhưng không chứa đúng 3 chuỗi họ tên đầy đủ, dẫn tới `recall` bị gán 0.00 một cách máy móc, trong khi LLM-as-judge nhận thấy đúng một phần nên cho 1/2 điểm.
- **Đề xuất sửa:**
  1. Trong benchmark: Bổ sung các alias hợp lệ vào `must_include` (ví dụ: `["Lê Minh Thành" hoặc "Thành"]`).
  2. Dùng điểm của LLM-as-judge làm trọng số chính thay vì phụ thuộc hoàn toàn vào string matching đối với các câu hỏi mang tính tổng hợp (aggregation).
  - *Đánh đổi:* Đánh giá bằng LLM tốn thêm chi phí gọi API và thời gian đánh giá, nhưng phản ánh trung thực hơn năng lực trích xuất của hệ thống.

---

## 4. Kết luận (5 điểm)

Từ số liệu thực nghiệm thu được:
1. **Khi nào Flat RAG là đủ:**
   - Với các bài toán hỏi đáp đơn nguồn (single-hop) mà câu trả lời nằm trọn trong một văn bản (như Q1 luật, Q2 tin tức), Flat RAG đạt **recall 1.00** và **judge 2/2** tương đương GraphRAG, nhưng có chi phí rẻ hơn **6.2 lần** ($0.00010 vs $0.00062) và không tốn chi phí dựng đồ thị tri thức ban đầu ($0.00580). Nếu ứng dụng chỉ phục vụ tra cứu văn bản cục bộ, Flat RAG là lựa chọn tối ưu về chi phí và độ phức tạp.
2. **Khi nào BẮT BUỘC dùng Knowledge Graph (GraphRAG):**
   - Khi bài toán yêu cầu **liên kết đa nguồn dữ liệu** (ở đây là Luật và Tin tức thực tế) hoặc **tổng hợp thông tin trên diện rộng** (aggregation).
   - Minh chứng: Flat RAG tụt recall xuống chỉ còn **0.33 – 0.40** ở các câu cross-kb và **0.00** ở câu aggregation, trong khi GraphRAG đạt **0.94 recall trung bình** (tăng 84%) và **1.83/2 điểm judge**.
   - Mức chi phí đầu tư ban đầu ($0.00580 để dựng graph) và phụ phí mỗi câu hỏi (~$0.00052) là **hoàn toàn xứng đáng** để giải quyết được bài toán mà Flat RAG bất khả thi.

---

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.11s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00054. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j đã chụp và lưu đúng vị trí:
- `report/img/kg_count.png`: Đếm số lượng node theo từng nhãn (Clause: 99, Person: 43, Article: 18, Case: 14, Crime: 13, Substance: 12, Location: 6).
- `report/img/kg_cross_kb.png`: Trực quan hóa đường đi xuyên 2 KB qua node cầu nối Crime (32 nodes, 34 relationships).
- `report/img/kg_my_case.png`: Trực quan hóa toàn bộ đường đi người → vụ → tội → luật.
**Người đã chọn cho `kg_my_case.png`:** `Cái Quang Huy` (vụ án vận chuyển ma túy MDMA qua sân bay Nội Bài).

---

## Vấn đề gặp phải (không tính điểm)

- **Lỗi 1:** Ban đầu gặp lỗi rate limit `429 credit_balance_exhausted` trên OpenAI do tài khoản hết hạn mức. Đã khắc phục bằng cách chuyển sang provider Gemini thông qua endpoint OpenAI-compatible (`gemini-3.5-flash-lite` và `gemini-embedding-001`).
- **Lỗi 2:** Do cơ chế Free Tier của Google AI Studio giới hạn 100 RPM cho embedding và 15 RPM cho chat, hệ thống đã được bổ sung cơ chế retry tự động với exponential backoff trong `src/llm.py` để chạy trơn tru mà không bị crash giữa chừng.

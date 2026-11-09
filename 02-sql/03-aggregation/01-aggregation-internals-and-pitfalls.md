# SQL Aggregation: Bản chất, Cơ chế thực thi & Cạm bẫy trong Data Engineering

> **Tập trung vào:** Tư duy tổng hợp dữ liệu quy mô lớn, thứ tự thực thi logic, sự khác biệt giữa WHERE và HAVING, và cơ chế tính toán bên dưới Database Engine.

---

## 1. Bản chất & Thứ tự Thực thi Logic (Logical Execution Order)

Trong các câu truy vấn có tổng hợp dữ liệu, thứ tự xử lý của Database Engine quyết định trực tiếp câu lệnh có hợp lệ và tối ưu hay không:

```text
1. FROM / JOIN     --> Ghép nối các bảng nguồn
2. WHERE           --> Lọc từng dòng bản ghi thô (Row-level filter)
3. GROUP BY        --> Gom các dòng có cùng khóa nhóm vào một bucket
4. HAVING          --> Lọc trên kết quả đã gom nhóm (Aggregated filter)
5. SELECT          --> Chiếu các cột khóa và tính toán các hàm tổng hợp
6. DISTINCT        --> Khử trùng lặp
7. ORDER BY        --> Sắp xếp kết quả cuối cùng
8. LIMIT / OFFSET  --> Cắt số dòng hiển thị
```

### 💡 Bài học rút ra cho Data Engineer:
* **`WHERE` chạy trước `GROUP BY`:** Luôn lọc dữ liệu tối đa ở `WHERE` (ví dụ: `WHERE status = 'SUCCESS'`) để loại bỏ các dòng không cần thiết trước khi hệ thống phải phân bổ bộ nhớ để gom nhóm.
* **`HAVING` chạy sau `GROUP BY`:** Chỉ dùng `HAVING` khi điều kiện lọc phụ thuộc vào giá trị tính toán của hàm tổng hợp (ví dụ: `HAVING SUM(balance) > 100000000`).
* **Tuyệt đối không dùng `HAVING` để lọc các cột không tổng hợp** (ví dụ viết `HAVING status = 'SUCCESS'` là anti-pattern nghiêm trọng, làm database phải gom nhóm toàn bộ rồi mới lọc bỏ).

---

## 2. Bên dưới Engine: Hash Aggregate vs Stream/Sort Aggregate

Khi chạy câu lệnh `GROUP BY`, Database Engine (Postgres, DuckDB, Spark, ClickHouse) xử lý theo 2 cơ chế chính:

```text
1. Hash Aggregate (Phổ biến nhất):
   Engine tạo một bảng băm (Hash Table) trong bộ nhớ RAM (WorkMem). 
   Mỗi giá trị Group Key sẽ được hash để tìm bucket tương ứng và cộng dồn các giá trị SUM, COUNT.
   => Ưu điểm: Cực nhanh với dữ liệu chưa sắp xếp.
   => Nhược điểm: Nếu số lượng nhóm phân biệt (Cardinality) quá lớn (hàng chục triệu khách hàng), 
      Hash Table sẽ tràn RAM và phải ghi xuống đĩa (Spill to Disk), làm giảm tốc độ nghiêm trọng.

2. Stream / Sort Aggregate:
   Dữ liệu được sắp xếp sẵn theo Group Key trước, sau đó Engine chỉ việc quét tuần tự (streaming) 
   từ trên xuống và cộng dồn khi nào gặp Key mới thì xuất kết quả.
   => Tốt nhất khi: Dữ liệu đã có Clustered Index hoặc đã được SORT/PARTITION sẵn trên đĩa.
```

---

## 3. Top 4 Cạm bẫy "xương máu" khi viết Aggregation trong DE

### ⚠️ Cạm bẫy 1: Sự khác biệt nguy hiểm của `COUNT(*)` vs `COUNT(column)`
* **`COUNT(*)`**: Đếm toàn bộ số dòng vật lý trong nhóm, **kể cả dòng chứa toàn giá trị NULL**.
* **`COUNT(column_name)`**: Chỉ đếm những dòng mà tại đó `column_name IS NOT NULL`.
* **Kinh nghiệm:**
  * Khi đếm số lượng giao dịch: Dùng `COUNT(transaction_id)`.
  * Khi đếm có điều kiện: Dùng `COUNT(CASE WHEN status = 'SUCCESS' THEN 1 END)`. Tránh viết `COUNT(CASE WHEN status = 'SUCCESS' THEN 1 ELSE 0 END)` vì giá trị `0` vẫn được `COUNT` tính là NOT NULL!

---

### ⚠️ Cạm bẫy 2: Hàm `AVG()` tự động nuốt chửng `NULL` làm sai lệch mẫu số
* **Hiện tượng:** Hàm `AVG(col)` tính bằng: $\frac{\sum \text{giá trị khác NULL}}{\text{số dòng khác NULL}}$.  
  Nếu 10 khách hàng có 5 người gửi 10 triệu và 5 người chưa gửi tiền (mang giá trị `NULL`):
  * `AVG(amount)` sẽ ra: $50 \text{ triệu} / 5 = 10 \text{ triệu}$ (Sai lệch nghiêm trọng bản chất trung bình toàn bộ khách hàng!).
* **Giải pháp:** Nếu muốn tính trung bình trên toàn bộ mẫu, bắt buộc bọc `COALESCE`:
  ```sql
  AVG(COALESCE(amount, 0))
  ```

---

### ⚠️ Cạm bẫy 3: Phép chia cho 0 (Division by Zero) trong các chỉ số phần trăm
* **Hiện tượng:** Trong các bài toán tính tỷ lệ như `Conversion Rate` hay `ROI` (Bài 9), nếu mẫu số (`total_reach` hoặc `budget`) bằng 0, câu lệnh SQL sẽ bị văng lỗi và dừng toàn bộ pipeline.
* **Giải pháp an toàn:** Luôn bọc mẫu số bằng hàm `NULLIF(col, 0)`:
  ```sql
  -- Nếu total_reach = 0, NULLIF trả về NULL -> Phép chia trả về NULL thay vì văng Exception
  ROUND(total_converted * 100.0 / NULLIF(total_reach, 0), 2) AS conversion_rate_pct
  ```

---

### ⚠️ Cạm bẫy 4: High Cardinality trong `COUNT(DISTINCT)` trên Big Data
* **Hiện tượng:** Phép tính `COUNT(DISTINCT device_id)` (Bài 10) yêu cầu Engine phải lưu giữ toàn bộ danh sách các ID duy nhất vào bộ nhớ để khử trùng lặp. Khi bảng lên hàng trăm triệu bản ghi, phép tính này cực kỳ tốn CPU và RAM.
* **Giải pháp trong Data Engineering hiện đại:**
  * Với các dashboard phân tích xu hướng lớn, các hệ thống Big Data (Presto, Trino, BigQuery, Snowflake) hỗ trợ hàm xấp xỉ **`APPROX_COUNT_DISTINCT()`** (dựa trên thuật toán HyperLogLog) cho tốc độ nhanh gấp 10-50 lần với sai số dưới 1-2%.

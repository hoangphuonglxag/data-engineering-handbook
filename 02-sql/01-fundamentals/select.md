# SQL Fundamentals: SELECT & Column Projection

> **Mục tiêu DE**: Hiểu rõ bản chất của phép chiếu cột (Projection), thứ tự thực thi logic của câu lệnh SQL, và tác động sống còn của việc lựa chọn cột đối với hiệu năng hệ thống Data Warehouse / Big Data.

---

## 1. Bản chất & Logical Execution Order

Trong SQL, thứ tự viết câu lệnh (Lexical Order) **hoàn toàn khác** với thứ tự thực thi bên dưới engine (Logical Processing Order):

```text
Thứ tự viết (Lexical)          Thứ tự thực thi thực tế (Logical)
1. SELECT                      1. FROM (chọn & join bảng)
2. FROM                        2. WHERE (lọc dòng)
3. WHERE                       3. GROUP BY (gom nhóm)
4. GROUP BY                    4. HAVING (lọc nhóm)
5. HAVING                      5. SELECT (chiếu cột & tính toán)  <-- Ở ĐÂY!
6. ORDER BY                    6. DISTINCT (khử trùng)
7. LIMIT / OFFSET              7. ORDER BY (sắp xếp)
                               8. LIMIT / OFFSET (cắt dòng)
```

### 💡 Bài học rút ra cho DE:
* Mệnh đề `SELECT` được tính toán **rất trễ** (sau `FROM`, `WHERE`, `GROUP BY`, `HAVING`).
* **Không thể dùng bí danh (Alias)** định nghĩa ở `SELECT` trong mệnh đề `WHERE` (vì lúc chạy `WHERE`, `SELECT` chưa hề được đánh giá).
* Trong `ORDER BY`, bạn **được phép dùng Alias** vì `ORDER BY` chạy sau `SELECT`.

---

## 2. DE Pitfalls & Best Practices

### ❌ Anti-pattern: `SELECT *` trong Data Engineering
Trong môi trường OLTP (ứng dụng nhỏ), `SELECT *` chỉ gây chậm nhẹ về mạng. Nhưng trong Data Engineering (Big Data & Data Warehouse), `SELECT *` là **tội đồ**:

1. **Columnar Storage (Parquet, ORC, BigQuery, Snowflake, DuckDB):**
   * Các hệ thống DWH lưu dữ liệu theo cột. Nếu bạn chỉ cần 3 cột nhưng viết `SELECT *` trên bảng 100 cột, hệ thống phải đọc I/O từ đĩa gấp **30 lần**.
   * Trên BigQuery: Chi phí tính theo số Byte quét qua. `SELECT *` trên bảng 1 TB tốn \$6.25; chỉ chọn 2 cột cần thiết có thể chỉ tốn \$0.10.
2. **Schema Drift / Pipeline Breakage:**
   * Khi bảng nguồn thêm, sửa hoặc xóa cột, các câu lệnh ETL phụ thuộc vào `SELECT *` sẽ bị lệch cấu trúc (Schema Mismatch), làm sập các job downstream.
3. **Network I/O & Memory Pressure:**
   * Truyền tải dữ liệu qua network giữa Database $\rightarrow$ Spark/Python gây nghẽn băng thông và dễ dẫn đến lỗi `OOM (Out Of Memory)`.

### ✅ Best Practices:
* **Explicit Projection:** Luôn liệt kê rõ ràng tên các cột cần lấy: `SELECT col1, col2, col3`.
* **Column Renaming & Casting:** Chuẩn hóa tên cột (snake_case) và kiểu dữ liệu ngay tại tầng trích xuất (ví dụ `CAST(amount AS DECIMAL(15, 2)) AS amount_vnd`).
* **Computed Columns & Sanitization:** Xử lý làm sạch sơ bộ (dùng `COALESCE`, `NULLIF`, `TRIM`, `LOWER`).

---

## 3. Real Business Problem (Mock Banking Dataset)

**Bối cảnh:**  
Bạn nhận được yêu cầu từ đội Compliance (Tuân thủ) ngân hàng:  
Trích xuất danh sách khách hàng cá nhân từ bảng `core_customers` để phục vụ audit, yêu cầu:
1. Chỉ lấy các trường cần thiết: Mã KH, họ tên, email, CCCD, ngày sinh, trạng thái KYC, rủi ro.
2. Tên cột kết quả phải viết theo chuẩn audit (`customer_code`, `full_name_clean`, `masked_cccd`, `audit_risk_level`).
3. Chuẩn hóa họ tên: Xóa khoảng trắng thừa.
4. Bảo mật dữ liệu (Data Masking cơ bản): Che 6 số đầu của CCCD, chỉ hiển thị 6 số cuối (dạng `******123456`). Nếu không có CCCD thì trả về `'NO_CCCD'`.

---

## 4. Solution & Phân tích

```sql
-- Solution: Chuẩn hóa và chiếu cột an toàn cho Audit
SELECT 
    customer_id AS customer_code,
    TRIM(full_name) AS full_name_clean,
    email,
    -- Data masking cho CCCD
    CASE 
        WHEN cccd IS NULL OR LENGTH(TRIM(cccd)) < 6 THEN 'NO_CCCD'
        ELSE CONCAT('******', RIGHT(TRIM(cccd), 6))
    END AS masked_cccd,
    dob AS date_of_birth,
    kyc_status,
    COALESCE(risk_level, 'UNKNOWN') AS audit_risk_level
FROM core_customers
LIMIT 100;
```

### 🔍 Điểm cần chú ý:
1. **`COALESCE(risk_level, 'UNKNOWN')`**: Tránh việc giá trị `NULL` làm sai lệch logic phân loại sau này.
2. **`TRIM()`**: Dữ liệu thực tế thường xuyên dính khoảng trắng đầu/cuối do người dùng hoặc lỗi hệ thống web/app.
3. **Masking logic**: Không bao giờ để lộ PII (Personally Identifiable Information) như CCCD đầy đủ ra báo cáo nếu không có quyền hạn.

---

## 5. Lab thực hành

* File script SQL thực thi: [01_select.sql](file:///E:/PJ/11_12/data-engineering-practice/sql/01-fundamentals/01_select.sql)
* Chạy trực tiếp trên dataset: `data-engineering-practice/datasets/mock-banking/core_customers.csv`

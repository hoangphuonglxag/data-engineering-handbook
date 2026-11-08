# SQL Joins: Bản chất, Cơ chế thực thi & Cạm bẫy trong Data Engineering

> **Tập trung vào:** Tư duy xử lý dữ liệu lớn, các thuật toán Join bên dưới Database Engine, và những kinh nghiệm "xương máu" rút ra từ bài toán Banking thực tế.

---

## 1. Bản chất các loại JOIN & Triết lý của DE

| Loại JOIN | Bản chất đại số quan hệ | Ứng dụng thực tế trong Data Engineering |
| :--- | :--- | :--- |
| **INNER JOIN** | Giao của hai tập hợp ($A \cap B$) | Khi muốn trích xuất dữ liệu hoàn chỉnh, bắt buộc tồn tại ở cả hai chiều (ví dụ: giao dịch phát sinh từ tài khoản hợp lệ). |
| **LEFT JOIN** | Giữ trọn vẹn tập trái ($A$), ghép nếu có bên phải | **Loại JOIN phổ biến nhất trong DE**. Dùng khi enrich dữ liệu (bảng Fact nối sang Dim), xây dựng báo cáo tổng thể mà không làm mất khách hàng/giao dịch. |
| **FULL OUTER JOIN** | Hợp của hai tập hợp ($A \cup B$) | **Đặc sản của bài toán Đối soát (Reconciliation)**: Phát hiện giao dịch có ở kênh A nhưng mất ở kênh B hoặc ngược lại. |
| **CROSS JOIN** | Tích Descartes ($A \times B$) | Sinh dữ liệu mô phỏng, ma trận ngày tháng $\times$ sản phẩm để lấp đầy ngày không có giao dịch (Date Spine). |

> 💡 **Kinh nghiệm 1: Tại sao Data Engineer tránh dùng `RIGHT JOIN`?**  
> Trong pipeline sản xuất, câu lệnh SQL luôn được đọc xuôi từ trên xuống dưới, từ trái sang phải: `FROM <Bảng gốc> LEFT JOIN <Bảng bổ sung>`. Sử dụng `RIGHT JOIN` làm đảo lộn luồng tư duy, tăng gánh nặng đọc hiểu (cognitive load) và rất dễ gây nhầm lẫn khi review code pipeline.

---

## 2. Bên dưới Database Engine: 3 thuật toán JOIN vật lý

Hiểu cơ chế engine thực thi giúp bạn tối ưu hóa câu lệnh khi dữ liệu phình to lên hàng triệu hay hàng tỷ dòng:

```text
1. Nested Loop Join:
   Duyệt từng dòng bảng ngoài, quét tìm trong bảng trong.
   => Tốt nhất khi: Bảng ngoài rất nhỏ và bảng trong có INDEX B-Tree trên khóa JOIN.

2. Hash Join (Phổ biến nhất trong DWH / OLAP):
   Tạo Hash Table từ bảng nhỏ trên RAM -> Quét bảng lớn để so sánh hash key.
   => Tốt nhất khi: Bảng lớn, chưa sắp xếp, quan hệ phép bằng (=).

3. Sort-Merge Join:
   Sắp xếp cả 2 bảng theo khóa JOIN -> Quét song song cả 2 bảng cùng lúc.
   => Tốt nhất khi: Cả hai tập dữ liệu đều cực lớn hoặc đã được sort/partition sẵn trên đĩa.
```

---

## 3. Top 5 Cạm bẫy "xương máu" rút ra từ thực tế

### ⚠️ Cạm bẫy 1: Fan-Out & Data Duplication (Nhân bản số dòng)
* **Hiện tượng:** Khi JOIN bảng cha (1) với hai bảng con (N) riêng biệt. Ví dụ: 1 khách hàng có 3 tài khoản và 2 thiết bị mobile. Nếu viết:
  ```sql
  -- ❌ SAI: Nhân bản dòng (Cartesian cục bộ) -> 3 tài khoản x 2 thiết bị = 6 dòng!
  SELECT c.customer_id, SUM(a.balance), COUNT(d.device_id)
  FROM core_customers c
  LEFT JOIN core_accounts a ON c.customer_id = a.customer_id
  LEFT JOIN mb_devices d ON c.customer_id = d.customer_id
  GROUP BY c.customer_id;
  ```
  Hàm `SUM(balance)` sẽ bị nhân đôi lên, làm sai lệch báo cáo tài chính!
* **Kinh nghiệm giải quyết (Pre-aggregation via CTE):**
  Luôn thu gọn dữ liệu ở từng nhánh con về đúng grain `customer_id` trước khi ghép nối vào bảng chính (như đã áp dụng ở Bài 1, Bài 2, Bài 3).

---

### ⚠️ Cạm bẫy 2: Vô tình biến `LEFT JOIN` thành `INNER JOIN` bởi mệnh đề `WHERE`
* **Hiện tượng:**
  ```sql
  -- ❌ SAI: Khách hàng chưa có tài khoản sẽ có a.status là NULL.
  -- Nhưng WHERE a.status = 'ACTIVE' sẽ loại bỏ toàn bộ NULL -> Mất sạch khách hàng mới!
  SELECT c.customer_id, c.full_name, a.balance
  FROM core_customers c
  LEFT JOIN core_accounts a ON c.customer_id = a.customer_id
  WHERE a.status = 'ACTIVE';
  ```
* **Kinh nghiệm giải quyết:**
  Chuyển điều kiện lọc vào mệnh đề `ON` hoặc lọc trước bên trong CTE:
  ```sql
  -- ✅ ĐÚNG: Lọc ngay khi ghép bảng
  FROM core_customers c
  LEFT JOIN core_accounts a 
      ON c.customer_id = a.customer_id AND a.status = 'ACTIVE'
  ```

---

### ⚠️ Cạm bẫy 3: Bất đồng bộ định dạng khóa giữa các hệ sinh thái (Key Transformation)
* **Thực tế:** Các hệ thống trong doanh nghiệp thường do các vendor hoặc team khác nhau phát triển.
  * CRM System lưu mã nhân viên: `RM_0090`.
  * Core HR System lưu mã nhân viên: `EMP000090`.
* **Kinh nghiệm giải quyết:**
  Cần làm sạch và chuẩn hóa khóa định danh ngay tại tầng trích xuất bằng các hàm chuỗi (`SUBSTRING`, `LPAD`, `CONCAT`) hoặc hàm ép kiểu trước khi thực hiện phép nối (như đã xử lý trong Bài 4).

---

### ⚠️ Cạm bẫy 4: Nguy cơ từ giá trị `NULL` sau `LEFT JOIN`
* **Hiện tượng:** Khi không có dữ liệu bên bảng phải, các cột sẽ trả về `NULL`. Nếu đưa thẳng `NULL` vào phép tính toán hoặc xuất ra API downstream, hệ thống có thể bị crash (NPE) hoặc sai số.
* **Kinh nghiệm giải quyết:**
  * Luôn bọc hàm `COALESCE(numeric_col, 0)` cho các chỉ số đếm hoặc số dư tiền tệ.
  * Phân biệt rõ:
    * `COUNT(*)`: Đếm số bản ghi (kể cả dòng toàn NULL do LEFT JOIN sinh ra).
    * `COUNT(column_name)`: Chỉ đếm những giá trị thực sự khác NULL.
    * `COUNT(CASE WHEN status = 'ACTIVE' THEN 1 END)`: Đếm có điều kiện an toàn.

---

### ⚠️ Cạm bẫy 5: Thứ tự đánh giá điều kiện trong kiểm toán & đối soát (Reconciliation)
* **Thực tế:** Khi đối soát giữa 2 nguồn (Core vs Digital Gateway), logic phân loại trạng thái phải tuân thủ nghiêm ngặt theo thứ tự loại trừ:
  1. Kiểm tra tồn tại trước (`c.transaction_id IS NULL` $\rightarrow$ `NOT_FOUND`).
  2. Kiểm tra tính toàn vẹn số tiền (`t.amount != c.amount` $\rightarrow$ `AMOUNT_MISMATCH`).
  3. Kiểm tra tính đồng bộ trạng thái (`t.status != c.status` $\rightarrow$ `STATUS_MISMATCH`).
  4. Còn lại mới xác nhận là `MATCHED`.
* Việc đảo lộn thứ tự `CASE WHEN` sẽ dẫn đến sót lỗi kiểm toán nghiêm trọng.

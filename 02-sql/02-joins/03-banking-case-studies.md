Bài 1: Customer 360 – Tổng hợp hồ sơ khách hàng
Business: Customer Analytics Basic
File: core_customers + crm_contacts + core_accounts

Bối cảnh Business:
Bộ phận Customer Experience muốn xây dựng báo cáo Customer 360 để biết:
- Khách hàng thuộc phân khúc nào (segment).
- Đang sở hữu bao nhiêu tài khoản (total_accounts).
- Bao nhiêu tài khoản đang hoạt động (active_accounts).
- Tổng số dư hiện tại là bao nhiêu (total_balance).

Yêu cầu:
- JOIN 3 bảng: core_customers, crm_contacts, core_accounts.
- Lấy toàn bộ khách hàng, bao gồm cả khách hàng chưa có tài khoản hoặc chưa có CRM contact.
- Hiển thị customer_id, full_name, segment, total_accounts, active_accounts, total_balance.

Kiến thức:
- LEFT JOIN nhiều bảng (Multiple LEFT JOINs).
- Pre-aggregation bằng CTE để chống lỗi Fan-out / nhân bản số dòng (1:N relationship).
- Xử lý NULL với COALESCE và COUNT CASE WHEN.
======

WITH acc_summary AS (
    SELECT 
        customer_id,
        COUNT(account_id) AS total_accounts,
        COUNT(CASE WHEN status = 'ACTIVE' THEN 1 END) AS active_accounts,
        COALESCE(SUM(balance), 0) AS total_balance
    FROM core_accounts
    GROUP BY customer_id
)
SELECT 
    c.customer_id,
    c.full_name,
    crm.segment,
    COALESCE(a.total_accounts, 0) AS total_accounts,
    COALESCE(a.active_accounts, 0) AS active_accounts,
    COALESCE(a.total_balance, 0) AS total_balance
FROM core_customers AS c
LEFT JOIN crm_contacts AS crm 
    ON c.customer_id = crm.customer_id
LEFT JOIN acc_summary AS a 
    ON c.customer_id = a.customer_id
ORDER BY c.customer_id
LIMIT 10;
======

Bài 2. Branch Performance – Đánh giá hiệu quả chi nhánh
Business: Branch Operations Basic
File: core_branches + core_customers + core_accounts

Bối cảnh Business:
Ban điều hành muốn biết chi nhánh nào có hiệu quả hoạt động cao dựa trên:
- Số lượng khách hàng trực thuộc chi nhánh (total_customers).
- Số lượng tài khoản đang hoạt động (active_accounts).
- Tổng số dư tài khoản hoạt động (total_balance).

Yêu cầu:
- JOIN 3 bảng: core_branches, core_customers, core_accounts.
- Chỉ tính tài khoản có status = 'ACTIVE'.
- Bao gồm cả chi nhánh chưa có khách hàng hoặc chưa có tài khoản phát sinh.
- Hiển thị: branch_id, branch_name, total_customers, active_accounts, total_balance.

Kiến thức:
- Multi-table LEFT JOIN.
- Pre-aggregation độc lập từng chiều (Branch Customers vs Branch Accounts) bằng CTE để tránh Cartesian product / Fan-out giữa hai bảng con.
- COALESCE xử lý chi nhánh mới chưa có dữ liệu.
======

WITH branch_customers AS (
    SELECT 
        branch_id,
        COUNT(DISTINCT customer_id) AS total_customers
    FROM core_customers
    GROUP BY branch_id
),
branch_accounts AS (
    SELECT 
        branch_id,
        COUNT(account_id) AS active_accounts,
        COALESCE(SUM(balance), 0) AS total_balance
    FROM core_accounts
    WHERE status = 'ACTIVE'
    GROUP BY branch_id
)
SELECT 
    b.branch_id,
    b.branch_name,
    COALESCE(c.total_customers, 0) AS total_customers,
    COALESCE(a.active_accounts, 0) AS active_accounts,
    COALESCE(a.total_balance, 0) AS total_balance
FROM core_branches AS b
LEFT JOIN branch_customers AS c 
    ON b.branch_id = c.branch_id
LEFT JOIN branch_accounts AS a 
    ON b.branch_id = a.branch_id
ORDER BY total_balance DESC
LIMIT 10;
======

Bài 3. Digital Banking Adoption – Khách hàng sử dụng ngân hàng số
Business: Digital Transformation Intermediate
File: core_customers + ib_users + mb_devices

Bối cảnh Business:
Ngân hàng muốn xác định khách hàng nào đã đăng ký Internet Banking (IB) và Mobile Banking (MB) nhằm đo tỷ lệ thâm nhập ngân hàng số.

Yêu cầu:
- JOIN: core_customers, ib_users, mb_devices.
- Trả về mỗi khách hàng đúng 1 dòng (grain: customer_id).
- Hiển thị: customer_id, full_name, has_ib ('YES'/'NO'), has_mb ('YES'/'NO'), device_count (số thiết bị mobile đã đăng ký).

Kiến thức:
- LEFT JOIN nhiều hệ thống phân tán (Core, Internet Banking, Mobile Banking).
- Conditional Aggregation và logic CASE WHEN phân loại trạng thái.
- Fan-out prevention khi 1 khách hàng có thể đăng ký nhiều thiết bị điện thoại.
======

WITH mb_summary AS (
    SELECT 
        customer_id,
        COUNT(device_id) AS device_count
    FROM mb_devices
    WHERE status = 'ACTIVE'
    GROUP BY customer_id
),
ib_summary AS (
    SELECT 
        customer_id,
        COUNT(ib_user_id) AS ib_count
    FROM ib_users
    WHERE status = 'ACTIVE'
    GROUP BY customer_id
)
SELECT 
    c.customer_id,
    c.full_name,
    CASE 
        WHEN COALESCE(ib.ib_count, 0) > 0 THEN 'YES' 
        ELSE 'NO' 
    END AS has_ib,
    CASE 
        WHEN COALESCE(mb.device_count, 0) > 0 THEN 'YES' 
        ELSE 'NO' 
    END AS has_mb,
    COALESCE(mb.device_count, 0) AS device_count
FROM core_customers AS c
LEFT JOIN ib_summary AS ib 
    ON c.customer_id = ib.customer_id
LEFT JOIN mb_summary AS mb 
    ON c.customer_id = mb.customer_id
ORDER BY c.customer_id
LIMIT 10;
======

Bài 4. Employee Performance – Hiệu quả chăm sóc khách hàng
Business: CRM Operations Intermediate
File: core_employees + crm_interactions + crm_contacts

Bối cảnh Business:
CRM Manager muốn đánh giá hiệu suất của nhân viên tư vấn/chăm sóc khách hàng (Relationship Manager / Call Center).

Yêu cầu:
- JOIN: core_employees, crm_interactions, crm_contacts.
- Tính toán theo từng nhân viên:
  - total_interactions: Tổng số lượt tương tác.
  - total_duration: Tổng thời lượng gọi/tư vấn (giây).
  - avg_satisfaction: Điểm đánh giá độ hài lòng trung bình (thang điểm 1-5).
  - customers_served: Số lượng khách hàng phân biệt đã được chăm sóc.
- Sắp xếp giảm dần theo số lượt tương tác.

Kiến thức:
- Chaining JOINs qua bảng trung gian (core_employees -> crm_interactions -> crm_contacts).
- Kỹ thuật Data Cleansing / Key Transformation khi mã ID giữa 2 hệ thống khác chuẩn (ví dụ tiền tố CRM `RM_0090` ánh xạ sang HR `EMP000090`).
- COUNT(DISTINCT) để đếm số khách hàng duy nhất.
======

SELECT 
    e.employee_id,
    e.full_name,
    COUNT(i.interaction_id) AS total_interactions,
    COALESCE(SUM(i.duration_seconds), 0) AS total_duration,
    ROUND(AVG(i.satisfaction_score), 2) AS avg_satisfaction,
    COUNT(DISTINCT c.customer_id) AS customers_served
FROM core_employees AS e
INNER JOIN crm_interactions AS i 
    ON e.employee_id = CONCAT('EMP', LPAD(SUBSTRING(i.agent_id, 4), 6, '0'))
INNER JOIN crm_contacts AS c 
    ON i.contact_id = c.contact_id
GROUP BY e.employee_id, e.full_name
ORDER BY total_interactions DESC
LIMIT 10;
======

Bài 5. Transaction Reconciliation – Đối soát giao dịch đa kênh
Business: Finance & Operations Advanced
File: ib_transfers + core_transactions

Bối cảnh Business:
Bộ phận Vận hành & Kế toán (Operations/Finance) cần đối soát tự động giữa lệnh chuyển tiền trên Internet Banking (`ib_transfers`) và bút toán ghi nhận thực tế trên Core Banking (`core_transactions`) thông qua mã tham chiếu `reference_code`.

Yêu cầu:
- LEFT JOIN từ `ib_transfers` sang `core_transactions` bằng `reference_code`.
- Phân loại trạng thái đối soát (`reconciliation_status`):
  - `MATCHED`: Khớp cả giao dịch, số tiền và trạng thái.
  - `AMOUNT_MISMATCH`: Tìm thấy giao dịch trong Core nhưng số tiền bị lệch.
  - `STATUS_MISMATCH`: Tìm thấy giao dịch nhưng trạng thái thực thi không khớp.
  - `NOT_FOUND`: Lệnh chuyển tiền trên IB không tìm thấy bút toán tương ứng trong Core Banking.
- Hiển thị: transfer_id, reference_code, ib_amount, core_amount, reconciliation_status.

Kiến thức:
- LEFT JOIN phát hiện dữ liệu bất đối xứng (Reconciliation / Data Discrepancy).
- Phân nhánh logic kiểm toán với CASE WHEN ưu tiên thứ tự điều kiện (Missing check -> Amount check -> Status check).
- Nghiệp vụ đối soát hệ thống thanh toán ngân hàng (Payment Reconciliation Pipeline).
======

SELECT 
    t.transfer_id,
    t.reference_code,
    t.amount AS ib_amount,
    c.amount AS core_amount,
    CASE 
        WHEN c.transaction_id IS NULL THEN 'NOT_FOUND'
        WHEN t.amount != c.amount THEN 'AMOUNT_MISMATCH'
        WHEN t.status != c.status THEN 'STATUS_MISMATCH'
        ELSE 'MATCHED'
    END AS reconciliation_status
FROM ib_transfers AS t
LEFT JOIN core_transactions AS c 
    ON t.reference_code = c.reference_code
ORDER BY t.transfer_id
LIMIT 15;
======

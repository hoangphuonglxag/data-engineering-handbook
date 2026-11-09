Bài 6: Monthly Transaction Report – Báo cáo giao dịch theo tháng
Business: Financial Operations & KPI Reporting
File: core_transactions

Bối cảnh Business:
Khối Vận hành Tài chính (Finance Department) cần theo dõi tổng giá trị và số lượng giao dịch thành công theo từng tháng để nắm bắt xu hướng tăng trưởng dòng tiền và tính toán giá trị trung bình trên mỗi giao dịch (Average Ticket Size).

Yêu cầu:
- Từ bảng core_transactions, lọc các giao dịch có status = 'SUCCESS'.
- Nhóm dữ liệu theo tháng giao dịch (định dạng YYYY-MM).
- Tính tổng số giao dịch thành công (total_transactions).
- Tính tổng giá trị giao dịch thành công (total_amount).
- Tính giá trị giao dịch trung bình (avg_amount, làm tròn 2 chữ số thập phân).
- Sắp xếp giảm dần theo tháng gần nhất.

Kiến thức:
- Định dạng chuỗi ngày tháng: STRFTIME(transaction_date, '%Y-%m') hoặc DATE_TRUNC.
- Hàm tổng hợp: COUNT(), SUM(), AVG(), ROUND().
- Thứ tự thực thi: WHERE lọc bỏ FAILED transaction trước khi đưa vào nhóm GROUP BY để tối ưu hiệu năng.
======

SELECT 
    STRFTIME(transaction_date, '%Y-%m') AS month,
    COUNT(transaction_id) AS total_transactions,
    SUM(amount) AS total_amount,
    ROUND(AVG(amount), 2) AS avg_amount
FROM core_transactions
WHERE status = 'SUCCESS'
GROUP BY STRFTIME(transaction_date, '%Y-%m')
ORDER BY month DESC
LIMIT 12;
======

Bài 7: Top Customer by Balance – Khách hàng có tổng tài sản lớn
Business: Private Banking & Wealth Management
File: core_customers + core_accounts

Bối cảnh Business:
Khối Khách hàng Ưu tiên (Private Banking) cần lọc danh sách những khách hàng có tổng tài sản hiện hữu (tổng số dư các tài khoản đang hoạt động) vượt ngưỡng 100 triệu VND để đưa vào danh sách chăm sóc đặc biệt.

Yêu cầu:
- JOIN giữa core_customers và core_accounts.
- Chỉ tính các tài khoản có status = 'ACTIVE'.
- Tính tổng số dư (total_balance) cho từng khách hàng.
- Sử dụng HAVING để lọc những khách hàng có tổng số dư > 100,000,000 VND.
- Sắp xếp giảm dần theo tổng tài sản.

Kiến thức:
- Phân biệt WHERE và HAVING: WHERE lọc các tài khoản ACTIVE trước khi gom nhóm; HAVING lọc trên kết quả đã tổng hợp SUM(balance) của khách hàng.
- Quan hệ 1:N giữa Customer và Account.
======

SELECT 
    c.customer_id,
    c.full_name,
    SUM(a.balance) AS total_balance
FROM core_customers AS c
INNER JOIN core_accounts AS a 
    ON c.customer_id = a.customer_id
WHERE a.status = 'ACTIVE'
GROUP BY c.customer_id, c.full_name
HAVING SUM(a.balance) > 100000000
ORDER BY total_balance DESC
LIMIT 15;
======

Bài 8: Branch Transaction Performance – Hiệu suất giao dịch theo chi nhánh
Business: Branch Network Performance
File: core_transactions + core_accounts + core_branches

Bối cảnh Business:
Ban Giám đốc Mạng lưới Chi nhánh muốn đánh giá hiệu quả kinh doanh của từng chi nhánh thông qua quy mô và giá trị giao dịch phát sinh từ các tài khoản do chi nhánh đó quản lý.

Yêu cầu:
- JOIN: core_transactions nối sang core_accounts (bằng from_account) rồi nối sang core_branches (bằng branch_id).
- Chỉ tính các giao dịch thành công (status = 'SUCCESS').
- Tính:
  - total_transactions: Tổng số lượng giao dịch thành công.
  - total_amount: Tổng giá trị dòng tiền chuyển đi.
  - avg_amount: Giá trị trung bình mỗi giao dịch.
- Gom nhóm theo branch_id, branch_name và sắp xếp giảm dần theo tổng giá trị.

Kiến thức:
- Multi-table Aggregation: Gom nhóm trên thuộc tính của bảng Dim (Branch) sau khi nối từ bảng Fact (Transactions).
- Tối ưu I/O: Lọc giao dịch thành công trước khi JOIN để giảm kích thước bảng dữ liệu trung gian.
======

SELECT 
    b.branch_id,
    b.branch_name,
    COUNT(t.transaction_id) AS total_transactions,
    SUM(t.amount) AS total_amount,
    ROUND(AVG(t.amount), 2) AS avg_amount
FROM core_transactions AS t
INNER JOIN core_accounts AS a 
    ON t.from_account = a.account_number
INNER JOIN core_branches AS b 
    ON a.branch_id = b.branch_id
WHERE t.status = 'SUCCESS'
GROUP BY b.branch_id, b.branch_name
ORDER BY total_amount DESC
LIMIT 15;
======

Bài 9: Campaign Performance – Đánh giá hiệu quả chiến dịch Marketing
Business: Marketing Campaign Analytics
File: crm_campaigns

Bối cảnh Business:
Giám đốc Marketing cần đo lường hiệu quả chuyển đổi và mức sinh lời thực tế của các chiến dịch đã kết thúc (`status = 'COMPLETED'`) nhằm phân bổ ngân sách cho quý tiếp theo.

Yêu cầu:
- Lấy dữ liệu từ bảng crm_campaigns với điều kiện status = 'COMPLETED'.
- Tính các chỉ số tài chính và hiệu quả:
  - net_return_vnd: Doanh thu tạo ra trừ đi ngân sách (revenue_generated - budget_vnd).
  - conversion_rate_pct: Tỷ lệ khách hàng chuyển đổi thành công trên tổng tiếp cận (total_converted * 100.0 / total_reach).
  - roi_pct: Tỷ suất sinh lời trên chi phí đầu tư ((revenue - budget) * 100.0 / budget).
- Sắp xếp giảm dần theo lợi nhuận ròng (net_return_vnd).

Kiến thức:
- Calculated Metrics & Phép chia an toàn: Dùng NULLIF(total_reach, 0) để phòng tránh lỗi Divide by Zero (ZeroDivisionError).
- Phân tích hiệu quả chiến dịch (Campaign ROI & Conversion funnel).
======

SELECT 
    campaign_id,
    campaign_name,
    target_segment,
    budget_vnd,
    revenue_generated,
    (revenue_generated - budget_vnd) AS net_return_vnd,
    total_reach,
    total_converted,
    ROUND(total_converted * 100.0 / NULLIF(total_reach, 0), 2) AS conversion_rate_pct,
    ROUND((revenue_generated - budget_vnd) * 100.0 / NULLIF(budget_vnd, 0), 2) AS roi_pct
FROM crm_campaigns
WHERE status = 'COMPLETED'
ORDER BY net_return_vnd DESC
LIMIT 15;
======

Bài 10: Digital Banking Usage – Đo lường mức độ sử dụng Mobile Banking
Business: Digital Product Analytics
File: mb_sessions

Bối cảnh Business:
Product Owner của ứng dụng Mobile Banking cần theo dõi chỉ số sử dụng hàng ngày (DAU, số phiên đăng nhập và thời lượng online) để tối ưu trải nghiệm người dùng và phát hiện các ngày có lượng truy cập đột biến.

Yêu cầu:
- Từ bảng mb_sessions, gom nhóm theo ngày bắt đầu phiên đăng nhập (session_date).
- Tính:
  - total_sessions: Tổng số lượt mở ứng dụng mỗi ngày.
  - active_devices: Số lượng thiết bị duy nhất hoạt động trong ngày (Daily Active Devices - DAU).
  - total_duration_hours: Tổng thời lượng người dùng tương tác trên app (quy đổi sang giờ, làm tròn 2 số thập phân).
  - avg_duration_minutes: Thời lượng trung bình của một phiên (quy đổi sang phút).
- Sắp xếp theo ngày giảm dần.

Kiến thức:
- COUNT(DISTINCT device_id): Đếm số lượng thực thể phân biệt (Distinct Counting) trong ngày.
- Đổi đơn vị thời gian (giây sang phút/giờ) và định dạng số thực.
======

SELECT 
    CAST(session_start AS DATE) AS session_date,
    COUNT(session_id) AS total_sessions,
    COUNT(DISTINCT device_id) AS active_devices,
    ROUND(SUM(duration_seconds) / 3600.0, 2) AS total_duration_hours,
    ROUND(AVG(duration_seconds) / 60.0, 2) AS avg_duration_minutes
FROM mb_sessions
GROUP BY CAST(session_start AS DATE)
ORDER BY session_date DESC
LIMIT 15;
======

01. Customer và Account
File: core_customers + core_accounts
Yêu cầu: Lấy danh sách khách hàng và tài khoản ngân hàng của họ.
Hiển thị customer_id, full_name, account_id, account_number, account_type, balance.
Chỉ lấy tài khoản đang ACTIVE.

Kiến thức:
INNER JOIN
One-to-Many relationship
JOIN bằng FK và PK.
======

SELECT
    c.customer_id,
    c.full_name,
    a.account_id,
    a.account_number,
    a.account_type,
    a.balance
FROM core_customers AS c
INNER JOIN core_accounts AS a
    ON c.customer_id = a.customer_id
WHERE a.status = 'ACTIVE'
LIMIT 5;
======

02. Customer không có Account
File: core_customers + core_accounts
Yêu cầu:
Tìm những khách hàng chưa mở tài khoản ngân hàng nào.
Hiển thị customer_id, full_name, phone, email.
Kiến thức:
LEFT JOIN
IS NULL
Anti Join
======

SELECT
    c.customer_id,
    c.full_name,
    c.phone,
    c.email,
FROM core_customers AS c
LEFT JOIN core_accounts AS a
    ON c.customer_id = a.customer_id
WHERE a.status != 'ACTIVE'
GROUP BY c.customer_id, c.full_name, c.phone, c.email
LIMIT 5;
======

03. Account và Branch
File: core_accounts + core_branches
Yêu cầu:
Hiển thị tài khoản cùng chi nhánh quản lý.
Bao gồm account_number, account_type, balance, branch_name, province.

Chỉ lấy tài khoản đang hoạt động.
Kiến thức:
INNER JOIN
Relationship 1:N
Filtering sau JOIN.

======
SELECT
    a.account_number,
    a.account_type,
    a.balance,
    b.branch_name,
    b.province,
FROM core_branches AS b
LEFT JOIN core_accounts AS a
    ON b.branch_id = a.branch_id
WHERE a.status = 'ACTIVE'

LIMIT 5;
======
Level 2 – Multiple JOINs
04. Customer 360 – Customer, Account và Branch
File: core_customers + core_accounts + core_branches

Yêu cầu:
Xây dựng danh sách tổng quan khách hàng.
Hiển thị tên khách hàng, tỉnh thành, số tài khoản, loại tài khoản, số dư và chi nhánh.
Bao gồm cả khách hàng chưa có tài khoản.

Kiến thức:
Multiple JOINs
LEFT JOIN
Relationship chaining.

05. Customer và Internet Banking
File: core_customers + ib_users + ib_sessions

Yêu cầu:

Liệt kê khách hàng có tài khoản Internet Banking.

Hiển thị username, trạng thái tài khoản và thời điểm đăng nhập.

Bao gồm cả người dùng chưa từng đăng nhập.

Kiến thức:

Multiple JOINs

LEFT JOIN

Join condition vs WHERE condition.

06. Customer và CRM Interactions
File: core_customers + crm_contacts + crm_interactions

Yêu cầu:

Liệt kê khách hàng và lịch sử tương tác CRM.

Hiển thị tên khách hàng, segment, interaction type, channel, outcome và thời gian tương tác.

Bao gồm khách hàng chưa từng tương tác.

Kiến thức:

Multi-table JOIN

One-to-Many

NULL handling.

Level 3 – Advanced JOIN & Fanout
07. Customer có nhiều tài khoản và nhiều giao dịch
File: core_customers + core_accounts + core_transactions

Yêu cầu:

Tính tổng số tài khoản của từng khách hàng.

Tính tổng số giao dịch gửi tiền từ các tài khoản đó.

Tính tổng giá trị giao dịch.

Chỉ tính giao dịch có trạng thái SUCCESS (nếu đây là giá trị trạng thái thực tế của dataset).

Kiến thức:

Multiple JOINs

GROUP BY

Aggregation

Fanout risk.

Fanout challenge
08. Customer sử dụng cả Internet Banking và Mobile Banking
File: core_customers + ib_users + mb_devices + mb_sessions

Yêu cầu:

Tìm khách hàng sử dụng cả Internet Banking và Mobile Banking.

Hiển thị số lượng IB users, số lượng thiết bị mobile và số phiên sử dụng.

Không để số liệu bị nhân lên do JOIN nhiều bảng 1:N.

Kiến thức:

Multiple One-to-Many JOINs

Pre-aggregation

CTE

Fanout prevention.

09. Transaction và Account – Self JOIN
File: core_transactions + core_accounts (JOIN hai lần)

Yêu cầu:

Hiển thị thông tin tài khoản gửi và tài khoản nhận của từng giao dịch.

Hiển thị transaction_id, amount, transaction_date, tên chủ tài khoản gửi và tên chủ tài khoản nhận.

Giữ lại giao dịch không có tài khoản nhận.

Kiến thức:

Self JOIN concept

Multiple aliases

LEFT JOIN

Role-playing dimensions.

10. Customer Digital Banking 360
File:

core_customers

core_accounts

core_transactions

ib_users

ib_transfers

mb_devices

mb_qr_payments

Yêu cầu:

Xây dựng một bảng Customer Analytics gồm:

Metric

Ý nghĩa

total_accounts

Tổng số tài khoản

total_balance

Tổng số dư

total_transactions

Tổng số giao dịch Core

total_transaction_amount

Tổng giá trị giao dịch Core

total_ib_transfers

Tổng số lượt chuyển tiền IB

total_qr_payments

Tổng số giao dịch QR

total_devices

Tổng số thiết bị Mobile

has_internet_banking

Có sử dụng IB không

has_mobile_banking

Có sử dụng MB không

Điều kiện quan trọng:

Mỗi khách hàng chỉ xuất hiện một dòng.

Không được để JOIN làm nhân bản:

Account.

Transaction.

IB Transfer.

QR Payment.

Mobile Device.
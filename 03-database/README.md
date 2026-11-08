# Module 03: Database Internals & Architecture for Data Engineers

> **Mục tiêu**: Hiểu sâu về cách Database lưu trữ dữ liệu xuống đĩa (Disk I/O), cơ chế xử lý truy vấn (Compute), và cách lựa chọn công nghệ CSDL phù hợp cho từng tầng trong hệ thống Data Architecture.

---

## 📌 Khung nội dung đề xuất cho Module 03

Thay vì liệt kê tính năng cơ bản của từng phần mềm (như MySQL, MongoDB là gì), module này tập trung vào **kiến trúc lõi và bài toán kỹ thuật**:

| Thứ tự | Chủ đề | File | Câu hỏi cốt lõi DE cần trả lời |
| :--- | :--- | :--- | :--- |
| **01** | **OLTP vs OLAP** | `01-oltp-vs-olap.md` | Tại sao không chạy query phân tích trực tiếp trên DB Production? Row-store vs Columnar-store khác nhau thế nào về mặt I/O? |
| **02** | **ACID & Transactions** | `02-acid-and-transactions.md` | Bản chất của Transaction trong ngân hàng. Isolation Levels (Read Committed, Repeatable Read, Serializable) và các hiện tượng Dirty Read, Phantom Read. |
| **03** | **Storage Engines & Indexing** | `03-storage-engines-indexes.md` | B-Tree (Postgres/MySQL) vs LSM-Tree (RocksDB/Cassandra/ClickHouse). Khi nào Index phản tác dụng? Clustered vs Non-clustered Index. |
| **04** | **Partitioning & Sharding** | `04-partitioning-sharding.md` | Chiến lược chia bảng: Range (theo ngày/tháng), Hash (chia tải đều), List. Partition Pruning giúp tối ưu query ra sao? |
| **05** | **NoSQL Systems in DE** | `05-nosql-in-de.md` | Khi nào DE dùng Redis (cache, deduplication window), Cassandra (time-series, write-heavy), MongoDB (semi-structured raw data)? |
| **06** | **Write-Ahead Logging (WAL) & CDC** | `06-wal-and-cdc.md` | WAL hoạt động ra sao? Nền tảng để công cụ CDC (như Debezium) bắt thay đổi dữ liệu streaming từ database sang data lake. |

---

## 💡 Tư duy cốt lõi của DE khi tiếp cận Database:
1. **Dữ liệu được tổ chức trên đĩa thế nào?** (Row-oriented hay Column-oriented? Có nén Snappy/ZSTD không?).
2. **Read-heavy hay Write-heavy?** (Cần tối ưu ghi hàng triệu event/giây hay cần tối ưu quét hàng tỷ bản ghi để tính sum/avg?).
3. **Mô hình tính nhất quán (Consistency):** Strong Consistency (ACID) hay Eventual Consistency (BASE / CAP Theorem)?

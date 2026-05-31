+++
date = '2026-05-31'
draft = false
title = '7 Sai Lầm Phổ Biến Nhất Khi Đánh Index Trong Database Mà Developer Thường Mắc Phải'
summary = "Index là con dao hai lưỡi — đánh đúng thì query nhanh như tên bắn, đánh sai thì INSERT/UPDATE cũng chậm. Đây là 7 sai lầm phổ biến nhất mà backend developer thường gặp khi đánh index."
tags = ['Database', 'Index', 'Performance', 'Backend']
+++

Đánh index tưởng đơn giản — cứ `CREATE INDEX` là xong. Nhưng thực tế, index là con dao hai lưỡi. Đánh đúng thì query nhanh như tên bắn, đánh sai thì database ì ạch, INSERT/UPDATE cũng chậm nốt.

Dưới đây là 7 sai lầm mà mình thấy backend developer mắc phải nhiều nhất.

---

## 1. Đánh Index "Vô Tội Vạ" (Over-Indexing)

Đánh index cho mọi cột "phòng khi cần" là sai lầm phổ biến nhất.

**Tại sao sai?** Mỗi index là một B-Tree riêng. Mỗi lần `INSERT`, `UPDATE`, `DELETE` — database phải cập nhật **tất cả** index của table đó.

```
Table có 10 index → 1 INSERT = 10 B-Tree update
```

Chi phí ẩn này ít ai để ý cho đến khi write performance tụt dốc.

**Rule:** Chỉ đánh index cho cột **thực sự** xuất hiện trong WHERE/JOIN/ORDER BY của query production.

---

## 2. Không Hiểu Leftmost Prefix Rule (Composite Index)

Composite index `(A, B, C)` không phải "đánh xong dùng thế nào cũng được".

Nó chỉ hoạt động khi query dùng các cột theo **thứ tự từ trái sang**:

| Query WHERE         | Dùng index `(A, B, C)`? |
|---------------------|------------------------|
| `A = ?`             | ✅ Có                   |
| `A = ? AND B = ?`   | ✅ Có                   |
| `A = ? AND B = ? AND C = ?` | ✅ Có         |
| `B = ?`             | ❌ Không               |
| `C = ?`             | ❌ Không               |
| `A = ? AND C = ?`   | ⚠️ Chỉ dùng được A     |

Query chỉ filter theo `B` hoặc `C` → index vô dụng, full table scan.

**Rule:** Thứ tự cột trong composite index = thứ tự quan trọng nhất trong query.

---

## 3. Đánh Index Cho Cột Có Độ Phân Biệt Thấp (Low Cardinality)

Index hiệu quả khi filter xong **loại bỏ được phần lớn** số row.

Ví dụ kinh điển: đánh index cho cột `gender` (Male/Female).

```sql
SELECT * FROM users WHERE gender = 'Male';
-- Index giúp gì? Vẫn phải scan ~50% table → optimizer thà full scan còn nhanh hơn
```

Ngược lại, index trên `email`, `phone`, `username` — query xong còn lại 1 row → index phát huy tác dụng.

**Rule:** Càng nhiều giá trị khác nhau → index càng hiệu quả.

---

## 4. Dùng Hàm / Biến Đổi Trên Cột Đã Đánh Index

```sql
-- ❌ Index trên created_at bị BỎ QUA
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
SELECT * FROM users WHERE UPPER(email) = 'ADMIN@EXAMPLE.COM';
SELECT * FROM logs WHERE DATE(timestamp) = '2024-01-15';
```

Database không thể dùng B-Tree index khi cột bị bao bởi hàm — vì giá trị trong tree là giá trị gốc, không phải kết quả sau khi transform.

**Viết lại để index hoạt động:**

```sql
-- ✅ Range scan — index hoạt động
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

SELECT * FROM users
WHERE email = 'admin@example.com';

SELECT * FROM logs
WHERE timestamp >= '2024-01-15 00:00:00'
  AND timestamp <  '2024-01-16 00:00:00';
```

---

## 5. Không Đánh Index Cho Foreign Key

Nhiều developer (nhất là MySQL/InnoDB) **tưởng FK tự động có index**. Sự thật: **InnoDB KHÔNG auto-create index cho FK**.

Hậu quả?

- `DELETE` / `UPDATE` với cascade → database phải scan toàn bộ table con để kiểm tra reference
- JOIN theo FK column → full table scan
- Table lock nghiêm trọng trên production

```sql
-- FK constraint có, nhưng KHÔNG có index → danger zone
ALTER TABLE orders ADD CONSTRAINT fk_customer
  FOREIGN KEY (customer_id) REFERENCES customers(id);

-- Phải đánh thêm index thủ công:
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```

PostgreSQL làm tốt hơn ở khoản này — auto index cho FK constraint.

---

## 6. Đánh Index Trên Cột Kiểu Dữ Liệu "Rộng"

Index trên `TEXT`, `VARCHAR(500)`, `JSON` → tốn memory, tốn disk, so sánh chậm.

**Giải pháp thay thế:**

```sql
-- Prefix index (chỉ index N ký tự đầu)
CREATE INDEX idx_email_prefix ON users(email(20));

-- Generated column + index (MySQL 5.7+)
ALTER TABLE data ADD COLUMN role VARCHAR(50)
  GENERATED ALWAYS AS (data->>'$.role') STORED;
CREATE INDEX idx_role ON data(role);
```

---

## 7. Implicit Type Conversion — Lỗi "Im Lặng" Nguy Hiểm Nhất

```sql
-- Cột phone kiểu VARCHAR
SELECT * FROM users WHERE phone = 0912345678;
-- ↑ Truyền số, không phải string → DB phải CONVERT mọi row sang number để so sánh
-- → Index bị BỎ QUA, nhưng query vẫn chạy bình thường (không error!)
```

Lỗi này đặc biệt nguy hiểm vì:
- Query **không báo lỗi**
- Vẫn trả kết quả đúng
- Nhưng performance tệ hơn hàng trăm lần

**Rule:** Kiểu dữ liệu của query parameter phải **match chính xác** với column definition. VARCHAR thì truyền string, INT thì truyền số.

---

## Tổng Kết

| Sai lầm | Hậu quả |
|---------|---------|
| Over-indexing | Chậm write, tốn disk |
| Sai thứ tự composite index | Index không được dùng |
| Index cột low cardinality | Waste resource |
| Hàm trên indexed column | Index bị vô hiệu hóa |
| Quên index FK | Full scan + table lock |
| Index kiểu dữ liệu rộng | Tốn memory/disk |
| Implicit type conversion | Index bị bypass âm thầm |

**Rule vàng:** Index để phục vụ query, không phải để trang trí. Luôn `EXPLAIN` trước và sau khi đánh index, và định kỳ xóa index không dùng.

---

*Bài viết ngắn nhưng hy vọng giúp bạn tránh được những cái bẫy mà chính mình cũng từng dính 😄*

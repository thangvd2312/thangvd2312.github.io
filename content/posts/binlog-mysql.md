+++
date = '2024-03-04'
draft = false
title = 'MySQL Binlog và Thứ Tự Hoạt Động Transaction'
summary = "Trong MySQL, Binlog là file ghi lại mọi thay đổi dữ liệu trong DB dưới dạng sự kiện (event). Nó có hai vai trò chính: Replication và Point-in-Time Recovery."
tags = ['Database', 'MySQL', 'Binlog']
+++

# 🧩 MySQL Binlog và Thứ Tự Hoạt Động Transaction

## 📦 1️⃣ Binlog là gì?

**Binlog (Binary Log)** là file ghi lại **mọi thay đổi dữ liệu** trong MySQL dưới dạng sự kiện (event).  
Ví dụ: mỗi câu lệnh `INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`... đều được ghi vào binlog.

### 🎯 Mục đích chính của Binlog
| Mục tiêu | Ý nghĩa |
|-----------|----------|
| **Replication (Master–Slave)** | Replica đọc binlog của master để “replay” lại các thay đổi và đồng bộ dữ liệu |
| **Backup / Point-in-Time Recovery (PITR)** | Có thể phục hồi dữ liệu đến thời điểm cụ thể bằng cách chạy lại binlog |
| **Theo dõi thay đổi dữ liệu (audit)** | Dễ dàng kiểm tra ai đã thay đổi gì trong DB |

### ⚙️ Tác dụng trong Replication
- Master ghi các sự kiện thay đổi dữ liệu vào binlog.
- Replica kết nối đến master và đọc binlog qua thread `I/O`.
- Replica lưu binlog này thành file **relay log** và “replay” nó qua thread `SQL`.

👉 Nhờ vậy, Replica có thể cập nhật dữ liệu **y hệt Master**, ngay cả khi chúng ở trên hai server khác nhau.

---

## ⚙️ 2️⃣ Thứ tự chạy một câu SQL trong MySQL

Giả sử bạn chạy lệnh:
```sql
INSERT INTO users VALUES (1, 'Thang');
```

### Bên trong MySQL sẽ diễn ra như sau:

| Bước | Mô tả | Ghi log nào | Ghi chú |
|------|--------|--------------|----------|
| 1️⃣ | Câu SQL được Parser và Optimize | — | Tạo plan để thực thi |
| 2️⃣ | Ghi thay đổi vào bộ nhớ (Buffer Pool) | — | Dữ liệu chỉ ở RAM |
| 3️⃣ | Ghi **Redo Log** với trạng thái *prepare* | 🧱 Redo Log | InnoDB lưu thay đổi vật lý, chuẩn bị commit |
| 4️⃣ | Ghi **Binlog** với nội dung logic (INSERT/UPDATE/DELETE) | 📦 Binlog | Dữ liệu phục vụ replication và backup |
| 5️⃣ | Đánh dấu **Redo Log → commit** | 🧱 Redo Log | Transaction chính thức hoàn tất |
| 6️⃣ | Trả kết quả `Query OK` cho client | — | Dữ liệu an toàn, có thể sync sang replica |

---

## 🔍 3️⃣ Vị trí của Binlog & Redo Log trong quá trình

| Thành phần | Vai trò | Khi ghi | Tác dụng chính |
|-------------|-----------|----------|----------------|
| 🧱 **Redo Log** | Ghi ở mức vật lý (page, block) | *Trước commit (prepare → commit)* | Phục hồi dữ liệu khi crash |
| 📦 **Binlog** | Ghi ở mức logic (SQL/row event) | *Sau khi prepare, trước commit redo* | Đồng bộ dữ liệu sang replica |

➡️ Hai log này hoạt động song song, được đồng bộ bằng **Two-Phase Commit**, đảm bảo:
> Hoặc cả hai cùng ghi thành công → transaction hoàn tất  
> Hoặc cả hai rollback → tránh sai lệch Master–Replica.

---

## 🧠 4️⃣ Tóm tắt nhanh

| Nội dung | Redo Log | Binlog |
|-----------|-----------|--------|
| Ghi bởi | InnoDB Engine | MySQL Server |
| Mục đích | Phục hồi sau crash | Replication & Backup |
| Dạng ghi | Thay đổi vật lý | Thay đổi logic |
| Vòng đời | Ngắn (vòng quay) | Dài (tuần/tháng) |
| Liên quan đến | Tính bền vững (Durability) | Tính đồng bộ (Consistency) |

---

## ✅ Kết luận

- **Binlog** là cầu nối giữa **Master và Replica**, giúp đồng bộ dữ liệu logic.  
- **Redo Log** đảm bảo dữ liệu **không mất khi crash**.  
- Hai log kết hợp thông qua **Two-Phase Commit**, đảm bảo MySQL luôn **an toàn và nhất quán**.

---

## 📚 Tham khảo & Đọc thêm

- [**Binlog trong MySQL - Thảo luận chi tiết**](https://chatgpt.com/share/69083298-6ee4-800f-9dee-246d4f32d2f8) - Cuộc trò chuyện về binlog, replication, server-id và các vấn đề thực tế trong production

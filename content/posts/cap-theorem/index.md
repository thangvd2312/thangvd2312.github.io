+++
date = '2025-03-10T07:15:58.412Z'
draft = false
title = 'CAP Theorem'
summary = "CAP trong hệ thống phân tán"
tags = ['System', 'Database']
+++

# CAP Theorem: Hiểu Rõ Consistency, Availability và Partition Tolerance

{{<figure src="./cap-theorem.png" width="500px" class="center">}}

## 1. CAP Theorem là gì?

CAP Theorem (định lý CAP) phát biểu rằng trong một hệ thống phân tán, ta chỉ có thể chọn **hai** trong ba thuộc tính sau:

1. **Consistency (Nhất quán - C)**: Mọi node trong hệ thống luôn có **dữ liệu giống nhau tại cùng một thời điểm**. Khi bạn đọc dữ liệu từ bất kỳ node nào, bạn sẽ luôn nhận được dữ liệu mới nhất.
2. **Availability (Sẵn sàng - A)**: Hệ thống **luôn phản hồi mọi request**, dù là đọc hay ghi, không có tình huống nào mà hệ thống từ chối yêu cầu của user.
3. **Partition Tolerance (Chịu lỗi phân vùng - P)**: Hệ thống **vẫn hoạt động ngay cả khi một số node không thể giao tiếp với nhau** do lỗi mạng.

### Khi có sự cố mạng (Partition xảy ra), ta phải chọn giữa **C hoặc A**

Do mạng không phải lúc nào cũng ổn định, nên hầu hết hệ thống thực tế **phải chịu Partition Tolerance (P)**. Do đó, ta buộc phải chọn giữa **C hoặc A**, tức là **CP hoặc AP**.

---

## 2. Ba mô hình CAP

### **1️⃣ CA (Consistency + Availability, mất Partition Tolerance)**

📌 **Không chịu được lỗi phân vùng mạng** → Nếu hai node không thể liên lạc, **hệ thống phải dừng để tránh sai dữ liệu**.

👉 **Ví dụ:** Hệ thống database đơn lẻ như MySQL, PostgreSQL trên một server duy nhất. Nếu server mất kết nối, toàn bộ hệ thống sập.

**⛔ Vấn đề:** Không thể mở rộng thành hệ thống phân tán thực sự vì nếu một node gặp sự cố mạng, toàn bộ hệ thống ngừng hoạt động.

---

### **2️⃣ CP (Consistency + Partition Tolerance, mất Availability)**

📌 **Dữ liệu luôn đồng bộ giữa các node** → Nếu mất kết nối, **hệ thống sẽ từ chối một số request để tránh dữ liệu sai** (giảm Availability).

👉 **Ví dụ:**

- Ngân hàng (transactions không thể bị sai, nên hệ thống phải từ chối request nếu dữ liệu không đồng bộ).
- Google Spanner – nếu mất mạng giữa các trung tâm dữ liệu, một số giao dịch bị chặn thay vì để dữ liệu không đồng bộ.

**⛔ Vấn đề:** Một số request sẽ bị từ chối trong trường hợp mất kết nối, dẫn đến trải nghiệm người dùng bị gián đoạn.

---

### **3️⃣ AP (Availability + Partition Tolerance, mất Consistency)**

📌 **Hệ thống vẫn hoạt động ngay cả khi mất kết nối giữa các node** → Nhưng có thể trả về **dữ liệu cũ hoặc không đồng bộ** (mất C).

👉 **Ví dụ:**

- **DNS, CDN, Redis, DynamoDB** – hệ thống cần phải luôn online, chấp nhận dữ liệu có thể "lệch" trong một thời gian ngắn.
- **Hệ thống mạng xã hội** (Facebook, Twitter) – có thể hiển thị dữ liệu cũ trong trường hợp lỗi mạng nhưng không bao giờ sập.

**⛔ Vấn đề:** Người dùng có thể nhận được dữ liệu lỗi thời hoặc không nhất quán trong một khoảng thời gian nhất định.

---

## 3. Tóm tắt CAP Theorem bằng bảng

| Kiểu hệ thống | Đảm bảo                            | Mất gì?             | Ví dụ                              |
| ------------- | ---------------------------------- | ------------------- | ---------------------------------- |
| **CA**        | Consistency + Availability         | Partition Tolerance | MySQL đơn lẻ, PostgreSQL           |
| **CP**        | Consistency + Partition Tolerance  | Availability        | Hệ thống ngân hàng, Google Spanner |
| **AP**        | Availability + Partition Tolerance | Consistency         | DNS, CDN, Redis, DynamoDB          |

---

## 4. Tại sao ngân hàng chọn **CP** thay vì **CA**?

- **Nếu dùng CA**, khi có lỗi mạng giữa các node, hệ thống **sẽ phải dừng hoàn toàn** → Ngân hàng không thể để toàn bộ hệ thống ngừng hoạt động.
- **Nếu dùng CP**, hệ thống vẫn hoạt động nhưng có thể từ chối một số request để đảm bảo **tính nhất quán dữ liệu**.
- **Ngân hàng không thể chọn AP**, vì nếu dữ liệu không đồng bộ, tài khoản khách hàng có thể bị sai lệch.

💡 **Kết luận:** Hệ thống tài chính ưu tiên **CP**, chấp nhận giảm Availability để đảm bảo dữ liệu chính xác.

---

## 5. Khi nào chọn **CP** và **AP**?

✔ **Chọn CP nếu dữ liệu phải luôn chính xác, ngay cả khi có thể làm chậm request**.  
✔ **Chọn AP nếu hệ thống phải luôn hoạt động, ngay cả khi có thể đọc dữ liệu cũ**.  
✔ **CA không thực tế trong hệ thống phân tán, vì nếu mất mạng thì toàn bộ hệ thống sẽ dừng.**

---

🔥 **Bạn đã hiểu rõ CAP Theorem!** 🚀

## 6. Tóm tắt ngắn gọn về CAP Theorem

📌 **Nếu mất mạng giữa các node:**

- **CA sẽ sập** (không thể hoạt động nếu không có kết nối giữa các node).
- **CP sẽ từ chối request** để đảm bảo dữ liệu luôn đồng bộ.
- **AP vẫn hoạt động**, nhưng có thể trả về dữ liệu cũ hoặc không đồng bộ.

💡 **Ví dụ thực tế:**

- **Ngân hàng chọn CP** vì họ không thể chấp nhận dữ liệu sai, dù có thể làm chậm giao dịch.
- **CDN hoặc DNS chọn AP** vì ưu tiên hoạt động liên tục, chấp nhận dữ liệu có thể cũ một chút.

---

# 📌 CAP Theorem trong Cơ Sở Dữ Liệu (DBMS)

CAP Theorem có ảnh hưởng lớn đến cách các hệ quản trị cơ sở dữ liệu được thiết kế, đặc biệt là trong môi trường **phân tán**. Theo định lý này, một hệ thống chỉ có thể đảm bảo **tối đa hai trong ba thuộc tính** sau:

- **C** (**Consistency - Nhất quán**): Tất cả các node luôn thấy cùng một dữ liệu.
- **A** (**Availability - Khả dụng**): Mọi request đều nhận được phản hồi (có thể là dữ liệu lỗi thời).
- **P** (**Partition Tolerance - Chịu lỗi phân mảnh**): Hệ thống vẫn hoạt động ngay cả khi có lỗi mạng giữa các node.

## 1️⃣ **Hệ thống CA (Consistency + Availability, mất Partition Tolerance)**

📌 **Không chịu được lỗi mạng giữa các node**, nếu một node bị mất kết nối, hệ thống sẽ **sập hoàn toàn**.

🔹 **Ví dụ:**  
- **MySQL (single-node), PostgreSQL** trên một máy chủ duy nhất.
- Đảm bảo **Consistency** và **Availability**, nhưng nếu server mất mạng, hệ thống ngừng hoạt động.

🛑 **Hạn chế:** Không thể mở rộng thành hệ thống phân tán thực sự vì nếu có lỗi mạng, toàn bộ hệ thống sẽ ngừng hoạt động.

---

## 2️⃣ **Hệ thống CP (Consistency + Partition Tolerance, mất Availability)**

📌 **Dữ liệu luôn chính xác, đồng bộ giữa các node**, nhưng khi có lỗi mạng, **một số request có thể bị từ chối** để đảm bảo tính nhất quán.

🔹 **Ví dụ:**  
- **MongoDB (mặc định là CP)** → Nếu mất mạng giữa các node, hệ thống có thể từ chối đọc/ghi dữ liệu từ các node lỗi để đảm bảo tính nhất quán.
- **Google Spanner** → Đảm bảo Consistency tuyệt đối, chấp nhận giảm Availability nếu cần.

💡 **Ứng dụng:**  
- **Ngân hàng, giao dịch tài chính** (ưu tiên Consistency để không mất tiền của khách hàng).
- **Hệ thống quan trọng yêu cầu tính đồng bộ cao**.

---

## 3️⃣ **Hệ thống AP (Availability + Partition Tolerance, mất Consistency)**

📌 **Hệ thống luôn phản hồi request, ngay cả khi mất kết nối giữa các node**, nhưng có thể trả về **dữ liệu cũ hoặc không đồng bộ**.

🔹 **Ví dụ:**  
- **Cassandra, DynamoDB, CouchDB** → Luôn sẵn sàng phục vụ request, ngay cả khi một số node bị mất kết nối.
- **DNS, CDN** → Phải luôn hoạt động, dữ liệu có thể lỗi thời một chút nhưng không thể ngừng.

💡 **Ứng dụng:**  
- **Hệ thống cần tốc độ cao** (ví dụ: mạng xã hội, hệ thống cache như Redis, Memcached).
- **Hệ thống phân tán toàn cầu** (ví dụ: DNS, CDN).

---

## 📊 **Bảng tổng hợp hệ quản trị CSDL theo CAP Theorem**

| Hệ quản trị DB | Loại | CAP ưu tiên | Ứng dụng thực tế |
|----------------|------|-------------|-------------------|
| **MySQL (single-node)** | Quan hệ | **CA** | Hệ thống nhỏ, không phân tán |
| **PostgreSQL** | Quan hệ | **CA** | OLTP, ERP, CRM |
| **MongoDB** | NoSQL | **CP (mặc định), AP (cấu hình)** | Ứng dụng linh hoạt, API, logging |
| **Cassandra** | NoSQL | **AP** | Hệ thống cần tốc độ cao, logs, IoT |
| **DynamoDB** | NoSQL | **AP** | Amazon Web Services |
| **Google Spanner** | NewSQL | **CP** | Hệ thống tài chính, ngân hàng |

---

## 💡 **Tóm tắt lựa chọn DB theo CAP**

✔ **Chọn CP nếu dữ liệu phải luôn chính xác**, ngay cả khi có thể làm chậm request (ngân hàng, giao dịch).  
✔ **Chọn AP nếu hệ thống phải luôn hoạt động**, chấp nhận dữ liệu cũ (CDN, DNS, logging).  
✔ **MongoDB linh hoạt giữa CP và AP**, có thể tùy chỉnh theo nhu cầu.  

---

Bạn có muốn sơ đồ minh họa chi tiết hơn cho từng hệ thống này không? 🚀
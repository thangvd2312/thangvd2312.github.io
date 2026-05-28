+++
title = 'Opaque Token - Everything You Need To Know'
date = '2026-05-28'
draft = false
summary = "Opaque token là gì, cách hoạt động, so sánh với JWT, và khi nào nên dùng"
tags = ['Security', 'Authentication', 'Backend']
+++

# Opaque Token — Tổng hợp chi tiết

> **Tagline:** *Access Token không chứa gì cả, chỉ là chuỗi random làm key để lookup trong DB.*

---

## 1. Access Token là gì?

**Access Token** là khái niệm chung — token mà client gửi lên server để truy cập tài nguyên protected. Nó không quy định format cụ thể.

Access Token có thể được implement dưới 2 format phổ biến:

```
Access Token (vai trò)
├── JWT format       → tự chứa thông tin (self-contained)
│   └── eyJhbGciOi...eyJ1c2VySWQiOiIxMjM...signature
│
└── Opaque format    → chỉ là chuỗi random, không chứa gì
    └── v2.local.adsjSDA723b1-asdnsa...
```

**Nói cách khác:** Opaque Token CHÍNH LÀ một loại Access Token. Không phải 2 thứ khác nhau.

---

## 2. JWT là gì? (Để so sánh)

JWT (JSON Web Token) là loại token **self-contained** — chứa đầy đủ thông tin trong chính nó:

```
header.payload.signature
```

- **Header:** Thuật toán mã hóa (HS256, RS256...)
- **Payload:** Dữ liệu user (userId, role, email...) — encode Base64, **ai cũng decode đọc được**
- **Signature:** Chữ ký điện tử — đảm bảo token không bị giả mạo

### Đặc điểm chính:

| Điểm | Chi tiết |
|---|---|
| **Stateless** | Server không cần lưu gì — verify signature là xong |
| **Có payload** | Chứa user info, decode Base64 là đọc được |
| **Có cryptographic signature** | Dùng secret/private key ký → verify bằng toán học |
| **Khó revoke** | Token đã phát hành thì đợi expire, không thu hồi được (trừ khi dùng blocklist) |
| **Nhanh** | Không cần query DB mỗi lần verify |

---

## 3. Opaque Token là gì?

Opaque Token (hay Reference Token) chỉ đơn giản là một **chuỗi ký tự ngẫu nhiên**, không chứa bất kỳ thông tin nào về người dùng.

Ví dụ: `v2.local.adsjSDA723b1-asdnsa...`

Token này chỉ đóng vai trò **"mã tham chiếu"**. Server phải tra cứu trong database để biết nó thuộc về ai.

### Đặc điểm chính:

| Điểm | Chi tiết |
|---|---|
| **Stateful** | Server phải lưu session/token info trong DB |
| **Không có payload** | Nhìn vào token không biết gì cả |
| **Không có signature** | Chỉ là random string, không ký gì hết |
| **Dễ revoke** | Xóa khỏi DB → token mất hiệu lực ngay lập tức |
| **Chậm hơn** | Mỗi lần verify = 1 lần query DB |

---

## 4. So sánh JWT vs Opaque Token

| Tiêu chí | Opaque Token | JWT |
|---|---|---|
| **Cấu trúc** | Chuỗi ngẫu nhiên vô nghĩa | `header.payload.signature` |
| **State** | **Stateful** — server lưu session | **Stateless** — server không lưu gì |
| **Bảo mật** | **Cao hơn** — bị lộ cũng vô dụng, revoke ngay | **Thấp hơn** — payload decode đọc được, khó revoke |
| **Hiệu năng** | **Chậm hơn** — mỗi lần verify = query DB | **Nhanh hơn** — verify signature bằng CPU |
| **Kích thước** | Nhỏ gọn, độ dài cố định | Lớn hơn, tùy số lượng claims |
| **Scale** | Phức tạp hơn — cần centralized auth | Dễ scale — service nào có key là verify được |

---

## 5. Ví dụ so sánh dễ hình dung

```
JWT = tấm hộ chiếu
→ Trên đó ghi rõ: tên, ngày sinh, quốc tịch
→ Có dấu mộc (signature) để xác thực không giả mạo
→ Nhìn vào là biết ai, không cần tra cứu

Opaque Token = số CMND
→ Chỉ là 1 con số: 0123456789
→ Nhìn số đó không biết gì cả
→ Phải tra cứu trong hệ thống mới biết ai
```

---

## 6. Khi nào dùng cái nào?

### ✅ Opaque Token khi:

- **Bảo mật là ưu tiên số 1** — cần revoke token ngay lập tức
  - User đổi mật khẩu → revoke tất cả token cũ
  - User logout → revoke token của device đó
  - Tài khoản bị khóa → revoke ngay
- **First-party apps** — app nội bộ, không cần share token cho bên thứ 3
- **Không muốn lộ info user trong token**

### ✅ JWT khi:

- **Hiệu năng quan trọng** — lượng request lớn, không muốn query DB
- **Microservices** — mỗi service tự verify được, không phụ thuộc centralized auth
- **Third-party apps** — cho bên ngoài tích hợp xác thực

---

## 7. Quy trình Opaque Token hoạt động

```
┌─────────┐         ┌──────────────────┐         ┌─────────┐
│  Client  │◄───────►│  Auth Server     │         │  Redis  │
│         │         │  (Express/Node)  │◄───────►│(store)  │
└─────────┘         └──────────────────┘         └─────────┘
     │                      │
     │ 1. POST /login       │
     │ (username/password)  │
     ├─────────────────────►│
     │                      │ 2. Verify credentials
     │                      │    → lấy user info từ DB
     │                      │ 3. Sinh random token
     │                      │    lưu vào Redis:
     │                      │    token → user info (TTL 24h)
     │ 4. Bearer <token>    │
     │◄─────────────────────┤
     │                      │
     │ 5. GET /protected    │
     │    Authorization: Bearer <token>
     ├─────────────────────►│
     │                      │ 6. Query Redis bằng token
     │                      │    → lấy user info
     │ 7. 200 OK + data     │
     │◄─────────────────────┤
```

---

## 8. Implement Opaque Token với Node.js + TypeScript + Redis

### 8.1 Cài đặt

```bash
npm install express redis ioredis uuid crypto
npm install -D typescript @types/express @types/node
```

### 8.2 Server chính (`src/server.ts`)

```typescript
import express, { Request, Response } from "express";
import { createClient } from "redis";
import crypto from "crypto";

const app = express();
app.use(express.json());

// Redis client
const redis = createClient({ url: "redis://localhost:6379" });
redis.connect();

// Sinh opaque token (32 bytes → 64 ký tự hex)
function generateToken(): string {
  return crypto.randomBytes(32).toString("hex");
  // Kết quả: "a8f3k2m9x7q1b5c2d4e6f8g0..."
}
```

### 8.3 Login endpoint → tạo token

```typescript
app.post("/login", async (req: Request, res: Response) => {
  const { username, password } = req.body;

  // 1. Verify credentials (mock — thực tế query DB thật)
  if (username !== "jerry" || password !== "password123") {
    return res.status(401).json({ error: "Sai username/password" });
  }

  // 2. Lấy user info từ DB
  const user = { userId: "123", username: "jerry", role: "admin" };

  // 3. Sinh opaque token
  const token = generateToken();

  // 4. Lưu vào Redis — key là token, value là user info
  // TTL: 24h (token tự expire)
  await redis.set(
    `session:${token}`,
    JSON.stringify(user),
    { EX: 24 * 60 * 60 } // 86400 seconds
  );

  // 5. Trả token cho client
  res.json({
    accessToken: token,
    tokenType: "Bearer",
    expiresIn: 86400,
  });
});
```

### 8.4 Middleware xác thực (`src/authMiddleware.ts`)

```typescript
import { Request, Response, NextFunction } from "express";
import { createClient } from "redis";

const redis = createClient({ url: "redis://localhost:6379" });
redis.connect();

// Extend Express Request để có req.user
declare global {
  namespace Express {
    interface Request {
      user?: { userId: string; username: string; role: string };
    }
  }
}

export async function authMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  // 1. Lấy token từ header
  const authHeader = req.headers["authorization"];
  const token = authHeader?.startsWith("Bearer ")
    ? authHeader.slice(7)
    : null;

  if (!token) {
    return res.status(401).json({ error: "Không có token" });
  }

  // 2. Query Redis bằng token
  const userData = await redis.get(`session:${token}`);

  if (!userData) {
    return res.status(401).json({ error: "Token không hợp lệ hoặc đã hết hạn" });
  }

  // 3. Parse user info → gắn vào req.user
  req.user = JSON.parse(userData);

  // 4. Cho qua
  next();
}
```

### 8.5 Protected route

```typescript
import { authMiddleware } from "./authMiddleware";

app.get("/profile", authMiddleware, (req: Request, res: Response) => {
  // req.user đã có từ middleware
  res.json({
    message: "Đây là data protected",
    user: req.user,
  });
});

app.get("/admin", authMiddleware, (req: Request, res: Response) => {
  if (req.user?.role !== "admin") {
    return res.status(403).json({ error: "Không có quyền" });
  }
  res.json({ message: "Welcome admin!" });
});
```

### 8.6 Logout → revoke token

```typescript
app.post("/logout", authMiddleware, async (req: Request, res: Response) => {
  const authHeader = req.headers["authorization"];
  const token = authHeader?.slice(7);

  // Xóa token khỏi Redis → revoke ngay lập tức
  if (token) {
    await redis.del(`session:${token}`);
  }

  res.json({ message: "Đăng xuất thành công" });
});
```

---

## 9. Test với cURL

```bash
# 1. Login
curl -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"username":"jerry","password":"password123"}'

# Response:
{
  "accessToken": "a8f3k2…0...",
  "tokenType": "Bearer",
  "expiresIn": 86400
}

# 2. Gọi API protected
curl http://localhost:3000/profile \
  -H "Authorization: Bearer a8f3k2…0..."

# Response:
{
  "message": "Đây là data protected",
  "user": { "userId": "123", "username": "jerry", "role": "admin" }
}

# 3. Logout (revoke token)
curl -X POST http://localhost:3000/logout \
  -H "Authorization: Bearer a8f3k2…0..."

# 4. Gọi lại với token cũ → 401
curl http://localhost:3000/profile \
  -H "Authorization: Bearer a8f3k2…0..."
# → {"error": "Token không hợp lệ hoặc đã hết hạn"}
```

---

## 10. So sánh middleware — JWT vs Opaque Token

```typescript
// JWT middleware — KHÔNG cần query DB
function jwtMiddleware(req, res, next) {
  const token = req.headers.authorization?.slice(7);
  const user = jwt.verify(token, SECRET_KEY);  // ✅ verify bằng CPU
  req.user = user;
  next();
}

// Opaque Token middleware — PHẢI query DB
async function opaqueMiddleware(req, res, next) {
  const token = req.headers.authorization?.slice(7);
  const userData = await redis.get(`session:${token}`);  // ❌ phải I/O
  if (!userData) return res.status(401).send("Invalid");
  req.user = JSON.parse(userData);
  next();
}
```

---

## 11. Điểm quan trọng cần nhớ

| Điểm | Chi tiết |
|---|---|
| **Key format trong Redis** | `session:<token>` — prefix để dễ quản lý, scan, cleanup |
| **TTL** | Set expire khi lưu — token tự hủy khi hết hạn |
| **Token length** | 32 bytes = 64 hex chars — đủ entropy, không brute force được |
| **Revoke** | Chỉ cần `redis.del` → token mất hiệu lực ngay lập tức |
| **Force logout all devices** | Scan all keys `session:*` filter theo userId → delete hết |
| **Refresh token** | Cần thêm endpoint `/refresh` — sinh token mới trước khi cũ expire |

---

## 12. Tại sao ít nghe "Opaque Token"?

Vì tutorial thường gọi chung là "access token", "session token", hoặc "bearer token" — ít khi gọi rõ "opaque token".

Thực chất:

- **Session ID** trong express-session → chính là opaque token
- **Refresh token** của OAuth2 → thường cũng là opaque token
- **API key** → cũng là opaque token

Tất cả đều là **"chuỗi random, server phải query DB để biết nó thuộc về ai"** — tức là opaque token hết.

---

> **Tóm lại 1 câu:**
> **Access Token = khái niệm chung. JWT và Opaque Token = 2 cách implement.**
> JWT = nhanh, dễ scale, khó revoke.
> Opaque Token = bảo mật hơn, revoke được ngay, nhưng chậm hơn vì phải query DB.

---

*Document version: 1.0 | Date: 2026-05-08 | Author: Tom 🐱*

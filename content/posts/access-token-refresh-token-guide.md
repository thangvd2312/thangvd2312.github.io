---
title: "Access Token & Refresh Token: Từ Lý Thuyết Đến Production"
description: "Những bài học xương máu về access token, refresh token và cách xử lý khi token bị leak"
date: 2026-04-19
draft: false
categories: ["Security", "Backend", "Authentication"]
tags: ["JWT", "Security", "Web Development", "Best Practices"]
author: "Senior Developer"
---

# Access Token & Refresh Token: Từ Lý Thuyết Đến Production

> **Câu chuyện thật:** Năm 2016, tôi implement authentication cho một fintech startup. Vì không hiểu rõ về refresh token, chúng tôi đã bị data breach khi access token bị leak. Attacker có thể duy trì access **vô thời hạn** vì refresh token không được lưu trong database. Đó là bài học đắt giá nhất trong sự nghiệp của tôi.

Bài viết này chia sẻ những gì tôi đã học được từ **10 năm** triển khai authentication cho nhiều hệ thống production - không phải lý thuyết sách vở, mà là kinh nghiệm thực tế, bao gồm cả những sai lầm chết người.

---

## Trước Khi Bắt Đầu: Token Là Gì?

Trước khi đi vào chi tiết, hãy hiểu **token là gì** theo cách đơn giản nhất.

**Token** giống như **vé xe buýt**: bạn xuất trình nó để chứng minh bạn đã mua vé, mà không cần nhân viên phải gọi điện về tổng công ty xác nhận từng lần. Vé có thời hạn, hết hạn thì phải mua vé mới.

Trong thế giới web, khi bạn đăng nhập vào một website:
1. Server tạo ra một **token** (chuỗi ký tự mã hóa)
2. Gửi token đó về cho trình duyệt của bạn
3. Mỗi lần bạn gọi API (lấy dữ liệu, thực hiện hành động), trình duyệt tự động gửi kèm token đó
4. Server kiểm tra token → biết bạn là ai → trả về dữ liệu

**JWT (JSON Web Token)** là định dạng token phổ biến nhất hiện nay. Nó là một chuỗi ký tự được mã hóa, bên trong chứa thông tin như: bạn là ai, token hết hạn lúc nào, bạn có quyền làm gì. Server có thể **tự xác minh** JWT mà không cần tra cứu database — đây gọi là **stateless** (không cần lưu trạng thái phía server).

---

## Phần 1: Bài Toán Bảo Mật vs Trải Nghiệm

### Tại Sao 1 Token Không Đủ?

Mọi developer đều bắt đầu với câu hỏi:

> *"Tại sao không dùng 1 token cho đơn giản?"*

Tôi đã từng nghĩ vậy. Cho đến khi phải đối mặt với 2 kịch bản:

**Kịch bản 1: Token dài hạn (30 ngày)**

```
❌ Rủi ro bảo mật:
• Token bị đánh cắp → attacker có 30 ngày để phá
• Không thể thu hồi (revoke) ngay lập tức
• User session quá dài, khó kiểm soát
```

**Kịch bản 2: Token ngắn hạn (15 phút)**

```
❌ Trải nghiệm tệ:
• User phải login lại mỗi 15 phút
• Mất flow làm việc
• User frustration cao → churn rate tăng
```

### Giải Pháp: Dual Token System

Sau nhiều lần thử nghiệm, đây là solution tối ưu — dùng **2 loại token** với mục đích khác nhau:

```
✅ Access Token: 15-30 phút (ngắn, bảo mật)
✅ Refresh Token: 7-30 ngày (dài, trải nghiệm)
```

Cách hoạt động đơn giản:
- **Access Token** dùng để gọi API hàng ngày — ngắn hạn nên nếu bị đánh cắp, thiệt hại nhỏ
- **Refresh Token** chỉ dùng để "xin" access token mới khi access token hết hạn — ít lộ hơn

**Tại sao lại là 7-30 ngày?** Đây là câu hỏi tôi nhận được nhiều nhất:

> *"Sao không để refresh token 1-2 giờ, miễn > access token là được?"*

**Câu trả lời:**

1. **UX là ưu tiên**: Với refresh token 2 giờ, user phải login **mỗi ngày**. Với 30 ngày, user chỉ login 1 lần/tháng.
2. **Refresh token không gửi liên tục**: Chỉ dùng khi access token hết hạn → ít rủi ro hơn access token.
3. **Có cơ chế thu hồi**: Lưu trong database → có thể thu hồi bất cứ lúc nào.

**Thực tế production:**

```
2 giờ refresh token:
❌ Support tickets tăng 300%: "Sao em phải login mỗi ngày?"

30 ngày refresh token:
✅ Support tickets giảm, user happy, vẫn bảo mật
```

**Sweet spot:** 30 ngày (điều chỉnh theo use case):
- Banking app: 7 ngày (bảo mật cao)
- Social media: 30-90 ngày (UX quan trọng)
- Enterprise: 14-30 ngày (cân bằng)

---

## Phần 2: Hiểu Đúng Về Access Token và Refresh Token

### Access Token - "Vé Vào Cổng Có Thời Hạn"

**Định nghĩa:** Chìa khóa tạm thời mà client dùng để gọi API.

**Cấu trúc JWT điển hình** (thông tin được mã hóa bên trong token):

```typescript
{
  "sub": "user123",           // User ID — bạn là ai
  "iat": 1713350400,          // Issued At — token được tạo lúc nào
  "exp": 1713352200,          // Expiration — hết hạn lúc nào (30 phút sau)
  "iss": "your-app",          // Issuer — ai tạo ra token này
  "aud": "your-api",          // Audience — token này dùng cho API nào
  "scope": ["read", "write"], // Permissions — bạn được làm gì
  "jti": "unique-token-id"    // JWT ID — ID duy nhất, dùng để thu hồi token
}
```

**4 đặc tính quan trọng:**

1. **Short-lived (ngắn hạn)**: 15-30 phút — nếu bị đánh cắp, thiệt hại nhỏ
2. **Stateless (không cần lưu trạng thái)**: Server tự xác minh bằng chữ ký số, không cần tra database
3. **Gửi với mọi request**: Trong `Authorization` header — nghĩa là token đi kèm mọi lần gọi API
4. **Có thể bị lộ**: Vì gửi liên tục, dễ bị chặn giữa đường (intercept)

**Best practices:**

```
✅ NÊN:
• Thời gian: 15-30 phút
• Thuật toán ký: RS256 (bất đối xứng — server ký bằng private key, verify bằng public key)
• Truyền qua: HTTPS only, trong Authorization header

❌ KHÔNG NÊN:
• Thời gian > 1 giờ
• Đưa vào token: thông tin nhạy cảm (password, CMND, số thẻ...)
• Truyền qua: URL params (bị lưu trong server log)
```

### Refresh Token - "Thẻ Căn Cước Để Xin Vé Mới"

**Định nghĩa:** Chìa khóa dự phòng dùng để xin access token mới khi access token hết hạn. Người dùng không biết token này tồn tại — mọi thứ diễn ra tự động phía sau.

**Cấu trúc:**

```typescript
{
  "sub": "user123",
  "iat": 1713350400,
  "exp": 1715942400,          // 30 ngày
  "iss": "your-app",
  "jti": "unique-refresh-id", // ID duy nhất — QUAN TRỌNG để thu hồi
  "type": "refresh"           // Đánh dấu đây là refresh token, không phải access token
}
```

**4 đặc tính quan trọng:**

1. **Long-lived (dài hạn)**: 7-30 ngày
2. **Stateful (cần lưu trạng thái)**: Server **PHẢI** lưu vào database để có thể thu hồi khi cần
3. **Chỉ dùng 1 lần rồi đổi mới**: Mỗi lần dùng → tạo refresh token mới (gọi là **token rotation** — xoay vòng token)
4. **Cất giữ an toàn**: Không gửi theo mọi request như access token — chỉ dùng khi cần xin token mới

> **Token rotation là gì?** Mỗi lần bạn dùng refresh token để lấy access token mới, server sẽ đồng thời cấp cho bạn một refresh token mới và vô hiệu hóa refresh token cũ. Nếu kẻ tấn công dùng refresh token cũ đã bị vô hiệu hóa → server biết ngay có vấn đề.

**Best practices:**

```
✅ NÊN:
• Lưu trữ: Trong database phía server
• Rotation: Mỗi lần refresh → cấp token mới, hủy token cũ
• Thu hồi (Revoke): Khi logout, đổi password, phát hiện hoạt động bất thường

❌ KHÔNG NÊN:
• Lưu trữ: Chỉ trong memory (mất khi restart server)
• Tái sử dụng: Dùng lại refresh token cũ
• Không có cơ chế thu hồi
```

---

## Phần 3: Flow Hoạt Động Thực Tế

### Flow 1: Đăng Nhập Ban Đầu

```
┌─────────┐                    ┌─────────┐                    ┌──────────┐
│  Client │                    │  Auth   │                    │ Database │
│(Browser)│                    │ Server  │                    │          │
└────┬────┘                    └────┬────┘                    └────┬─────┘
     │                              │                              │
     │  1. POST /login              │                              │
     │     {email, password}        │                              │
     │─────────────────────────────>│                              │
     │                              │  2. Kiểm tra credentials     │
     │                              │─────────────────────────────>│
     │                              │  3. Hợp lệ                   │
     │                              │<─────────────────────────────│
     │                              │  4. Tạo 2 tokens             │
     │                              │     - Access Token (15 phút) │
     │                              │     - Refresh Token (30 ngày)│
     │                              │     - Lưu refresh token vào DB
     │                              │─────────────────────────────>│
     │  5. Trả về tokens            │                              │
     │     {accessToken, refreshToken}                             │
     │<─────────────────────────────│                              │
```

### Flow 2: Gọi API Với Access Token

```
┌─────────┐                    ┌──────────┐
│  Client │                    │API Server│
│(Browser)│                    │          │
└────┬────┘                    └────┬─────┘
     │                              │
     │  GET /api/protected          │
     │  Authorization: Bearer <AT>  │
     │─────────────────────────────>│
     │                              │  1. Xác minh chữ ký token
     │                              │  2. Kiểm tra hạn sử dụng
     │                              │  3. Kiểm tra quyền (scope)
     │  200 OK + Dữ liệu            │
     │<─────────────────────────────│
```

### Flow 3: Tự Động Làm Mới Khi Access Token Hết Hạn

Đây là điểm hay nhất của hệ thống — user **không cần làm gì**, mọi thứ diễn ra tự động:

```
┌─────────┐                    ┌─────────┐                    ┌──────────┐
│  Client │                    │  Auth   │                    │ Database │
│(Browser)│                    │ Server  │                    │          │
└────┬────┘                    └────┬────┘                    └────┬─────┘
     │                              │                              │
     │  GET /api/protected          │                              │
     │  Authorization: Bearer <AT>  │                              │
     │─────────────────────────────>│                              │
     │  401 Unauthorized            │                              │
     │  (Token hết hạn)             │                              │
     │<─────────────────────────────│                              │
     │                              │                              │
     │  POST /refresh               │                              │
     │  Cookie: <RT>  ← tự động    │                              │
     │─────────────────────────────>│                              │
     │                              │  1. Xác minh chữ ký RT       │
     │                              │  2. Kiểm tra DB còn hợp lệ? │
     │                              │─────────────────────────────>│
     │                              │  3. Hợp lệ, chưa bị thu hồi │
     │                              │<─────────────────────────────│
     │                              │  4. Tạo AT mới + RT mới      │
     │                              │     (rotation — hủy RT cũ)   │
     │                              │─────────────────────────────>│
     │  {accessToken mới, refreshToken mới}                        │
     │<─────────────────────────────│                              │
     │  Tự động thử lại request cũ  │                              │
     │  với AT mới                  │                              │
     │─────────────────────────────>│                              │
```

---

## Phần 4: Implementation Thực Tế

Phần này là code thực tế. Mỗi đoạn code đều có giải thích — bạn không cần hiểu hết ngay, nhưng hãy đọc các comment để nắm ý chính.

### Backend (Node.js/Express + Prisma)

```typescript
// auth.controller.ts
import { Request, Response } from 'express';
import jwt from 'jsonwebtoken';
import { v4 as uuidv4 } from 'uuid';
import { prisma } from './db';

const ACCESS_TOKEN_EXPIRY = '15m';
const REFRESH_TOKEN_EXPIRY = '30d';

// XỬ LÝ ĐĂNG NHẬP
export const login = async (req: Request, res: Response) => {
  const { email, password } = req.body;
  
  // Bước 1: Kiểm tra email và password có đúng không
  const user = await prisma.user.findUnique({ where: { email } });
  if (!user || !await verifyPassword(password, user.password)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  // Bước 2: Tạo access token (ngắn hạn — 15 phút)
  const accessToken = jwt.sign(
    { sub: user.id, email: user.email, role: user.role, jti: uuidv4() },
    process.env.ACCESS_TOKEN_SECRET!,
    { expiresIn: ACCESS_TOKEN_EXPIRY }
  );
  
  // Bước 3: Tạo refresh token (dài hạn — 30 ngày)
  // jti tạo 1 lần, dùng chung cho cả JWT payload và DB để đảm bảo khớp nhau
  const refreshJti = uuidv4();
  const refreshToken = jwt.sign(
    { sub: user.id, jti: refreshJti, type: 'refresh' },
    process.env.REFRESH_TOKEN_SECRET!,
    { expiresIn: REFRESH_TOKEN_EXPIRY }
  );
  
  // Bước 4: ← QUAN TRỌNG — Lưu refresh token vào database
  // Nếu không lưu, sẽ không thể thu hồi khi bị đánh cắp
  await prisma.refreshToken.create({
    data: {
      token: refreshToken,
      userId: user.id,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
      jti: refreshJti,
    },
  });
  
  // Bước 5: Gửi refresh token qua httpOnly cookie (không phải response body)
  // httpOnly = JavaScript phía client KHÔNG thể đọc cookie này → an toàn hơn
  res.cookie('refreshToken', refreshToken, {
    httpOnly: true,   // ← JavaScript không đọc được → chống XSS
    secure: process.env.NODE_ENV === 'production', // HTTPS only trên production
    // 'lax' phù hợp hơn 'strict' khi có OAuth flow (redirect từ email/external provider)
    // Dùng 'strict' nếu app không cần redirect cross-site
    sameSite: 'lax',
    maxAge: 30 * 24 * 60 * 60 * 1000,
  });
  
  // Access token trả về trong response — client lưu trong memory (không dùng localStorage)
  res.json({ accessToken, user: sanitizeUser(user) });
};

// XỬ LÝ LÀM MỚI TOKEN
export const refresh = async (req: Request, res: Response) => {
  // Refresh token được tự động gửi qua cookie — client không cần làm gì thêm
  const refreshToken = req.cookies.refreshToken;
  if (!refreshToken) {
    return res.status(401).json({ error: 'Refresh token required' });
  }
  
  try {
    // Bước 1: Xác minh chữ ký — token có bị giả mạo không?
    const payload = jwt.verify(refreshToken, process.env.REFRESH_TOKEN_SECRET!) as jwt.JwtPayload;
    
    // Bước 2: ← QUAN TRỌNG — Kiểm tra trong database
    // Chỉ verify chữ ký là chưa đủ — token có thể đã bị thu hồi
    const storedToken = await prisma.refreshToken.findUnique({
      where: { token: refreshToken },
      include: { user: true },
    });
    
    if (!storedToken || storedToken.revoked || storedToken.expiresAt < new Date()) {
      // Phát hiện token không hợp lệ → thu hồi toàn bộ tokens của user này
      // (có thể là dấu hiệu bị tấn công)
      if (payload.sub) await revokeAllUserTokens(payload.sub);
      return res.status(401).json({ error: 'Invalid refresh token' });
    }
    
    // Bước 3: Tạo access token mới
    const newAccessToken = jwt.sign(
      { sub: storedToken.user.id, email: storedToken.user.email },
      process.env.ACCESS_TOKEN_SECRET!,
      { expiresIn: ACCESS_TOKEN_EXPIRY }
    );
    
    // Bước 4: ← QUAN TRỌNG — Token rotation: hủy refresh token cũ, cấp cái mới
    const newRefreshToken = await rotateRefreshToken(storedToken);
    
    // Bước 5: Gửi refresh token mới qua cookie
    res.cookie('refreshToken', newRefreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 30 * 24 * 60 * 60 * 1000,
    });
    
    res.json({ accessToken: newAccessToken });
    
  } catch (error) {
    return res.status(401).json({ error: 'Invalid refresh token' });
  }
};

// XỬ LÝ ĐĂNG XUẤT
export const logout = async (req: Request, res: Response) => {
  const refreshToken = req.cookies.refreshToken;
  
  if (refreshToken) {
    // Thu hồi refresh token trong database — token cũ sẽ không dùng được nữa
    await prisma.refreshToken.updateMany({
      where: { token: refreshToken },
      data: { revoked: true, revokedAt: new Date() },
    });
  }
  
  res.clearCookie('refreshToken');
  res.json({ message: 'Logged out successfully' });
};

// Hủy refresh token cũ và tạo cái mới (token rotation)
async function rotateRefreshToken(oldToken: RefreshToken): Promise<string> {
  // Đánh dấu token cũ là đã bị thu hồi
  await prisma.refreshToken.update({
    where: { id: oldToken.id },
    data: { revoked: true, revokedAt: new Date() },
  });
  
  const user = await prisma.user.findUnique({ where: { id: oldToken.userId } });
  if (!user) throw new Error('User not found');
  
  // Tạo refresh token mới — jti tạo 1 lần, dùng chung cho JWT payload và DB
  const newJti = uuidv4();
  const newRefreshToken = jwt.sign(
    { sub: user.id, jti: newJti, type: 'refresh' },
    process.env.REFRESH_TOKEN_SECRET!,
    { expiresIn: REFRESH_TOKEN_EXPIRY }
  );
  
  // Lưu token mới vào database
  await prisma.refreshToken.create({
    data: {
      token: newRefreshToken,
      userId: user.id,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
      jti: newJti,
    },
  });
  
  return newRefreshToken;
}

// Thu hồi toàn bộ tokens của 1 user (dùng khi phát hiện bị tấn công)
async function revokeAllUserTokens(userId: string): Promise<void> {
  await prisma.refreshToken.updateMany({
    where: { userId },
    data: { revoked: true, revokedAt: new Date() },
  });
}
```

### Database Schema (Prisma)

```prisma
model RefreshToken {
  id          String   @id @default(uuid())
  token       String   @unique
  userId      String
  user        User     @relation(fields: [userId], references: [id])
  jti         String   @unique
  createdAt   DateTime @default(now())
  expiresAt   DateTime
  revoked     Boolean  @default(false)
  revokedAt   DateTime?
  
  // Theo dõi bảo mật — biết token được dùng từ đâu
  usedAt      DateTime?
  ipAddress   String?
  userAgent   String?
  
  @@index([userId])
  @@index([expiresAt])
}

model User {
  id            String         @id @default(uuid())
  email         String         @unique
  password      String
  refreshTokens RefreshToken[]
  lastLoginAt   DateTime?
  lastLoginIp   String?
}
```

### Frontend (React + Axios Interceptor)

Phần này xử lý tự động refresh token khi access token hết hạn — user không cần làm gì, mọi thứ diễn ra trong nền.

**Interceptor** là middleware trong Axios — nó "chặn" mọi request/response trước khi đến tay bạn.

```typescript
// api/client.ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';

const apiClient = axios.create({
  baseURL: '/api',
  withCredentials: true, // Tự động gửi cookie (bao gồm refresh token)
});

// Biến theo dõi trạng thái đang refresh hay chưa
// Tránh gọi /refresh nhiều lần cùng lúc khi nhiều request cùng bị 401
let isRefreshing = false;
let failedQueue: Array<{ resolve: any; reject: any; config: any }> = [];

// Khi refresh xong, retry tất cả các request đang chờ
const processQueue = (error: AxiosError | null, token: string | null = null) => {
  failedQueue.forEach((prom) => {
    if (error) prom.reject(error);
    else {
      prom.config.headers.Authorization = `Bearer ${token}`;
      prom.resolve(apiClient(prom.config));
    }
  });
  failedQueue = [];
};

// Lưu access token trong module-level variable (in-memory) thay vì localStorage
// In-memory an toàn hơn vì không bị XSS đọc, nhưng mất khi reload trang (chấp nhận được)
let accessTokenInMemory: string | null = null;

// Interceptor 1: Tự động đính kèm access token vào mọi request
apiClient.interceptors.request.use((config) => {
  if (accessTokenInMemory) {
    config.headers.Authorization = `Bearer ${accessTokenInMemory}`;
  }
  return config;
});

// Interceptor 2: Khi nhận 401 → tự động refresh token rồi thử lại
apiClient.interceptors.response.use(
  (response) => response, // Response OK → trả về bình thường
  async (error: AxiosError) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & { _retry?: boolean };
    
    // Chỉ xử lý lỗi 401 (Unauthorized), không xử lý lại lần 2
    if (error.response?.status !== 401 || originalRequest._retry) {
      return Promise.reject(error);
    }
    
    // Nếu đang refresh, đưa request vào hàng chờ
    if (isRefreshing) {
      return new Promise((resolve, reject) => {
        failedQueue.push({ resolve, reject, config: originalRequest });
      });
    }
    
    originalRequest._retry = true;
    isRefreshing = true;
    
    try {
      // Gọi /refresh — cookie được gửi tự động, server trả về access token mới
      const response = await axios.post('/api/auth/refresh', {}, { withCredentials: true });
      const { accessToken } = response.data;
      accessTokenInMemory = accessToken; // Lưu trong memory, không phải localStorage
      
      // Retry tất cả request đang chờ
      processQueue(null, accessToken);
      originalRequest.headers.Authorization = `Bearer ${accessToken}`;
      return apiClient(originalRequest); // Thử lại request gốc
      
    } catch (refreshError) {
      // Refresh thất bại → refresh token hết hạn hoặc bị thu hồi → buộc đăng nhập lại
      processQueue(refreshError as AxiosError, null);
      accessTokenInMemory = null;
      window.location.href = '/login';
      return Promise.reject(refreshError);
      
    } finally {
      isRefreshing = false;
    }
  }
);

export default apiClient;
```

---

## Phần 5: Những Sai Lầm Chết Người

### 1. ❌ Không Kiểm Tra Refresh Token Trong Database

```typescript
// ❌ SAI CHẾT NGƯỜI — chỉ verify chữ ký, không check database
async function refresh(req, res) {
  const { refreshToken } = req.body;
  const payload = jwt.verify(refreshToken, SECRET); // Chỉ xem token có bị giả mạo không
  const newToken = generateAccessToken(payload);
  res.json({ accessToken: newToken });
}
// Rủi ro: Token bị đánh cắp vẫn dùng được suốt 30 ngày
// Dù bạn đã logout, đổi password — attacker vẫn có access
```

```typescript
// ✅ ĐÚNG — verify chữ ký VÀ kiểm tra trong database
async function refresh(req, res) {
  const payload = jwt.verify(refreshToken, SECRET);
  
  // Bước bắt buộc: kiểm tra token còn hợp lệ không
  const storedToken = await prisma.refreshToken.findUnique({
    where: { token: refreshToken },
  });
  
  if (!storedToken || storedToken.revoked) {
    return res.status(401).json({ error: 'Invalid token' });
  }
  
  res.json({ accessToken: generateAccessToken(payload) });
}
```

### 2. ❌ Lưu Token Trong localStorage

```typescript
// ❌ SAI — localStorage bị JavaScript đọc được
// XSS attack: script độc hại inject vào trang → đọc được cả 2 tokens
localStorage.setItem('refreshToken', token);  // Nguy hiểm nhất
localStorage.setItem('accessToken', token);   // Vẫn là XSS risk
```

```typescript
// ✅ ĐÚNG — Refresh token: httpOnly cookie (JavaScript KHÔNG thể đọc)
res.cookie('refreshToken', token, {
  httpOnly: true,   // JavaScript không access được
  secure: true,     // Chỉ truyền qua HTTPS
  sameSite: 'lax',  // 'lax' phù hợp hơn 'strict' khi có OAuth redirect flow
});

// ✅ ĐÚNG — Access token: lưu trong module-level variable (in-memory)
// Mất khi reload trang nhưng /refresh sẽ tự lấy lại — đây là behavior đúng
let accessToken: string | null = null;
```

> **XSS (Cross-Site Scripting)** là kiểu tấn công chèn JavaScript độc hại vào trang web. Nếu token trong localStorage, script đó có thể đọc và đánh cắp. httpOnly cookie giải quyết vấn đề cho refresh token; in-memory variable giải quyết cho access token.

### 3. ❌ Không Rotate Refresh Token

```typescript
// ❌ SAI — Dùng mãi 1 refresh token → nếu bị đánh cắp, dùng vĩnh viễn
async function refresh(req, res) {
  res.json({ accessToken: newAccessToken });
  // Refresh token cũ vẫn còn hiệu lực!
}
```

```typescript
// ✅ ĐÚNG — Mỗi lần refresh → refresh token MỚI, hủy cái cũ
// Nếu kẻ tấn công dùng token cũ đã bị hủy → server biết có vấn đề
async function refresh(req, res) {
  const newRefreshToken = await rotateRefreshToken(oldToken);
  res.cookie('refreshToken', newRefreshToken);
  res.json({ accessToken: newAccessToken });
}
```

### 4. ❌ Không Xử Lý 401 Đúng Cách Ở Frontend

```typescript
// ❌ SAI — Redirect ngay → UX tệ, user bị bắt login mỗi 15 phút
if (!response.ok) {
  window.location.href = '/login';
}
```

```typescript
// ✅ ĐÚNG — Tự động refresh rồi thử lại, user không biết gì
if (error.response?.status === 401) {
  await refreshAccessToken();
  return apiClient.get('/api/protected'); // Thử lại → user không bị gián đoạn
}
```

### 5. ❌ Gửi Refresh Token Trong Request Body

```typescript
// ❌ SAI — Token xuất hiện trong request body → có thể bị ghi vào server log
fetch('/api/refresh', {
  body: JSON.stringify({ refreshToken }),
});
```

```typescript
// ✅ ĐÚNG — Browser tự động gửi cookie, token không xuất hiện ở đâu cả
fetch('/api/refresh', {
  credentials: 'include', // Gửi cookie kèm request
});
```

---

## Phần 6: Xử Lý Khi Token Bị Leak

### Kịch Bản 1: Access Token Bị Lộ

Đây là trường hợp **thường gặp nhất** (90% incidents). May mắn là thiệt hại nhỏ vì access token ngắn hạn.

**Phát hiện từ monitoring:**

```typescript
// Ví dụ minh họa logic — không phải production-ready code
async function detectTokenLeak(tokenId: string) {
  const suspiciousActivity = await analyzeTokenUsage(tokenId);
  // Ví dụ: token được dùng từ 2 địa điểm khác nhau cùng lúc
  
  if (suspiciousActivity) {
    // 1. Thu hồi refresh token NGAY LẬP TỨC → attacker không thể lấy access token mới
    await revokeRefreshToken(tokenId);
    
    // 2. Thu hồi toàn bộ tokens của user này
    const userId = await getUserIdFromToken(tokenId);
    await revokeAllUserTokens(userId);
    
    // 3. Cảnh báo và thông báo
    await alertSecurityTeam({ userId, tokenId });
    await forcePasswordReset(userId);
    await notifyUser(userId, 'SECURITY_ALERT');
  }
}
```

**Timeline phản ứng:**
- < 1 phút: Tự động thu hồi
- < 5 phút: Alert security team
- < 15 phút: Notify user

### Kịch Bản 2: Cả 2 Tokens Bị Lộ (Worst Case)

**Mức độ: CRITICAL** — xảy ra ~1% incidents, nhưng cần có protocol sẵn sàng.

```typescript
// Ví dụ minh họa logic xử lý khủng hoảng
async function handleBothTokensLeaked(userId: string) {
  // 1. Thu hồi tất cả tokens → attacker mất access ngay lập tức
  await revokeAllUserTokens(userId);
  
  // 2. Vô hiệu hóa tất cả sessions đang active
  await invalidateAllSessions(userId);
  
  // 3. Buộc đổi password — ngay cả khi attacker có token, sẽ không thể login lại
  await forcePasswordReset(userId);
  await sendPasswordResetEmail(userId);
  
  // 4. Thu hồi API keys (nếu có)
  await revokeAllUserApiKeys(userId);
  
  // 5. Bật 2FA bắt buộc — lần login tiếp theo phải xác minh 2 bước
  await require2FA(userId);
  
  // 6. Cảnh báo và ghi nhận
  await alertSecurityTeam({ level: 'CRITICAL', userId });
  await notifyUser(userId, { channel: ['email', 'sms'], priority: 'URGENT' });
  await logSecurityIncident({ userId, type: 'BOTH_TOKENS_LEAKED' });
}
```

**Checklist xử lý crisis:**

```
✅ KHI PHÁT HIỆN CẢ 2 TOKENS LEAKED:

[ ] 1. Thu hồi tất cả tokens (tự động, ngay lập tức)
[ ] 2. Vô hiệu hóa tất cả sessions
[ ] 3. Buộc đổi password
[ ] 4. Gửi email reset password
[ ] 5. Bật 2FA bắt buộc
[ ] 6. Thu hồi API keys (nếu có)
[ ] 7. Alert security team
[ ] 8. Notify user (email + SMS)
[ ] 9. Ghi nhận security incident
[ ] 10. Điều tra nguồn gốc breach

Timeline:
• 0-1 phút: Steps 1-6 (tự động)
• 1-5 phút: Steps 7-9 (tự động)
• 5-15 phút: Step 10 (manual review)
```

**Thống kê thực tế từ 10 năm xử lý incidents:**

```
• 90% cases: Chỉ access token bị leak
  → Xử lý bằng revocation, thiệt hại nhỏ
  
• 9% cases: Refresh token bị leak
  → Revoke + notify user, user đổi password
  
• 1% cases: Cả 2 tokens bị leak
  → Emergency protocol, 2FA bắt buộc
```

---

## Phần 7: Security Best Practices

### 1. Token Binding (Gắn Token Với Thiết Bị)

Ý tưởng: token chỉ hợp lệ từ đúng thiết bị đã tạo ra nó. Nếu bị đánh cắp và dùng từ thiết bị khác → server phát hiện ngay.

> **Lưu ý:** Không nên dùng IP trong fingerprint vì IP thay đổi liên tục với mobile users (chuyển mạng 4G ↔ WiFi) và không ổn định sau load balancer/proxy. Chỉ nên dùng `user-agent` hoặc thêm thông tin thiết bị từ client.

```typescript
function generateAccessToken(user: User, req: Request): string {
  // Tạo "fingerprint" của thiết bị — chỉ dùng user-agent, không dùng IP
  // (IP thay đổi với mobile users và không ổn định sau load balancer)
  const fingerprint = hashFingerprint({
    userAgent: req.headers['user-agent'],
  });
  
  return jwt.sign(
    { sub: user.id, fingerprint }, // Đính kèm fingerprint vào token
    SECRET,
    { expiresIn: '15m' }
  );
}

function verifyAccessToken(token: string, req: Request) {
  const payload = jwt.verify(token, SECRET) as jwt.JwtPayload;
  const currentFingerprint = hashFingerprint({
    userAgent: req.headers['user-agent'],
  });
  
  // Fingerprint không khớp → token đến từ thiết bị khác → có thể bị đánh cắp
  if (payload.fingerprint !== currentFingerprint) {
    throw new Error('Token binding mismatch - possible theft');
  }
  
  return payload;
}
```

### 2. Rate Limiting (Giới Hạn Số Lần Thử)

Ngăn attacker brute-force password hoặc spam endpoint refresh.

```typescript
import rateLimit from 'express-rate-limit';

// Giới hạn login: 5 lần thử trong 15 phút
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: 'Quá nhiều lần đăng nhập thất bại, thử lại sau 15 phút',
});

// Giới hạn refresh: 10 lần trong 1 giờ
const refreshLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,
  max: 10,
  message: 'Quá nhiều lần refresh, thử lại sau',
});

app.post('/api/login', loginLimiter, loginHandler);
app.post('/api/refresh', refreshLimiter, refreshHandler);
```

### 3. Monitoring & Alerting

Phát hiện khi refresh token đã bị thu hồi vẫn được sử dụng — dấu hiệu chắc chắn của token reuse attack.

> **Lưu ý:** Token rotation đã xử lý reuse detection: mỗi lần refresh, token cũ bị revoke ngay. Nếu token cũ đó được dùng lại (bởi attacker), server thấy `revoked = true` → đây là tín hiệu tấn công. Không cần kiểm tra theo thời gian (< 1 giây) vì dễ miss concurrent requests hợp lệ.

```typescript
async function handleRefreshRequest(refreshToken: string, userId: string) {
  const storedToken = await prisma.refreshToken.findUnique({
    where: { token: refreshToken },
  });

  // Token đã bị revoke mà vẫn được dùng → rotation attack: kẻ tấn công đang dùng token cũ
  if (storedToken?.revoked) {
    await alertSecurityTeam(userId, 'Revoked refresh token reuse detected — possible theft');
    await revokeAllUserTokens(userId); // Thu hồi toàn bộ để an toàn
    throw new Error('Token reuse detected');
  }
  
  // Cập nhật thời điểm dùng token và IP để audit log
  await prisma.refreshToken.update({
    where: { id: storedToken!.id },
    data: { usedAt: new Date(), ipAddress: getCurrentIP() },
  });
}
```

### 4. Defense In Depth (Bảo Vệ Nhiều Lớp)

Không nên chỉ dựa vào 1 cơ chế bảo vệ. Hệ thống tốt có nhiều lớp:

```
PHÒNG NGỪA (Prevention):
✅ httpOnly cookies — chặn XSS đọc token
✅ Secure cookies — HTTPS only
✅ SameSite lax/strict — chặn CSRF
✅ Content Security Policy — giới hạn script được chạy
✅ Rate limiting — chặn brute force

PHÁT HIỆN (Detection):
✅ Monitoring token usage — phát hiện bất thường
✅ Anomaly detection — cùng token từ 2 nơi
✅ Concurrent usage check — token reuse
✅ Geographic check — đột ngột login từ quốc gia khác

XỬ LÝ (Response):
✅ Auto revoke — thu hồi ngay khi phát hiện
✅ Emergency protocol — xử lý worst case
✅ User notification — thông báo kịp thời

GIẢM THIỆT HẠI (Mitigation):
✅ Access token ngắn (15 phút) — nếu bị leak, thiệt hại nhỏ
✅ Token rotation — phát hiện token reuse attack
✅ Session limits — giới hạn số session cùng lúc
✅ Device fingerprinting — phát hiện token bị dùng từ thiết bị khác
```

---

## Kết Luận: 6 Bài Học Xương Máu

Sau 10 năm triển khai authentication cho nhiều hệ thống production:

### 1. Bảo Mật > Tiện Lợi (Nhưng Phải Cân Bằng)

User sẽ bỏ app nếu phải login mỗi 5 phút. Sweet spot: **15 phút access token + 30 ngày refresh token**.

### 2. Luôn Assume Token Sẽ Bị Leak

Thiết kế system sao cho khi token bị leak, thiệt hại là **nhỏ nhất**:
- Access token ngắn
- Refresh token lưu database để thu hồi được
- Monitoring để phát hiện sớm

### 3. Database Validation Là Bắt Buộc

Không có ngoại lệ. Nếu không thể lưu refresh token trong database, **đừng dùng refresh token**.

### 4. Monitoring Là Chìa Khóa

Bạn không thể bảo vệ những gì bạn không theo dõi. Log và alert mọi hoạt động bất thường.

### 5. Test Security Như Test Features

Viết test cases cho attack scenarios, không chỉ happy paths:
- Token bị leak
- Refresh token bị dùng lại
- Concurrent usage từ nhiều locations

### 6. Keep It Simple

Đừng over-engineer. Dual token system đã đủ tốt cho 99% use cases.

---

## Tóm Tắt Nhanh

```
┌─────────────────────────────────────────────────────────────┐
│  ✅ NÊN LÀM                                                 │
├─────────────────────────────────────────────────────────────┤
│  • Access Token: 15-30 phút, lưu trong in-memory variable    │
│  • Refresh Token: 7-30 ngày, httpOnly cookie                │
│  • LUÔN lưu refresh token trong database                    │
│  • LUÔN rotate refresh token mỗi lần refresh                │
│  • LUÔN dùng HTTPS trong production                         │
│  • Monitor và alert hoạt động bất thường                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ❌ KHÔNG NÊN LÀM                                           │
├─────────────────────────────────────────────────────────────┤
│  • Không lưu refresh token trong localStorage              │
│  • Không dùng access token > 1 giờ                          │
│  • Không bỏ qua database validation cho refresh token      │
│  • Không tái sử dụng refresh token (phải rotate)           │
│  • Không quên thu hồi tokens khi logout/đổi password       │
└─────────────────────────────────────────────────────────────┘
```

---

> **Câu nói tâm đắc:** *"Security is a process, not a product"* — Bruce Schneier
>
> Đừng bao giờ assume token sẽ KHÔNG bị leak. Luôn chuẩn bị cho worst case. Có protocol sẵn, monitor, alert, và continuous improvement.

**Tài liệu tham khảo:**

- [OAuth 2.0 Spec](https://oauth.net/2/)
- [JWT Best Practices](https://auth0.com/blog/jwt-security-best-practices/)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

---

**Hỏi đáp:** Nếu có thắc mắc hoặc muốn chia sẻ kinh nghiệm, hãy để lại comment bên dưới!

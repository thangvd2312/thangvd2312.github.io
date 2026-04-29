+++
date = '2026-04-29T00:00:00+07:00'
draft = false
title = 'Rate Limit từ A đến Z'
summary = "Rate limit là gì, tại sao cần nó, các thuật toán phổ biến, và cách triển khai thực tế trong hệ thống backend."
tags = ['Rate Limit', 'System Design', 'Backend', 'API']
+++

# Rate Limit từ A đến Z

## 1. Rate Limit là gì?

Hãy tưởng tượng bạn mở một quán cà phê. Mỗi buổi sáng bạn chỉ pha được tối đa 100 ly. Nếu hôm đó có 500 người kéo vào cùng lúc, bạn không thể phục vụ hết — bạn phải từ chối hoặc yêu cầu họ chờ.

**Rate Limit** hoạt động giống hệt vậy trong phần mềm: giới hạn số lượng request mà một client (người dùng, IP, API key...) được phép gửi đến server trong một khoảng thời gian nhất định.

Ví dụ:
- Tối đa **100 request / phút** mỗi user.
- Tối đa **10 lần đăng nhập sai / giờ** mỗi IP.
- Tối đa **5 lần gửi OTP / ngày** mỗi số điện thoại.

---

## 2. Tại sao cần Rate Limit?

### Bảo vệ server khỏi bị quá tải

Không có rate limit, một client duy nhất (hoặc kẻ tấn công) có thể gửi hàng triệu request, làm server cạn kiệt tài nguyên và sập.

### Ngăn chặn tấn công

- **Brute force**: thử mật khẩu liên tục → rate limit đăng nhập.
- **DDoS**: gửi traffic khổng lồ → rate limit theo IP.
- **Credential stuffing**: thử hàng loạt tài khoản → rate limit per user.

### Đảm bảo công bằng

Trong hệ thống nhiều tenant, một user không được chiếm hết tài nguyên, ảnh hưởng đến user khác.

### Kiểm soát chi phí

Nếu bạn gọi API bên thứ ba mỗi khi có request, rate limit giúp tránh bị tính phí vượt mức.

---

## 3. Các thuật toán Rate Limit phổ biến

Đây là phần quan trọng nhất. Có 5 thuật toán chính, mỗi cái có điểm mạnh riêng.

### 3.1 Fixed Window Counter (Cửa sổ cố định)

**Ý tưởng**: chia thời gian thành các cửa sổ cố định (ví dụ: mỗi phút), đếm số request trong cửa sổ đó.

```
|-- phút 1 --|-- phút 2 --|-- phút 3 --|
|  100 req   |   100 req  |   100 req  |
```

**Triển khai đơn giản**:
```
key = "user:123:2026-04-29-10:05"   ← timestamp cắt theo phút
count = INCR(key)
EXPIRE(key, 60)

if count > 100:
    return 429 Too Many Requests
```

✅ **Ưu điểm**: đơn giản, hiệu quả, dễ implement với Redis.

❌ **Nhược điểm — vấn đề boundary (biên cửa sổ)**:

Giả sử giới hạn là 100 req/phút. User gửi 100 req vào giây cuối của phút 1, rồi ngay đầu phút 2 gửi thêm 100 req. Trong khoảng 2 giây đó, server nhận đến **200 request** — gấp đôi giới hạn.

```
phút 1: ... [99 req] [100 req] |
phút 2:                        | [100 req] [101 req] ...
                        ↑ biên ↑
          ← 2s nhận 200 req →
```

---

### 3.2 Sliding Window Log (Nhật ký cửa sổ trượt)

**Ý tưởng**: lưu lại **timestamp của mỗi request**. Khi có request mới, xóa các timestamp cũ hơn 1 phút và đếm lại.

```
Timestamps: [10:05:01, 10:05:15, 10:05:44, 10:05:58, 10:06:02]
                                                      ↑ request mới lúc 10:06:02
Xóa cái nào < 10:05:02 → còn [10:05:15, 10:05:44, 10:05:58, 10:06:02]
Count = 4 → OK
```

✅ **Ưu điểm**: chính xác tuyệt đối, không có vấn đề biên cửa sổ.

❌ **Nhược điểm**: tốn bộ nhớ. Mỗi user lưu cả danh sách timestamp, hệ thống có triệu user là vấn đề.

---

### 3.3 Sliding Window Counter (Cửa sổ trượt ước lượng)

**Ý tưởng**: kết hợp Fixed Window (nhẹ) và Sliding Window Log (chính xác) — dùng công thức xấp xỉ.

```
count = count_phút_trước × (1 - tỉ_lệ_đã_qua) + count_phút_hiện_tại
```

Ví dụ: 
- Phút trước có 80 req.
- Phút hiện tại đã qua được 30% (18 giây).
- Phút hiện tại có 40 req.

→ `count ≈ 80 × (1 - 0.3) + 40 = 56 + 40 = 96` → dưới 100 → cho qua.

✅ **Ưu điểm**: cân bằng giữa chính xác và hiệu năng. Chỉ cần lưu 2 số thay vì toàn bộ log.

❌ **Nhược điểm**: không chính xác 100%, nhưng sai số rất nhỏ (< 1%) và thường chấp nhận được.

---

### 3.4 Token Bucket (Xô token)

**Ý tưởng**: hình dung một cái xô chứa token. Token được thêm vào đều đặn (ví dụ 10 token/giây). Mỗi request tiêu thụ 1 token. Nếu xô hết token → từ chối.

```
Xô capacity: 100 token
Refill rate:  10 token/giây

Lúc rảnh: xô đầy 100 token → user có thể burst 100 req cùng lúc
Sau burst: phải chờ token nạp lại → tối đa 10 req/giây
```

✅ **Ưu điểm**:
- Cho phép **burst traffic** ngắn hạn (xô đầy → xả hết một lúc).
- Dùng rộng rãi trong các API gateway (AWS, Stripe, GitHub đều dùng).

❌ **Nhược điểm**: cần lưu trạng thái (số token hiện tại, thời điểm refill cuối). Distributed system cần đồng bộ trạng thái.

---

### 3.5 Leaky Bucket (Xô rò rỉ)

**Ý tưởng**: request vào xô như nước, xô "rò" ra một lượng cố định. Nếu xô đầy → từ chối request mới.

```
Request vào → [=======Xô=======] → xử lý đều đặn 10 req/giây
Nếu xô đầy   →  429 Too Many Requests
```

✅ **Ưu điểm**: output **hoàn toàn đều đặn**, bảo vệ downstream khỏi spike.

❌ **Nhược điểm**: không xử lý burst — ngay cả burst hợp lý cũng bị từ chối nếu xô đầy.

---

### Bảng so sánh

| Thuật toán | Burst | Bộ nhớ | Độ chính xác | Phù hợp |
|---|---|---|---|---|
| Fixed Window | ✅ Có | Thấp | Thấp (biên) | Đơn giản, low traffic |
| Sliding Window Log | ❌ Không | Cao | Cao | Cần chính xác tuyệt đối |
| Sliding Window Counter | ✅ Có | Thấp | Cao (~99%) | Đại đa số use case |
| Token Bucket | ✅ Có | Thấp | Cao | API gateway, public API |
| Leaky Bucket | ❌ Không | Thấp | Cao | Output rate cần cố định |

---

## 4. Rate Limit ở đâu trong hệ thống?

```
Client → [Rate Limiter] → API Gateway → Service → Database
```

Rate limit có thể đặt ở nhiều tầng:

| Tầng | Ví dụ |
|---|---|
| **Client** | Tự throttle trước khi gửi (SDK) |
| **CDN / Edge** | Cloudflare, AWS CloudFront |
| **API Gateway** | Kong, AWS API Gateway, NGINX |
| **Application** | Code trong service, middleware |
| **Database** | Connection pool limit |

Thông thường nên đặt **càng sớm càng tốt** (edge/gateway) để tránh request độc hại đi sâu vào hệ thống.

---

## 5. Rate Limit theo chiều nào?

Bạn có thể giới hạn theo nhiều chiều khác nhau:

- **Per IP**: chống bot, DDoS.
- **Per User / Account**: giới hạn công bằng giữa các user.
- **Per API Key**: phân tier (free vs premium).
- **Per Endpoint**: `/login` giới hạn chặt hơn `/homepage`.
- **Per Tenant**: SaaS multi-tenant giới hạn theo org/company.
- **Global**: giới hạn tổng traffic toàn hệ thống.

Hệ thống thực tế thường kết hợp nhiều chiều: ví dụ giới hạn cả per-IP lẫn per-user.

---

## 6. Distributed Rate Limiting

Khi chạy nhiều server (horizontal scaling), mỗi server không thể tự đếm độc lập — cần **centralized store**.

```
Server 1  ↘
Server 2  → [Redis Cluster] ← lưu counter dùng chung
Server 3  ↗
```

**Redis** là lựa chọn phổ biến nhất nhờ:
- Atomic operations (`INCR`, `EXPIRE`).
- Tốc độ cực nhanh (in-memory).
- Hỗ trợ Lua script để xử lý logic phức tạp atomic.

**Ví dụ Redis + Lua (Fixed Window)**:
```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local count = redis.call('INCR', key)
if count == 1 then
    redis.call('EXPIRE', key, window)
end

if count > limit then
    return 0  -- bị giới hạn
end
return 1  -- cho qua
```

Dùng Lua đảm bảo INCR + EXPIRE là **atomic** — tránh race condition khi nhiều server cùng check.

---

## 7. Response khi bị Rate Limit

Khi request bị từ chối, trả về **HTTP 429 Too Many Requests** kèm headers để client biết:

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1714358400
Retry-After: 60
Content-Type: application/json

{
  "error": "rate_limit_exceeded",
  "message": "Bạn đã vượt quá giới hạn 100 request/phút. Thử lại sau 60 giây."
}
```

| Header | Ý nghĩa |
|---|---|
| `X-RateLimit-Limit` | Giới hạn tối đa |
| `X-RateLimit-Remaining` | Còn lại bao nhiêu |
| `X-RateLimit-Reset` | Unix timestamp khi reset |
| `Retry-After` | Giây cần chờ trước khi thử lại |

Client nên đọc `Retry-After` để implement **exponential backoff** thay vì hammer server.

---

## 8. Thực hành tốt (Best Practices)

**Thiết kế giới hạn hợp lý**: đừng quá chặt (làm khó user thật) hay quá lỏng (không bảo vệ được). Dựa trên data thực tế về traffic pattern.

**Phân biệt user thật và bot**: user thật hiếm khi gửi quá 10 req/giây, bot có thể gửi hàng nghìn. Giới hạn theo tier phù hợp.

**Whitelist IP nội bộ**: các service nội bộ, CI/CD, monitoring không nên bị rate limit.

**Giám sát và alert**: theo dõi tỉ lệ 429, bắt được pattern tấn công sớm.

**Graceful degradation**: khi Redis (centralized store) gặp sự cố, quyết định rõ ràng: **fail open** (cho qua hết) hay **fail closed** (chặn hết). Thông thường chọn fail open để không ảnh hưởng user thật.

**Communicate rõ ràng**: document giới hạn trong API docs, trả về headers đầy đủ để developer biết cách xử lý.

---

## 9. Rate Limit vs Throttling

Hai khái niệm này thường dùng lẫn nhau nhưng có điểm khác biệt:

| | Rate Limit | Throttling |
|---|---|---|
| **Hành động** | Chặn request (từ chối) | Làm chậm request (delay) |
| **Response** | 429, trả về ngay | Delay rồi xử lý |
| **Trải nghiệm** | Client phải retry | Client chờ lâu hơn |
| **Dùng khi** | Bảo vệ khỏi abuse | Đảm bảo QoS fair |

---

## 10. Ví dụ thực tế

### GitHub API
- Unauthenticated: 60 req/giờ per IP.
- Authenticated: 5000 req/giờ per token.
- Search API: 30 req/phút (chặt hơn vì tốn tài nguyên).

### Stripe API
- 100 req/giây per secret key.
- Dùng Token Bucket → cho phép burst ngắn hạn.

### Twitter/X API
- Free tier: 1500 tweet writes/tháng.
- Granular theo endpoint và tier.

---

## 11. Các bài toán thực tế thường gặp trong business

Đây là những tình huống mà gần như mọi hệ thống production đều sẽ gặp phải.

---

### Bài toán 1: OTP / SMS bị spam → chi phí nổ

**Tình huống**: Hệ thống có form "Gửi OTP". Kẻ xấu (hoặc script) gọi API này liên tục với số điện thoại của người khác — mỗi lần gọi tốn tiền SMS, nạn nhân bị spam tin nhắn.

**Vấn đề đặc thù**: không thể chỉ rate limit theo IP vì attacker dùng nhiều IP. Không thể chỉ rate limit theo số điện thoại vì có thể dùng số rác.

**Giải pháp kết hợp nhiều tầng**:

```
Tầng 1 — Per IP:          tối đa 5 request OTP / 10 phút / IP
Tầng 2 — Per số điện thoại: tối đa 5 OTP / ngày / số
Tầng 3 — Per user account: tối đa 10 OTP / ngày / tài khoản
Tầng 4 — Global:          alert nếu tổng OTP/phút vượt ngưỡng bình thường
```

**Thêm**: cooling-off period — sau khi gửi OTP, bắt buộc chờ 60 giây trước khi gửi lại (bất kể limit). Hiển thị countdown timer trên UI để giảm áp lực người dùng spam nút.

---

### Bài toán 2: Flash sale — hàng nghìn người checkout cùng lúc

**Tình huống**: Shopee, Tiki, Lazada mở flash sale 12h trưa. Hàng chục nghìn user bấm "Mua ngay" đồng thời. Database không thể handle, server sập.

**Vấn đề đặc thù**: đây không phải tấn công — đây là user thật. Không thể chặn hết, phải phục vụ công bằng.

**Giải pháp — Queue + Rate Limit kết hợp**:

```
User bấm "Mua ngay"
    ↓
Rate Limit: tối đa 1 checkout request / 5 giây / user (chống double-click, bot)
    ↓
Nếu vượt limit → 429: "Bạn đang thao tác quá nhanh"
    ↓
Nếu OK → đẩy vào hàng đợi (Queue)
    ↓
Worker xử lý tuần tự: 500 đơn/giây (vừa với capacity DB)
    ↓
User nhận kết quả qua WebSocket / polling
```

**Điểm quan trọng**: rate limit ở đây không phải để từ chối user — mà để bảo vệ queue khỏi bị flood. Kết hợp với **optimistic lock** ở DB để tránh oversell.

---

### Bài toán 3: Login bị brute force

**Tình huống**: Attacker thử lần lượt hàng triệu mật khẩu vào tài khoản `admin@company.com`.

**Vấn đề đặc thù**: IP rotation — mỗi vài lần thử là đổi IP. Rate limit theo IP đơn thuần không đủ.

**Giải pháp nhiều lớp**:

```
Lớp 1 — Per (IP + username):  5 lần sai → block 15 phút
Lớp 2 — Per username:         20 lần sai trong 1 giờ → lock tài khoản tạm, gửi email cảnh báo
Lớp 3 — Per IP:               50 lần sai nhiều tài khoản → block IP 1 giờ
Lớp 4 — Behavioral:           pattern lạ (tốc độ đều, không có mouse movement) → yêu cầu CAPTCHA
```

**Progressive penalty** — hình phạt tăng dần:

| Lần sai | Hành động |
|---|---|
| 1–3 | Thông báo thông thường |
| 4–5 | Thêm delay 2 giây |
| 6–10 | Yêu cầu CAPTCHA |
| > 10 | Block IP + alert security team |

---

### Bài toán 4: API public bị một client "ngốn" hết quota

**Tình huống**: Bạn cung cấp API cho nhiều đối tác (B2B SaaS). Một đối tác chạy cron job lỗi, gọi API của bạn hàng nghìn lần/phút, làm các đối tác khác bị chậm.

**Vấn đề đặc thù**: đây là client hợp lệ (có API key thật), không phải tấn công — họ chỉ bị bug.

**Giải pháp — Rate limit theo tier + isolation**:

```
Free tier:      100 req/phút
Starter tier:   1.000 req/phút
Business tier:  10.000 req/phút
Enterprise:     custom SLA
```

**Quan trọng hơn**: mỗi tenant chạy trong "lane" độc lập — tenant A dù hết quota cũng không ảnh hưởng tenant B. Dùng **separate Redis key space** hoặc **separate queue per tenant**.

Khi một client liên tục hit limit, tự động gửi email cảnh báo thay vì chỉ trả 429 — đây thường là bug của họ, cần được thông báo để fix.

---

### Bài toán 5: Search / autocomplete gọi DB quá nhiều

**Tình huống**: Ô tìm kiếm gọi API mỗi khi user gõ một ký tự. User gõ "iphone 15 pro max" = 16 API calls. Hệ thống có 10.000 user đang gõ cùng lúc = 160.000 req/giây vào DB tìm kiếm.

**Vấn đề đặc thù**: đây là feature bình thường, không phải abuse. Nhưng volume quá lớn.

**Giải pháp — Debounce phía client + rate limit phía server**:

```
Client side:   debounce 300ms — chỉ gọi API sau khi user ngừng gõ 300ms
Server side:   5 search req/giây/user — nếu vượt → trả cache result cũ
Cache layer:   cache kết quả search 30 giây (ElasticSearch/Redis)
```

Rate limit ở đây đóng vai trò **safety net** — nếu client debounce bị bypass (script, API call trực tiếp), server vẫn được bảo vệ.

---

### Bài toán 6: Webhook delivery thất bại → retry bão hòa

**Tình huống**: Bạn gửi webhook đến hệ thống của partner khi có event. Partner bị downtime, webhook thất bại. Hệ thống retry ngay lập tức → gửi hàng nghìn request vào server đang chết của partner → làm họ càng khó recover hơn.

**Vấn đề đặc thù**: rate limit ở đây là để **bảo vệ bên nhận**, không phải bên gửi.

**Giải pháp — Exponential backoff + circuit breaker**:

```
Lần 1 thất bại → retry sau 10 giây
Lần 2 thất bại → retry sau 30 giây
Lần 3 thất bại → retry sau 2 phút
Lần 4 thất bại → retry sau 10 phút
Lần 5 thất bại → retry sau 1 giờ
Sau 24 giờ thất bại liên tục → dead letter queue + alert
```

**Circuit breaker**: nếu tỉ lệ thất bại của một endpoint > 50% trong 5 phút → ngừng gửi hoàn toàn 10 phút, sau đó thử lại với 1 request thăm dò trước.

---

### Bài toán 7: Internal service gọi nhau quá nhiều (microservices)

**Tình huống**: Service A gọi Service B để lấy user profile. Khi có traffic spike, Service A spawn hàng nghìn goroutine/thread gọi B cùng lúc → B quá tải → B chậm → A timeout → A retry → B càng quá tải hơn. Đây là **cascading failure**.

**Vấn đề đặc thù**: đây không phải attacker — là chính hệ thống của mình tự đánh nhau.

**Giải pháp — Rate limit internal + bulkhead**:

```
Service A chỉ được giữ tối đa 50 concurrent connections đến B (connection pool limit)
Nếu queue đầy → fail fast, trả lỗi ngay thay vì chờ
Service B tự rate limit inbound: tối đa 1000 req/giây từ bất kỳ caller nào
```

**Bulkhead pattern**: mỗi upstream caller có connection pool riêng biệt — Service A dùng pool riêng, Service C dùng pool riêng. A bị sự cố không ảnh hưởng pool của C.

---

### Tổng kết các pattern

| Bài toán | Chiều giới hạn chính | Thuật toán phù hợp | Điểm đặc biệt |
|---|---|---|---|
| OTP spam | Per phone + Per IP | Fixed Window | Cooling-off period |
| Flash sale | Per user | Token Bucket | Kết hợp Queue |
| Brute force login | Per (IP+username) | Fixed Window | Progressive penalty |
| API B2B quota | Per API key / tenant | Token Bucket | Tenant isolation |
| Search autocomplete | Per user | Sliding Window | Debounce client-side |
| Webhook retry | Per endpoint | N/A | Exponential backoff |
| Internal microservice | Per service caller | Leaky Bucket | Bulkhead pattern |

---



Rate limit là công cụ **không thể thiếu** trong bất kỳ hệ thống production nào. Điểm then chốt cần nhớ:

1. **Tại sao**: bảo vệ server, chống tấn công, đảm bảo công bằng.
2. **Thuật toán**: Token Bucket và Sliding Window Counter là lựa chọn tốt nhất cho đại đa số use case.
3. **Triển khai**: Redis làm centralized store, NGINX/API Gateway làm điểm thực thi.
4. **Response**: HTTP 429 + đủ headers để client tự xử lý.
5. **Giám sát**: luôn theo dõi tỉ lệ 429 để phát hiện sớm bất thường.

Rate limit không phải để làm khó user — mà để đảm bảo **mọi user đều có trải nghiệm tốt** ngay cả khi hệ thống đang chịu áp lực.

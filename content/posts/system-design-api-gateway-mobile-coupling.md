+++
date = '2026-06-11'
draft = false
title = 'System Design 1/30: khi mobile app gọi quá nhiều backend service'
summary = "Phân tích bài Daily.dev về mobile app bị coupling với nhiều backend service: API Gateway, BFF, Load Balancer hay GraphQL Federation?"
tags = ['System Design', 'Backend', 'API Gateway', 'Microservices']
+++

# System Design 1/30: khi mobile app gọi quá nhiều backend service

Bài đầu tiên trong thử thách system design đặt ra một tình huống rất thực tế: mobile app hiện gọi trực tiếp 3 backend service:

- `UserService`
- `OrderService`
- `PaymentService`

Sprint tới sẽ có thêm `NotificationService`. Mobile team bắt đầu quá tải vì mỗi service mới lại kéo theo domain mới cần whitelist, auth scheme mới cần xử lý, error shape mới cần parse, logging/observability phân tán hơn, và logic routing bị đẩy ra client.

Câu hỏi là: nên chọn gì để giảm coupling trước khi service thứ 4 xuất hiện?

- A) API Gateway
- B) BFF - Backend for Frontend
- C) Load Balancer
- D) GraphQL Federation

Đáp án hợp lý nhất cho bài này là **A: API Gateway**.

## Vì sao API Gateway thắng?

Vấn đề gốc không phải là app cần payload đẹp hơn. Vấn đề gốc là client đang biết quá nhiều về topology backend.

Mobile app không nên cần biết `users.api.com`, `orders.api.com`, `payments.api.com`, rồi tuần sau biết thêm `notifications.api.com`. Nó nên gọi một domain ổn định, ví dụ:

```txt
api.yourapp.com/users
api.yourapp.com/orders
api.yourapp.com/payments
api.yourapp.com/notifications
```

API Gateway đứng trước các service và xử lý phần application-layer routing. Khi `NotificationService` ra mắt, backend thêm route ở gateway; mobile app không phải ship bản mới chỉ để biết thêm một domain.

Gateway cũng gom những thứ nên được quản lý tập trung:

- AuthN/AuthZ
- TLS termination
- Rate limiting
- Request logging
- Error normalization
- API versioning
- Routing theo path/header/version

Đó là lý do comment được upvote nhiều nhất chọn A: một entry point cho frontend, che giấu internal services, giảm routing/auth/security/logging rải rác ở mobile client.

## BFF đúng, nhưng không đúng bài này

BFF là pattern thật, không phải đáp án vô lý. Đây cũng là bẫy của câu hỏi.

BFF giải quyết bài toán **data shape theo từng client**. Ví dụ mobile cần payload mỏng, web cần payload giàu hơn, TV app cần layout khác. Khi đó BFF gom data từ nhiều service và trả về response đúng nhu cầu từng client.

Nhưng trong đề bài này, mobile team chưa kêu vì payload shape. Họ kêu vì:

- nhiều domain
- nhiều auth scheme
- nhiều error format
- thêm service là client lại phải đổi

Nếu dựng BFF chỉ để giấu 4 URL, ta đang thêm một service do team tự vận hành mà chưa giải quyết đúng bản chất bằng pattern nhẹ hơn. Có thể sau này đặt BFF phía sau gateway, nhưng bước đầu tiên nên là gateway để cắt coupling ở tầng transport/routing.

## Load Balancer sai ở đâu?

Load Balancer chủ yếu phân phối traffic giữa nhiều instance tương đương của cùng một service.

Ví dụ:

```txt
OrderService instance 1
OrderService instance 2
OrderService instance 3
```

Load balancer chọn instance nào nhận request. Nó không sinh ra contract API thống nhất, không normalize error, không xử lý auth policy theo API product, và không tự biến 4 service khác nhau thành một backend API có governance.

Một comment có chia sẻ case dùng Route 53 + Load Balancer + path routing để đi tới target group khác nhau. Cách đó thực tế có thể chạy được, nhất là trên AWS, nhưng bản chất lúc này load balancer đang bị dùng như một reverse proxy/gateway nhẹ. Nếu cần quản lý auth, versioning, rate limit, observability và policy nghiêm túc, đó vẫn là đất của API Gateway.

## GraphQL Federation sai ở đâu?

GraphQL Federation giải quyết bài toán **schema unification**: client query một graph thống nhất, còn phía sau có nhiều subgraph/service.

Nó hợp khi team thật sự cần graph data model, client hay phải compose dữ liệu từ nhiều domain, và tổ chức đã sẵn sàng vận hành GraphQL gateway/subgraph. Nhưng với bài này, mục tiêu là giảm transport-layer coupling nhanh trước sprint tới.

Nếu chọn GraphQL Federation, team sẽ phải:

- thay đổi contract của nhiều backend service
- expose subgraph/schema
- vận hành federation gateway
- train team về query, resolver, cache, auth, observability kiểu GraphQL

Đó là migration lớn cho một vấn đề có thể giải quyết nhanh hơn bằng API Gateway.

## Tranh luận trong comment

Phần comment khá hay vì không chỉ có đáp án A. Có vài nhóm ý kiến:

Nhóm chọn **API Gateway** lập luận rằng mobile cần một entry point, một auth flow, một error contract. Đây là hướng trực diện nhất với vấn đề domain sprawl và client coupling.

Nhóm hỏi "vì sao các đáp án còn lại sai" làm bài viết đáng giá hơn, vì BFF và GraphQL đều là pattern thật. Sai ở đây không phải vì chúng tệ, mà vì chúng tối ưu cho bài toán khác.

Nhóm chọn **BFF** nhìn từ góc độ client experience: backend nên gánh việc stitch data, frontend tập trung UI/UX. Ý này đúng trong nhiều sản phẩm, nhưng nó giả định bài toán chính là aggregation/data composition. Đề bài lại nhấn mạnh domain, auth và error shape.

Nhóm đề xuất **Gateway + BFF** cũng hợp lý ở kiến trúc lớn hơn: gateway làm entry point/policy/routing, BFF làm aggregation cho từng client. Nhưng nếu câu hỏi bắt chọn một bước trước sprint tới, gateway là bước nền.

Nhóm dùng **Load Balancer/path routing** phản ánh thực tế vận hành: nhiều team tận dụng hạ tầng sẵn có thay vì thêm AWS API Gateway/Kong/Apigee/Nginx gateway riêng. Đây là trade-off đáng cân nhắc, nhưng nếu path routing bắt đầu gánh auth, versioning, error normalization, rate limit, logging policy, thì ta đang tiến dần về API Gateway dù tên resource là gì.

## Kết luận

Bài này không hỏi "pattern nào xịn nhất". Nó hỏi "pattern nào giải quyết đúng coupling đang làm mobile team chết đuối".

Kết luận của mình:

**Chọn API Gateway trước.**

Lý do:

- Cắt domain sprawl cho mobile app.
- Che giấu internal service topology.
- Chuẩn hoá auth, routing, logging, rate limit, error contract.
- Thêm service mới không bắt client biết thêm host mới.
- Ít migration hơn GraphQL Federation.
- Đúng bài toán hơn BFF nếu pain chính là transport/routing coupling.

BFF và GraphQL Federation vẫn có chỗ đứng, nhưng nên xuất hiện khi bài toán thật sự là data composition, client-specific response, hoặc unified graph. Load Balancer thì giải quyết scaling/traffic distribution, không phải API contract.

Một câu nhớ nhanh:

> Nếu client đang biết quá nhiều service: đặt API Gateway. Nếu client cần data shape riêng: thêm BFF. Nếu client cần query graph thống nhất: cân nhắc GraphQL Federation. Nếu service cần scale nhiều instance: dùng Load Balancer.

## Nguồn đọc

- [Daily.dev: 1/30 Days System Design Question](https://app.daily.dev/posts/1-30-days-system-design-question-mxghviyp7)
- Comment thread Daily.dev của bài viết, gồm các ý kiến chọn API Gateway, phản biện BFF, Load Balancer, GraphQL Federation, và đề xuất kết hợp Gateway + BFF.

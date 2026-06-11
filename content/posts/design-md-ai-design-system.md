+++
date = '2026-06-11'
draft = false
title = 'DESIGN.md: design system có nên trở thành hợp đồng cho AI?'
summary = "Ghi chú ngày đầu tiên tìm hiểu design system: DESIGN.md của Google Stitch, tranh luận từ Daily.dev, và kết luận thực dụng cho team dev."
tags = ['Design System', 'AI', 'Frontend', 'Workflow']
+++

# DESIGN.md: design system có nên trở thành hợp đồng cho AI?

Ngày đầu tiên tìm hiểu về design system, mình đọc bài Daily.dev về `DESIGN.md` - một đề xuất của Google Stitch cho việc đóng gói design system thành một file Markdown mà AI agent có thể đọc được.

Ý tưởng cốt lõi rất đơn giản: thay vì mỗi lần dùng AI lại phải nhắc "dùng màu này", "spacing như này", "button theo style này", team đặt những quy tắc đó vào một file có cấu trúc. File này vừa có phần machine-readable như token màu sắc, typography, spacing, vừa có phần human-readable như design intent, usage rule, accessibility constraint và anti-pattern.

Nói cách khác, `DESIGN.md` muốn trở thành một "design contract" nằm trong workflow của sản phẩm. AI không chỉ nhìn screenshot rồi đoán style, cũng không chỉ đọc token rồi ghép UI. Nó đọc cả lý do đằng sau design system.

## Bài viết nói gì?

Bài Daily.dev tóm tắt rằng Google Stitch giới thiệu `DESIGN.md` như một format open-source để chuẩn hoá cách chia sẻ UI design system cho AI tool. File này có thể mô tả token, ngữ cảnh thiết kế, validation như accessibility check, và có thể kết nối với workflow như Tailwind hoặc CI pipeline.

Nếu làm được, lợi ích rõ nhất là giảm lặp lại giải thích giữa designer, developer và AI. Team có một nơi để nói: "đây là cách UI của mình hoạt động". Sau đó tool nào sinh UI cũng phải đọc và tôn trọng nguồn sự thật đó.

## Cuộc tranh luận trong comment

Comment trên Daily.dev chia thành vài luồng khá rõ.

Luồng ủng hộ cho rằng đây là bước đúng hướng. Một comment nói nếu format này "stick", nó có thể chuẩn hoá design intent giống cách OpenAPI chuẩn hoá API. Điểm thắng không nằm ở việc AI vẽ màn hình đẹp hơn, mà nằm ở consistency, validation và giảm ambiguity giữa các team/tool.

Một luồng thực dụng hơn nói họ đã làm điều tương tự: đặt quy tắc design vào các file `.md`, rồi link từ file instruction của AI agent. Điều này cho thấy nhu cầu thật sự đã tồn tại trước khi Google đặt tên cho nó. `DESIGN.md` không phải phát minh mới hoàn toàn; nó là cách đóng gói một practice đang nổi lên trong AI coding workflow.

Luồng hoài nghi tập trung vào chuyện "lại thêm một file Markdown nữa". Có comment đưa ra so sánh với `AGENTS.md` và `CLAUDE.md`: ý tưởng hay, nhưng để rồi có thể nằm trong đống "gần như là standard". Đây là phần tranh luận quan trọng nhất, vì design system chết không phải do thiếu tài liệu, mà do tài liệu không được maintain.

Một số comment khác hỏi thẳng: nó có thay Figma không? Câu trả lời của mình là không. `DESIGN.md` không nên thay Figma. Figma vẫn là nơi designer explore, prototype, review visual. `DESIGN.md` nên là lớp contract trích ra từ Figma, codebase và design guideline để AI/dev tool đọc được.

Cũng có luồng comment nhìn nó như một cơ hội cho creator/agency: mỗi client có brand rule khác nhau, nên một file gốc để AI đọc sẽ giúp sinh UI đúng brand hơn. Đây là use case rất thực tế, nhất là khi làm landing page, dashboard, component library cho nhiều khách hàng.

## Điểm đáng tranh luận thật sự

Tranh luận không phải là "Markdown có đủ mạnh không?". Markdown chỉ là container.

Tranh luận đúng hơn là:

- Ai là source of truth: Figma, codebase, hay `DESIGN.md`?
- File này được sinh từ đâu, hay viết tay?
- Ai review thay đổi: designer, frontend lead, hay cả hai?
- CI có validate token, contrast, component usage không?
- Khi design system đổi, `DESIGN.md` có được update cùng lúc không?

Nếu không trả lời những câu này, `DESIGN.md` sẽ thành một file doc đẹp lúc ban đầu, rồi stale sau vài sprint. Khi đó AI vẫn đọc nó một cách tự tin và sinh ra UI sai.

## Góc nhìn của mình

Mình nghĩ `DESIGN.md` chỉ có giá trị nếu coi nó như code, không phải như document.

Nghĩa là:

- Nằm trong repo.
- Có owner rõ ràng.
- Thay đổi qua pull request.
- Được review bởi cả design và frontend.
- Có liên kết với design token/component source.
- Có CI check những thứ có thể check được: token tồn tại, contrast, naming, component usage rule.

Nếu làm theo cách này, nó có thể trở thành cầu nối rất mạnh giữa design system và AI coding agent. AI lúc đó không chỉ "tạo UI đẹp", mà tạo UI đúng hệ thống.

## Kết luận

`DESIGN.md` không phải cây đũa thần thay thế designer, Figma hay design system. Nó là một lớp hợp đồng để AI và developer hiểu design system theo cách ổn định, review được, version được.

Kết luận của ngày đầu tiên:

Design system trong kỷ nguyên AI không nên chỉ là Figma library hay bộ token. Nó nên có một artifact đọc được bởi máy, giải thích được bởi người, và sống cùng codebase. `DESIGN.md` là một hướng đi đáng thử, nhưng chỉ thành standard thật nếu nó có governance tốt.

Nếu không có governance, nó sẽ là "một file Markdown nữa".

Nếu có governance, nó có thể là OpenAPI của UI.

## Nguồn đọc

- [Daily.dev: DESIGN.md - UI design systems format shared with AI](https://app.daily.dev/posts/design-md---ui-design-systems-format-shared-with-ai-ciob1x0bb)
- [Google Stitch Docs: What is DESIGN.md?](https://stitch.withgoogle.com/docs/design-md/overview)
- Comment thread Daily.dev của bài viết, gồm các ý kiến về chuẩn hoá, nguy cơ "almost a standard", thay thế Figma, và workflow AI instruction file.

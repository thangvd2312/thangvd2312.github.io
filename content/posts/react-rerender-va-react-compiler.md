+++
date = '2026-04-09'
draft = false
title = 'React Re-render và React Compiler'
summary = "Hiểu rõ cơ chế re-render trong React, cách memoization hoạt động, và vì sao React Compiler giúp code sạch hơn mà vẫn tối ưu hiệu năng."
tags = ['React', 'Frontend', 'Performance']
+++

# React Re-render và React Compiler

Trong React, hiệu năng thường đi xuống không phải vì logic nặng, mà vì component re-render nhiều hơn mức cần thiết. Bài này tập trung vào bản chất re-render, khi nào cần tối ưu thủ công, và khi nào để React Compiler tự xử lý.

## 1) Quy tắc nền tảng: Parent render thi con render

Khi component cha re-render, toàn bộ component con của nó sẽ được gọi lại.  
Điều này xảy ra ngay cả khi props của con không đổi.

```tsx
const Parent = () => {
  const [count, setCount] = useState(0)

  return (
    <div>
      <p>count: {count}</p>
      <button onClick={() => setCount((c) => c + 1)}>Tang</button>
      <Child name="Thang" />
    </div>
  )
}

const Child = ({ name }: { name: string }) => {
  console.log("Child rendered")
  return <p>Hello {name}</p>
}
```

`Child` không nhận `count`, nhưng vẫn re-render mỗi lần bấm nút vì `Parent` đã re-render.

## 2) Bộ 3 tối ưu re-render thủ công

### `React.memo`

`React.memo` bọc component để React so sánh props trước khi render lại.

- Props không đổi: bỏ qua render
- Props đổi: render bình thường

### `useMemo`

`useMemo` giữ lại kết quả tính toán để tránh tạo object/array mới ở mỗi lần render.

```tsx
const items = useMemo(() => data.map((x) => x.name), [data])
```

### `useCallback`

`useCallback` giữ ổn định reference của function giữa các lần render.

```tsx
const handleClick = useCallback(() => doSomething(id), [id])
```

## 3) Điểm hay bị hiểu sai

`useMemo`/`useCallback` chỉ giúp giữ reference ổn định.  
`React.memo` mới là thứ quyết định có skip render hay không.

Nên để tối ưu render cho component con qua props, thường phải kết hợp cả hai phía:

- Cha: ổn định props (value/function)
- Con: dùng `React.memo` để kiểm tra props

## 4) Vì sao React không tự memo mọi thứ từ đầu?

Lý do chính:

- Memo có chi phí so sánh props, không phải miễn phí.
- Không phải component nào cũng hưởng lợi từ memoization.
- Tối ưu sai dependency có thể gây bug stale data khó debug.

React ưu tiên tính đúng trước, sau đó mới tối ưu hiệu năng.

## 5) React Compiler giải quyết bài toán này như thế nào?

React Compiler là công cụ phân tích component ở build time và tự thêm cache/memoization khi an toàn.

Bạn viết code đơn giản:

```tsx
const Sidebar = ({ messages, send }) => {
  const items = messages.map((m) => m.text)
  const onSend = async (content: string) => {
    await send(content)
  }

  return <Child items={items} onSend={onSend} />
}
```

Compiler sẽ sinh code tối ưu tương đương với việc bạn tự thêm memoization.

## 6) Build time vs runtime

React Compiler chạy lúc build (`hugo` không liên quan; React app build bằng bundler như Vite/Webpack).  
Tức là tối ưu được áp dụng trước khi code vào browser.

## 7) Khi nào nên dùng cách nào?

- Dự án React 19 + Compiler: ưu tiên code đơn giản, để compiler lo memoization.
- Dự án React 18 trở xuống: tối ưu thủ công ở những component nặng hoặc render dày đặc.
- Component nhỏ, rẻ: không cần memo, tránh tối ưu sớm.

## 8) Checklist ngắn để giảm re-render

- Đo trước khi tối ưu (React DevTools Profiler)
- Tránh tạo object/function inline nếu truyền sâu xuống tree
- Tách component lớn thành các phần nhỏ, trách nhiệm rõ ràng
- Chỉ memo ở nơi thật sự có lợi

## Kết luận

Hiểu re-render giúp bạn chọn đúng chiến lược: tối ưu thủ công khi cần, và tận dụng React Compiler khi stack hỗ trợ.  
Mục tiêu cuối cùng là code dễ đọc, đúng hành vi, và nhanh đủ cho người dùng thực tế.

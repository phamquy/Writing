# Compound Assignment Operators

Giống như trong C, Swift cung cấp toán tử tính và gán giá trị (_compound assignment operators_), toán tử này ghép một toán tử gán (assigment) với môt toán tử khác. Ví dụ như toán tử cộng và gán (addition assignment operator) `+=`:

```swift
var a = 1
a += 2
// a now equals to 3
```

Biểu thức `a += 2` là cách viết ngắn gọn cho `a = a + 2`. Toán tử ghép `+=` gộp phép cộng và gán giá trị vào một toán tử và thực hiện cả hai đồng thời.

> CHÚ Ý
>
> Toán tử ghép không trả về giá tri. Ví dụ, ta không thể viết let `b = a += 2`

Để xem danh sách các toán tử ghép có trong Swift, tham khảo _Swift Standard Library Operator Reference_.

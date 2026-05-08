# Comments

Sử dụng comment để thêm các dòng chú thich, đó la nhừng câu không phải câu lệnh của chương trình. Những đoạn comment bị trình biên dịch Swift bỏ qua khi biên dịch.

Comments trong swift rất giống với comment trong C. Dòng comment đơn lẻ được bắt đầu với hai dấu gạch xiên(forward-slashes '//' )

```swift
// This is a comment
```

Comment nhiều dòng bắt đầu với `/*` và kết thức vói `*/`

```swift
/* this is also a comment,
but written over multiple lines */
```

Không giống như comment nhiều dòng trong C, comment nhiều dòng trong Swift có thể được đăt trong đoạn comment khác.

```swift
/* this is the start of the first multiline comment
/* this is the second, nested multiline comment */
this is the end of the first multiline comment */
```

Việc cho phép đặt đoạn comment trong đoạn comment khác cho phép ta comment một đoạn code dài một cách dễ dàng kể cả khi trong đoạn code đó có chứ đoạn comment khác. Điều này là không thể trong C.

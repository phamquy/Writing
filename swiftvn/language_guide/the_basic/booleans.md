# Booleans

Swift có kiểu _Boolean_ là `Bool`. Giá trị của boolean có chỉ có thể là `true` và `false`.

```swift
let orangesAreOrange = true
let turnipsAreDelicious = false
```

Kiểu của `orangeAreOrange` và `turnipAreDelicious` được nội suy là kiêu Boolean vì chúng được khởi tạo với giá trị của kiểu `Bool`.

Boolean được sử dụng khi làm việc với các câu lệnh điểu kiện như lệnh rẽ nhành `if`

```swift
if turnipsAreDelicious {
    print("Mm, tasty turnips!");
} else {
    print("Eww, turnips are horrible");
}
```

Câu lệnh điều kiện như `if` được giải thích kĩ trong phần `Control Flow` (TODO: need reference)

An toàn kiểu trong swift không cho phép dùng các kiểu khác để thay thế cho kiểu boolean (như trong C hay Java). Ví dụ trong đoạn code sau, swift sẽ báo lỗi biên dịch

```swift
let i = 1
if i {
    // Câu lệnh trên sẽ gây lỗi biên dịch
}
```

Tuy nhiên đoạn lệnh sau sẽ dc biên dịch bình thường

```swift
let i = 1
if i == 1 {
    // Đoạn lệnh trên được dịch thành công
}
```

Kểt quả của phép so sánh `i==1` là một giá trị kiểu boolean, do đó đoạn lênh trên sẽ được dịch thành công. Cũng giống như các ví dụ khác về an toàn kiểu, cách tiếp cận này của swift ngăn ngừa các lỗi do vô tình dùng nhầm kiểu, dồng thời làm cho code rõ nghĩa hơn.

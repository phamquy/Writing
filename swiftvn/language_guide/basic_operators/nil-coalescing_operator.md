# Nil-Coalescing Operator

Toán tử kiểm tra nil-coalescing được viết như sau `a ?? b` thực hiện bóc giá trị optional a và kiểm tra nếu no chứa giá trị thì trả về `a` nếu không thì trả về `b`. Toán hạng `a` luôn là kiểu giá trị optional, toán hạng `b` phải cùng kiểu được chứa trong optional `a`.

nil-coalescing là dạng viết tắt của dòng lệnh sau:

```swift
a != nil ? a! : b
```

Dòng code trên dùng toán tử điều kiện để kiểm tra giá trị của `a`, và trả về giá trị `a` nều có giá trị và `b` nếu không chứa giá trị. Với cú pháp nil-coalescing, dòng lệnh trên được viết một các ngắn gọn và dễ nhìn hơn.

> NOTE
>
> Nếu giá trị của `a` không phải là `nil`, thi giá trị của `b` không được tính (Tức nếu `b` là một biểu thức thì nó sẽ bị bỏ qua). Tính năng này được gọi là `short-circuit evaluation`

Trong đoạn ví dụ dưới đây nil-coalescing được dùng để chọn giữa haiv giá trị: tên màu, và một giá trị tên màu kiểu optional

```swift
let defaultColorName = "red"
var userDefinedColorName : String?

var colorNameToUse = userDefinedColorName ?? defaultColorName;

// userDefinedColorName co gia tri nil, nên colorNameToUse sẽ được gán tên là "red"
```

Biến `userDefinedColorName` được định nghĩa là kiểu optional, với giá trị mặc định là `nil`. Do nó là kiểu optional nên ta có thể dùng nil-coalescing để kiểm tra giá trị trong `userDefinedColorName` và trả về gía trị phù hợp. Trong trường hợp này, do `userDefinedColorName` có giá trị `nil` nên `colorNameToUse` sẽ được gán giá trị của `defaultColorName`, tức là "red".

Nếu ta gán giá trị nào đó khác `nil` cho `userDefinedColorName` trong trường hợp này thì `colorNameToUse` sẽ được gán giá trị đó giống như trong ví dụ sau:

```swift
userDefinedColorName = "green"
colorNameToUse = userDefinedColorName ?? defaultColorName
// Bây giờ colorNameToUse sẽ có giá trị là "green"
```

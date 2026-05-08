# Control Flow

`switch` và `if` để tạo các rẽ nhánh có điều kiện, `for-in` ,`for`, `while` và `repeat-while` để tạo các vòng lặp. Dấu ngoắc quanh đoạn code điều kiện là không bắt buộc, nhưng bắt buộc phải có `{...}` bao quanh đoan mã thực hiện.

```swift
let individualScores = [75, 43, 103, 87, 12]
var teamScore = 0
for score in individualScores {
    if score > 50 {
        teamScore += 3
    } else {
        teamScore += 1
    }
}
print(teamScore)
```

Trong cấu trúc điều kiện `ìf`, giá trị của điều kiện phải có kiều `Boolean`, các giá trị không phải `Boolean` khi dùng cho điều kiện `ìf` tạo ra lỗi khi biên dịch

> **NOTE**
>
> Trong C/C++ và ObjectiveC, các giá trị của các kiều khác `Boolean` có thể được dùng như điều kiện, ví dụ:
>
> ```swift
> if ( numberOfItem ) {
>    ...
> }
> ```

`ìf` và `let` có thể được dùng trong cùng một dòng lệnh để làm nên cú pháp tắt kiểm tra giá trị kiểu `optional`. Một giá trị được gọi là `optional` tức là nó có thể có giá trị hoặc `nil` nếu không chứa giá trị gì. Dấu `?` đăt sau tên của kiều giá trị chỉ định giá trị của hằng/biến là `optional`

```swift
var optionalString: String? = "Hello"
print(optionalString == nil)

var optionalName: String? = "John Appleseed"
var greeting = "Hello!"
if let name = optionalName {
    greeting = "Hello, \(name)"
}
```

> **THỤC HÀNH**
>
> Đặt giá trị cho `optionalName` thành `nil` thêm mã xử lí cho nhánh `else`.

Nếu giá tri `optional` là `nil`, đoạn mã trong nhánh `ìf` bị bỏ qua, nếu không giá trị thật của biến sẽ được gán cho hằng số đặt sau `let`. Trong đoạn mã trên, `name` sẽ được gán gia trị `John Appleseed`.

Một cách khác để xử lí giá trị `optional` là sử dụng toán tử `??`. Nếu giá trị không tồn tại, thì một giá trị mặc định khác sẽ được dùng:

```swift
let nickName: String? = nil
let fullName: String = "John Appleseed"
let informalGreeting = "Hi \(nickName ?? fullName)
```

Trong đoạn mã trên, `informalGreeting` sẽ được gán chuỗi `"John Appleseed"` do `nickName` không chứa giá trị (hay chứa giá trị `nil`)

Lệnh `switch` trong Swift khác so vói trong C/C++ hay Objective C, trong swift `switch` có thể nhận bất cứ kiểu giá trị nào

```swift
let vegetable = "red pepper"
switch vegetable {
case "celery":
    print("Add some raisins and make ants on a log.")
case "cucumber", "watercress":
    print("That would make a good tea sandwich.")
case let x where x.hasSuffix("pepper"):
    print("Is it a spicy \(x)?")
default:
    print("Everything tastes good in soup.")
}
```

> **THỰC HÀNH**
>
> Bỏ nhánh `default` trong lệnh trên để xem lỗi gì xảy ra.

Chú ý nhánh lệnh có chứa `let`, `let` có thể được dùng trong nhánh của `switch` để nhận giá trị thoả mãn điều kiện của nhánh đó. Trong ví dụ trên, `x` sẽ nhận giá trị `"red pepper"`

Một khác biệt nữa của lệnh `swicth` trong `swift` là sau khi thực hiện lệnh của môth nhánh, chương trình sẽ thoát khỏi cấu trúc `switch` và thục hiện lệnh tiếp theo `switch` (nó không tràn qua thực hiện lệnh trong nhánh tiếo theo như trong C/C++). Do đó không cần phải co `break` ở cuối mỗi nhánh giông như trong các ngôn ngứ lập trình khác.

Có thể dung `for-in` để duyệt các cặp `key:value` trong dictionary. Dictionaty là một tập hợp không được xắp xếp, do đó các phần tử của nó đươc duyệt qua theo một thứ tư ngẫu nhiên.

```swift
let interestingNumbers = [
    "Prime": [2, 3, 5, 7, 11, 13],
    "Fibonacci": [1, 1, 2, 3, 5, 8],
    "Square": [1, 4, 9, 16, 25],
]
var largest = 0
for (kind, numbers) in interestingNumbers {
    for number in numbers {
        if number > largest {
            largest = number
        }
    }
}
print(largest)
```

Vòng lăp `ưhile` dùng để thực hiện lap lại một đoạn mà (không giới hạn số lần) cho đến khi điều kiện của nó được thoả mãn:

```swift
var n = 2
while n < 100 {
    n = n * 2
}
print(n)

var m = 2
repeat {
    m = m * 2
} while m < 100
print(m)
```

Khi đăt `while` ơ cuối, thì doạn mà trong vòng lặp được đảm bảo chạy it nhát 1 lần.

`index` của vòng lặp `for` có thể đươc tạo ra bằng `..<` hoắc được viết cụ thể. Ví dụ hai vòng lặp sau thực hiện cùng một viẹc giống nhau

```swift
var firstForLoop = 0
for i in 0..<4 {
    firstForLoop += i
}
print(firstForLoop)

var secondForLoop = 0
for var i = 0; i < 4; ++i {
    secondForLoop += i
}
print(secondForLoop)
```

`..<` tao ra mọt dải giá trị không chứa giá trị cận trên. Ví dụ `0..<4` sẽ tạo ra `0 1 2 3`. Để tạo dải trị bao gồm cả cận trên thì dùng `...`. Ví dụ `0...4` tạo ra `0 1 2 3 4`

# Simple Values

Sử dụng `let` de tạo một hằng số (constant) và `var` tạo biến. Giá trị của hằng có thể không xác định lúc biên dịch, nhưng phải được và chỉ được gán giá trị đúng một lần. Hằng số dùng để đặt tên cho một giá trị nào đó và sử dụng lại nhiều lần.

```swift
var myVariable = 12
myVariable = 50
let myConstant = 23
```

Một hằng số hay biến phải có cung kiểu so với dữ liệu được gán. Tuy nhiên, không phải lúc nào kiểu phải được viết rõ ràng. Trình biên dịch có thể tự suy được kiểu của biến/hằng từ giá trị. Trong ví dụ trên, trình biên dịch sẽ suy được kiểu của `myVariable` là `integer` ví giá trị gán ban đầu (giá trị khởi tạo) là một số nguyên `12`.

Nếu giá trị khởi tạo không cung cấp đủ thông tin để suy ra kiểu (hoặc trong trường hợp không có giá trị khởi tạo), kiểu của hằng/biến được khai báo đầy đủ:

```swift
let implicitInteger = 70
let implicitDouble = 70.0
let explicitDouble: Double = 70”
```

> **Thực hành**
>
> Tạo một hằng số với khai báo rõ ràng kiểu `Float` với giá trị khởi tạo `4`

Kiểu của giá trị không bao giờ bị đổi. Trong trường hợp cần phải đổi kiểu của giá trị, cần phải tạo một biên/hằng số mới

```swift
let label = "the width is"
let width = 24
let widthLabel = label + String(width
```

> **Thực hành**
>
> Thử bỏ `String` trong đoạn code trên để xem trình biên dịch thông báo gì

Trong Swift, có môt cách đơn giản hơn dể ghép một giá trị vào môt `String` bằng cú pháp `\()`. Ví dụ

```swift
let apples = 3
let orange = 2
let appleSummary = "Tôi có \(apples) apples"
let fruitSummary = "Tôi có \(appleSummary + fruitSummary) quả"
```

> **Thực hành**
>
> Dùng `\()` để ghép một số `float` vào một `string`

Tạo một mảng giá trị (array) hoặc mội `dictionary` bằng cú pháp `[]`, truy xuất giá trị trong bắng cách viết chỉ số `index` hoắc từ khoá `key` bên trong `[]`

```swift
var shoppingList = ["catfish", "water", "tulips", "blue paint"]
shoppingList[1] = "bottle of water"
var occupations = [
    "Malcolm": "Captain",
    "Kaylee": "Mechanic",
]
occupations["Jayne"] = "Public Relations”
```

Để tạo một array hoặc dictionary rỗng, dúng cú pháp khởi tạo:

```objectivec
let emptyArrray = [String]()
let emptyDictionary = [String:Float]()
```

Nếu kiểu của phần tử trong array hoặc dictionary có thể được suy từ giá trị, thì các khai báo kiểu có thể được bỏ qua - ví dụ khi truyển giá trị cho một tham số của hàm, khi đó thông tin về kiểu được suy ra từ khai bào của hàm.

```swift
var shoppingList = []
var occupation = [:]
```

# Function and Closures

Function được khai báo bằng từ khoá `func`. Function được gọi bằng tên và tập tham số bên trong dấu ngoặc đơn `()`. Kiểu của giá trị trả về được đặt sau kí tự `->`, ví dụ:

```swift
func greet(name: String, day: String) -> String {
    return "Hello \(name), today is \(day)."
}

greet("Bob", day: "Tuesday")
```

> **THỤC HÀNH**
>
> Xoá `day` khi gọi hàm `greet`.

Tuple dùng để tạo một cụm giá trị - ví dụ một hàm trả về nhiều giá trị cùng lúc. Mỗi phần tử của tuple có thể được tham chiếu đến bằng tên hoặc vị trí của nó trong tuple.

```swift
func calculateStatistic(scores: [Int]) -> (min: Int, max: Int, sum: Int) {
    var min = scores[0]
    var max = scores[0]
    var sum = 0

    for score in scores {
        if score > max {
            max = score
        } else if score < min {
            min = score
        }
        sum += score
    }

    return (min, max, sum)
}

let statistics = calculateStatistics([5, 3, 100, 3, 9])
print(statistics.sum)
print(statistics.2)
```

Function có thể nhận vào môt tập biến với số lượng không xác định, biến được lưu trong mảng

```swift
func sumOf(numbers: Int...) -> Int {
    var sum = 0
    for number in numbers {
        sum += number
    }
    return sum
}
sumOf()
sumOf(42, 597, 12)
```

> **THỰC HÀNH** Viết một hàm để tính giá trị trung bình của các giá trị chuyển vào

Function(hàm) có thể được chưá bên trong function khác (nested function). Nested-function có thể truy cập đến các biến được khai báo trong function chứa nó. nested-function được dùng để tổ chức code dài và phức tạp trong một hàm

```swift
func returnFifteen() -> Int {
    var y = 10
    func add() {
        y += 5
    }
    add()
    return y
}
returnFifteen()
```

Function thực chất là một lớp trong swift, tức là nó có thể được dùng như là kiểu giá trị trả về của một function khác, để gán gía trị cho biến...

```swift
func makeIncrementer() -> ((Int) -> Int) {
    func addOne(number: Int) -> Int {
        return 1 + number
    }
    return addOne
}
var increment = makeIncrementer()
increment(7)
```

Function có thể nhân môt function khác như tham số của nó

```swift
func hasAnyMatches(list: [Int], condition: (Int) -> Bool) -> Bool {
    for item in list {
        if condition(item) {
            return true
        }
    }
    return false
}
func lessThanTen(number: Int) -> Bool {
    return number < 10
}
var numbers = [20, 19, 7, 12]
hasAnyMatches(numbers, condition: lessThanTen)
```

Function thực chất là môt trường hợp đặc biệt của **closure**: nó là một đoạn code mà có thể được gọi lại (dùng lại). Đoạn code trong một closure có quyền truy cập đến các phần tử được khai báo trong **scope** mà closure đó được tạo ra. Closure có thể được khai báo mà không cần tên mà đơn giản chỉ dùng `{}` để đánh dâu scope cuả doạn mã Ví dụ:

```swift
numbers.map({
    (number: Int) -> Int in
    let result = 3 * number
    return result
})
```

> **THỰC HÀNH**
>
> Viết hàm để dặt tát cả các phần tử ở vị trí 0 hoặc các vị trí index lẻ

Có một số cách để có thể viết một closure đúng và ngắn gọn. Khi kiểu trả về đã biết trước (chẳng hạn như callback function), khi đó kiểu trả về có thể không cần phải viết rõ ra. Với closure chỉ có một câu lệnh, closure mặc định trả về giá trị của dòng lệnh đó

```swift
let mappedNumbers = numbers.map({number in 3 * number})
print (mappedNumbers)
```

Trong ví dụ trên, kiểu trả về của callback giống kiểu của phần tủ trong array `numbers` nên không cần phải khai báo rõ kiểu giá trị trả về. Thân của closure chỉ chứa một dòng lệnh `3* number` nên cũng không cần phải dùng tử khoá `return`

Tham số của hàm cũng có thể được tham chiếu đến thông qua vị trí của nó trong danh sách các tham số (phương pháp này rất hữu ích cho các closure ngắn). Nếu vị trí của closure nằm ở cuối danh sach tham số của một function, thì nội dung của closure có thể được viết sau dấu ngoặc đơn `()`. Nếu closure là tham số duy nhất của function thì dấu ngoặc đơn cũng có thể lược bỏ được. Ví dụ:

```swift
let sortNumbers = number.sort {$0 > $1}

// Phiên bản đầy đủ
let sortNumbers = number.sort ((number1, number2) -> Int {
    return number1 > number2
})
```

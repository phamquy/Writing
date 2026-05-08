# Protocols and Extensions

Protocol được khai báo bằng `protocol`

```swift
protocol ExampleProtocol {
    var simpleDescription : String { get }
    mutating func adjust()
}
```

class, enumberation, và struture có thể triển khai (adopt) protocols.

```swift
class SimpleClass: ExampleProtocol {
    var simpleDescription: String = "A very simple class."
    var anotherProperty: Int = 69105
    func adjust() {
        simpleDescription += "  Now 100% adjusted."
    }
}
var a = SimpleClass()
a.adjust()
let aDescription = a.simpleDescription

struct SimpleStructure: ExampleProtocol {
    var simpleDescription: String = "A simple structure"
    mutating func adjust() {
        simpleDescription += " (adjusted)"
    }
}
var b = SimpleStructure()
b.adjust()
let bDescription = b.simpleDescription
```

> **THỰC HÀNH**
>
> Viét một enumeration để hiện thực protocol

Chú cách y từ khoá `mutating` được dùng để đánh dấu môt phương thức làm thay đổi nội dung của class. Trong khai báo của `SimpleClass`, từ khoá nay là không cần thiết vì phương thức của môt class luôn có thể nội dung (trang thái) của class.

`extension` dùng để bổ xung thêm phương thức cho một class đã được định nghĩa từ trước. Nó có thể dùng để làm cho một class đã có hiện thực một protocol.

```swift
extension Int: ExampleProtocol {
    var simpleDescription: String {
        return "The number \(self)"
    }
    mutating func adjust() {
        self += 42
    }
}
print(7.simpleDescription)
```

> **THỰC HÀNH**
>
> Định nghĩa một extension cho `Double`, thêm property trả về giá trị tuyệt đối của double

protocol có thể dùng gióng như class (tạo mang, truyền tham số ...).

```swift
let protocolValue: ExampleProtocol = a
print(protocolValue.simpleDescription)
// print(protocolValue.anotherProperty)  // Uncomment to see the error
```

Trong đoạn code trên, mặc dù `protocolValue` có giá trị kiểu class, nhưng vì được khai báo là kiểu protocol nên nó chỉ có thể truy xuất đến những gì được định nghĩa trong protocol.

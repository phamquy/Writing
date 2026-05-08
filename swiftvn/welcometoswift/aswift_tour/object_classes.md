# Object and classes

Để định nghĩa một lớp tron swift, dùng từ khoá `class` theo sau là tên của lớp.Thuộc tính (property) của lớp được khai báo giống như khai báo biến (`var`)hoặc hằng số (`let`). Method của một lớp được khai báo giống như hàm (`func`). Ví dụ

```swift
class Shape {
    var numberOfSides = 0
    func simpleDescription() -> String {
        return "A shape with \(numberOfSides) sides."
    }
}
```

> **THỰC HÀNH**
>
> Thêm hằng số bằng từ khoá `let`, thêm method có nhận tham số

Để tạo một đối tượng (object) của lớp, dùng dấu ngoặc đơn sau tên của lớp. Dùng kí hiệu `.` để truy xuất thuộc tình (property). Ví dụ:

```swift
var shape = Shape()
shape.numberOfSides = 7
var shapeDescription  = shape.simpleDescription()
```

Đoạn code định nghĩa `Shape` trên thiếu một phần quan trọng của lớp: hàm khởi tạo (initializer). Hàm khởi tạo được dùng để đặt giá trị cho các thuộc tính của object mới được tạo. `init` là một hàm khởi tạo mặc định. Ví dụ:

```swift
class NamedShape {
    var numberOfSide: Int = 0
    var name: String

    init(name:String) {
        self.name = name
    }

    func simpleDescription() -> String {
        return "A shape with \(numberOfSides) sides"
    }
}
```

Để í từ khoá `self` được dùng để phân biệt biến truyền vào initializer và thuộc tính của lớp. Tham số của hàm khởi tạo được dùng giống như cách dùng đối với một hàm bình thường. Tất cả thuộc tính của một đối tượng _phải được gán giá trị_ - khi khai báo (như với `numberOfSide`), hoặc khi khởi tạo (như với `name`).

Ngược với initializer, `deinit` là hàm destructor. Nó được gọi trước khi đối tượng bị huỷ và xoá khỏi bộ nhớ (memory), đây là hàm dùng để thực hiện việc giải phóng các tài nguyên mà đối tượng đang dùng (kết nối DB, memory ...)

Lớp con (subclass) được khai báo bằng các viết tên của lớp cha (parent class) sau dấu `:`. Một lớp khi khai báo không bắt buộc phải có lớp cha.

Trong subclass nếu khai báo lại một phương thức của lớp cha (ghi đè - override) phải được khai báo sau từ khoá `override`. Ghi đè một method mà không có từ khoá `override` sẽ sinh ra lỗi khi biên dịch. Trình biên dịch cùng cảnh báo nếu dùng `override` mà không thực sụ ghi đẻ method nào của lớp cha.

```swift
class Square: NamedShape {
    var sideLength: Double

    init(sideLength: Double, name: String) {
        self.sideLength = sideLength
        super.init(name: name)
        numberOfSides = 4
    }

    func area() ->  Double {
        return sideLength * sideLength
    }

    override func simpleDescription() -> String {
        return "A square with sides of length \(sideLength)."
    }
}
let test = Square(sideLength: 5.2, name: "my test square")
test.area()
test.simpleDescription()
```

Propertis của lớp, có thể có getter và setter

```swift
class EquilateralTriangle: NamedShape {
    var sideLength: Double = 0.0

    init(sideLength: Double, name: String) {
        self.sideLength = sideLength
        super.init(name: name)
        numberOfSides = 3
    }

    var perimeter: Double {
        get {
            return 3.0 * sideLength
        }
        set {
            sideLength = newValue / 3.0
        }
    }

    override func simpleDescription() -> String {
        return "An equilateral triangle with sides of length \(sideLength)."
    }
}
var triangle = EquilateralTriangle(sideLength: 3.1, name: "a triangle")
print(triangle.perimeter)
triangle.perimeter = 9.9
print(triangle.sideLength)
```

Trong setter của `perimeter`, giá tri được truyền vào có tên mặc định là `newValue`, tuy nhiên có thể được đặt tên giống như trong định nghĩa hàm.

Chú trong đoạn code trên, hàm khởi tạo của `EquilateralTriangle` có 3 bước khác nhau: 1. Gán giá trị cho các thuộc tính của lớp con 2. Gọi hàm khởi tạo của lớp cha 3. Gán giá trị của các thuộc tính lớp cha. Các lênh khởi tạo khác nếu có có thể được thực hiện ở đây.

Nếu không cần phải tính giá trị của property, nhưng cần phải thực thi lệnh trước hoặc sau khi property được gán giá trị, dùng `willSet` và `didSet`. Code trong `willSet` và `didSet` sẽ được thưc thi mỗi khi property được gán giá trị, _trừ khi initializer_.

```swift
class TriangleAndSquare {
    var triangle: EquilateralTriangle {
        willSet {
            square.sideLength = newValue.sideLength
        }
    }
    var square: Square {
        willSet {
            triangle.sideLength = newValue.sideLength
        }
    }
    init(size: Double, name: String) {
        square = Square(sideLength: size, name: name)
        triangle = EquilateralTriangle(sideLength: size, name: name)
    }
}
var triangleAndSquare = TriangleAndSquare(size: 10, name: "another test shape")
print(triangleAndSquare.square.sideLength)
print(triangleAndSquare.triangle.sideLength)
triangleAndSquare.square = Square(sideLength: 50, name: "larger square")
print(triangleAndSquare.triangle.sideLength)
```

Khi làm việc với giá trị kiểu optional, `?` có thể được dùng trước: phương thức, thuộc tính hoắc phương thức truy xuát mảng (subscripting). Nếu giá trị trước `?` là `nil`, mọi lệnh sau nó sẽ bị bỏ qua và giá trị trả về của biểu thức là nil, Ngược lại, nếu khác `nil`, giá trị thật sẽ được dùng trong các lệnh sau đó. Trong cả hai trường hợp, giá trị trả về của biểu thức là optional. Ví dụ:

```swift
let optionalSquare: Square? = Square(sideLength: 2.5, name: "optional square")
let sideLength = optionalSquare?.sideLength
```

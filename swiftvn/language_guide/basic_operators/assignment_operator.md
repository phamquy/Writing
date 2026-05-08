# Assignment Operator

Phép gán giá trị được thực hiển bởi toán tử `=` , được dùng để khởi tạo hoặc cập nhật giá trị của biến

```swift
let b = 10
var a = 5 
a = b
//cuối cùng a có giá tri là 10
```

Nếu giá trị bên phải của toán tử là kiểu ghép (tuple) với nhiểu thành phần thì các thành phần trong tuple sẽ được gán giá trị tương ứng

```swift
let (x,y) = (1,2)
// x sẽ là 1, y sẽ là 2
```

Không giống như phép gán trong C hay Objective-C, phép gán trong Swift không trả về giá trị, nên đoạn code sau không hợp lệ trong Swift

```swift
if x = y {
  // Điểu nay không hợp lệ vì phép gán không trả vê giá tri.
}
```

Do toán tử `=` không trả về giá trị nên khi code, nếu ta có viêt nhầm `==` thành `=` thì compiler sẽ báo lỗi, giúp ta phát hiện sớm.

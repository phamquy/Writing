# Type Safety and Type Reference

Swift là một ngôn ngữ định kiểu an toàn _type-safe_. Ngôn ngữ định kiểu an toàn khuyến khích ta phải làm rõ về kiểu của giá trị, ví dụ: Nếu một đoạn code cần nhận vào một giá trị kiểu `String` thì ta không thể vô tình truyền vào giá trị kiểu `Int`.

Bởi vì Swift là ngôn ngữ type-safe nên khi biên dịch, kiểu của giá trị được kiểm tra và những chỗ sử dụng sai kiểu sẽ được dánh dấu như lỗi biên dịch. Điều này giúp ta phát hiện và sửa lỗi sớm trong khi phát triển.

Việc kiểm tra kiểu giúp tránh được lỗi khi ta làm việc với nhiểu kiểu khác nhau. Tuy nhiên nó không có nghĩa là ta phải khai báo rõ rang kiểu của tất cả các biến và hằng số. Nếu ta không khai báo rõ ràng kiểu của, Swift sẽ nội suy kiểu của biến từ giá trị hoặc biểu thức được gán cho biến hay hằng số đó.

Do nội suy kiểu, nên Swift yêu cầu rất it khai báo kiểu so với các ngôn ngữ như ObjectiveC hay C. Kiểu hằng số và biến vẫn được xác định một cách rõ ràng như phần lớn đã được thực hiện bởi nội suy kiểu.

Nội suy kiểu đặc biệt hữu ích khi ta khai báo hằng hoặc biến với giá trị khởi tạo. Giá trị khởi tạo thường được viết trực tiếp trong code khi ta khai báo biến hoặc hằng số, gọi là _literal value_.

Ví dụ khi ta gán giá trị cho một hằng số một literal-value là 42, Swift sẽ nội suy ra kiểu của hằng số sẽ là `Int` bởi giá trị khởi tạo của hằng số giống như số nguyên.

```swift
let meaningOfLife = 42
// meaningOfLife được nội suy là thuộc kiểu `Int`
```

Tương tự, nếu không khai báo kiểu là dấu phẩy động, swift sẽ nội suy kiểu giá trị là `Double` trong ví dụ sau

```swift
let pi = 3.14159
// pi được nội suy kiểu là `Double`
```

Swift luôn chọn kiểu `Double` (thay vì `Float`) khi nội suy kiểu dấu phẩy động.

Nều ta kết hợp giá trị của số nguyên và số dấu phẩy động trong một biểu thức, kiểu `Double` sẽ được nội suy từ biểu thức:

```swift
let anotherPi = 3 + 0.14.159
// anotherPi nhận kiểu là Double
```

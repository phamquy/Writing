# Tuples

_Tuple_ là nhóm các giá trị trong một tổ hợp. Các giá trị bên trong kiểu ghép có thể là kiểu bất kì, và không nhất thiết phải cùng kiểu.

Trong ví dụ sau: `(404, "Not found")` là một tuple biểu diễn một mã trạng thái HTTP. Một mã trạng thái HTTP trả về từ server khi có một yêu cầu được gửi dến. Mã `404 Not Found` được trả về khi trang web được yêu cầu khồng tồn tại.

```swift
let http404Error = (404, "Not Found")
// http404Error có kiểu (Int, String) và có giá trị (404, "Not Found")
```

The `(404, "Not Found")` ghép một giá trị `Int` và một giá trị `String` để diễn tả mã trang thái HTTP dưới hai dạng: mã 404, và văn bản dễ đọc "Not Found".

Ta có thể tạo kiểu tuple bằng cách nhóm bất kì kiểu nào theo một thứ tự bất kì. Ví dụ như `(Int, Int, Int)` hay `(String, Bool)`.

Tuple cho phép tách các giá trị trong một tuple ra các biến/hằng riêng lẻ, ví dụ:

```swift
let (statusCode, statusMessage) = http404Error
print ("The status code is \(statusCode)")
print ("The status message í \(statusMessage)")
```

Khi chỉ cần tách giá trị của một phần trong tuple mà không quan tâm giá trị các phần khác, `_` dùng để thay thế cho các phần không cần đó. Ví dụ

```swift
let (justTheStatusCode, _) = http404Error
print("The status code is \(justTheStatusCode)")
```

Giá trị trong tuple có thể được truy cập bằng vị trí trong tuple. Ví dụ:

```swift
print("The status code is\(http404Error.0)")
print("The status message is \(http404Error.1)")
```

Các thành phần trong tuple có thể được đặt tên, ví dụ:

```swift
let http200Status = (statusCode: 200, description: "OK")
```

Nếu các thành phần được đặt tên, chúng có thể được truy cập bằng tên đó, ví dụ:

```swift
print("The status code is \(http200Status.statusCode)")
print("The message is \(http200Status.description)")
```

Tuple đặc biệt hữu ích khi được dùng như giá trị trả về của một function. Một function yêu cầu một trang web có thể trả về giá trị (Int, String). Bằng cách trả vê giá trị kiểu tuple, function cung cấp nhiều thông tin hơn việc chỉ trả vê một giá trị đơn lẻ. Xem thêm chi tiêt (TODO: thêm link đến Function with Multiple Return Values)

> CHÚ Ý
>
> Tuple hữu ích khi dùng cho một nhóm các giá trị tạm thời. Tuple không thích hợp khi dùng để biểu diễn các kiểu cấu trúc dữ liệu phức tạp. Nếu dữ liệu được xử lí được giữ ngoài khu vục tạm thời, nên dùng structure hoặc class. Xem chi tiết [Classes and Structure](../class_and_structures/)

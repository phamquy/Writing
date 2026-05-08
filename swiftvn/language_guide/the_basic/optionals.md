# Optionals

Optionals được dùng để chỉ định có thể biến hoặc hằng không có giá trị. Khi một biến hoặc hằng số có kiểu là optional, thì nó có thể: có giá trị có thể được tách để sử dụng, hoặc không có giá trị.

> CHÚ Ý
>
> Khái niệm optional không tồn tại trong C hay Objective-C. Điều gần giống nhất với optional trong Objective-C là việc trả về giá trị `nil` hoặc một đối tượng từ một hàm số. Tuy nhiên nó chỉ có thể ap dụng cho kiểu đối tượng, không áp dụng cho các kiểu cơ bàn, structure, hoặc enum. Đổi với các kiểu đó, trong Objective-C, hàm thường trả một giá trị dặc biệt (chẳng hạn `NSNotFound`) để ám chỉ sự _không có giá trị_. Tuy nhiên cách tiếp cận này yêu cầu đối tượng sử dụng hàm này phải biết ý nghĩa của giá trị đặc biệt. Kiểu optional trong Swift cho phép chỉ định sự vắng của giá trị cho tất cả các kiểu mà không cần phải sừ dụng một giá trị đặc biệt nào.

Dưới đây là ví dụ về cách sử dụng kiểu giá trị optional. Trong swift, kiểu `Int` có một hàm khởi tạo nhân vào một `String` và tạo một so mà string đó biều diễn. Tuy nhiên không phải tất cả các string đều biểu diẽn số. Do đó hàm khởi tạo sẽ trả về một số `Int` hoặc _không có giá trị_

Ví dụ

```swift
let possibleNumber = "123"
let convertedNumber = Int(possibleNumber)
// converteNumber sẽ được nội suy ra kiểu là `Int?`, hiểu là optional int
```

Vì hàm khởi tao có thể không tạo được số nên nó trả về `Int?` thay vì `Int`.

## nil

Để đặt một biến dang optional về trạng thái vô giá trị, ta gán cho nó giá trị `nil`

```swift
var serverResponseCode : Int? = 404
// serverResponseCode chứa giá trị 404
serverResponseCode = nil
// serverResponseCode bay gio không có giá trị
```

> CHÚ Ý
>
> `nil` không thể được dùng cho biến không phải là optional. Nếu một hằng hoạc biến có thể lúc nào đó ở trạng thái không có giá trị, nên khai báo no là kiểu optional

Nêu khai báo một biến optional mà không có giá trị khởi tạo, nó se nhận giá trị mặc định là `nil`

```swift
var surveyAnswer : String?
// surveyAnswer được tự động gán cho giá trị nil
```

> CHÚ Ý
>
> Giá trị `nil` trong swift khác với `nil` trong Objective-C. Trong objective-c, Nil là một con trỏ trỏ đến một đổi tượng không tồn tại. Trong swift, `nil` không phải là con trỏ, nó là trạng thái _không có giá trị_ của một kiểu nào đó. Optional của tất cả các kiểu có thể được gán `nil` không nhất thiêt phải là kiêu đối tượng.

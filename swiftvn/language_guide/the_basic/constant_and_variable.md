# Constant and variable

Hằng số và biến số gắn kết một tên (ví dụ `maximumNumberOfLogin` hoặc `welcomeMessage`) với một giá trị thuộc một kiểu cụ thể nào đó (ví dụ số `10` hoặc chuỗi kí tự `Hello`). Giá trị của hằng số (constant) không thay đổi được sau khi được gán giá trị lần đầu tiên, giá trị của biến thì có thể.

## Khai báo hằng số và biến số

Hằng và biến phải được khai báo trước khi nó được sử dụng. Hằng số được khai báo bằng từ khoá `let` con biến số được khai báo bằng `var`. Ví dụ sau khai báo hằng số lưu giá trị số lần login tối đa, và biến số lưu giá trị số lần login hiện tại:

```swift
let maximumNumberOfLoginAttempts = 10
var currentLoginAttempt = 0
```

Đoạn code trên có thể được hiểu:

"Khai báo một hằng số tên là `maximumNumberOfLoginAttempts` và gán giá trị `10` cho nó. Sau đó khai báo một biến số tên `currentOfLoginAttempt` và gán giá trị ban đầu cho biến là `0`."

Trong ví dụ này, số lần login tối đa được khai báo là hằng số vì nó không thay đổi. Trong khi đó, số lần login hiện tại được khai báo là biến số vì nó tăng dần sau mỗi lần login thất bại.

Bạn có thể khai báo nhiểu biến hoặc hằng số trên cùng một dòng bằng cách ngăn cách với dấu phẩy:

```swift
var x = 0.0, y = 0.0, z = 0.0
```

> CHÚ Ý
>
> Nếu giá trị được lưu sẽ không bị thay đổi, nó nên và luôn luôn nên được khai báo với từ khoá `let`. Chỉ sử dụng `var` nếu giá trị lưu có thể bị thay đổi.

## Khai báo kiểu

Kiểu của biến hoặc hằng số có thể được khai báo khi khai báo biến hoăc hằng số đó. Kiểu được khai báo bằng cách thêm `:` sau tên của hằng hoặc biến và viết tên của kiểu sau `:` cách một khoàng trống.

Ví dụ sau khai báo kiểu của biến là `String`

```swift
var welcomeMessage: String
```

Dấu `:` có nghĩa _"...thuộc kiểu..."_, và đoạn code trên có thể được đọc: khai báo một biến tên `welcomeMessage` thuộc kiểu `String`

Biến `welcomeMessage` sau đó có thể được gán với bất cứ chuỗi kí tự nào

```swift
welcomeMessage = "hello"
```

Ta có thể khai báo nhiều biến với cùng một kiểu trên cùng một dòng

```swift
var red, green, blue: Double
```

> CHÚ Ý
>
> Rất hiếm khi ta phải khai báo kiểu, nếu giá trị khởi tạo của được gán. Kiểu của biến sẽ được suy ra từ giá trị được gán đó. Trong ví dụ trên, khi gán giá trị cho `welcomeMessage` là một chuỗi kí tự, kiểu của nó dược suy là `String`

## Đặt tên có biến và hằng số

Tên của biến và hằng số có thể chứa bất cứ một kí tự nào bao gồm cả các kí tự Unicode:

```swift
let π = 3.14159
let 你好 = "你好世界"
let 🐶🐮 = "dogcow"
```

Tên của hằng và biến số không được phép chứa khoảng trống, kí tự toán học, dấu mũi tên, các kí tự không hợp lệ hoắc kí tự được sử dụng vào mục đích đặc biệt, các kí tự để vẽ đường thẳng và ô vuông. Kí tự số có thể xuất hiện bất kì vị trí nào trong tên trừ vị trí đầu tiên.

Khi một tên biến được khai báo, ta không thể dùng tên đó cho biến khác, hoặc thay đổi kiểu của biến, ta cũng không thể thay đổi từ dạng biến sang hằng số và ngược lại.

> CHÚ Ý
>
> Nếu ta đặt tên biến giống với một từ khoá trong Swift, ta phải đặt tên biến đó trong dấu (\`) khi sử dụng tên đó. Tuy nhiên không nên sử dụng từ khoá như là tên của biến trừ khi bắt buộc.

Ta có thể thay đổi giá trị của một biến đã có sang một giá trị mới cùng kiểu. Ví dụ:

```swift
var friendlyWelcome = "Hello!"
friendlyWelcome = "Bonjour!"
```

Không giống như biến, giá trị của hằng số không thể bị thay đổi sau lần gán đàu tiên. Việc gán giá trị mới cho hằng số sẽ tạo ra lỗi khi biên dịch

```swift
let languageName = "Swift"
languageName = "Swift++"
// this is a compile-time error - languageName cannot be changed
```

## Hiển thị hằng và biến

Ta có thể in giá trị hiện tại của biên hoặc hằng số với hàm `print(_:separator:terminator:)`

```swift
print(friendlyWelcome)
// Prints "Bonjour!"
```

Hàm `print(_:separator:terminator:)` là một hàm toàn cục (global), nó in một hoặc nhiều giá trị ra output thích hợp. Trong Xcode, nó sẽ in giá trị ra console. `separator` và `ternimator` có giá trị mặc định nên ta có thể không cần truyền giá trị cho nó khi gọi. `terminator` có giá trị mặc định là kí tự kết thúc dòng, để in giá trị mà không xuống dòng, truyền vào string rỗng cho `terminator`, ví dụ: `print(somevalue, terminator: "")`. Xem chi tiết về tham số và giá trị mặc định: TODO: link to Default Parameter Value

Swift sử dụng _nội suy chuỗi kí tự_ để chèn giá trị vào trong một chuối dài hơn. Để chèn giá trị vào một chuỗi được in ra, ta bọc giá trị đó trong dấu ngoặc và thêm dấu  trước nó:

```swift
print("The current value of friendlyWelcome is \(friendlyWelcome)")
```

> CHÚ Ý
>
> Tất các các tuỳ chọn được dùng trong nội suy chuỗi kí tự được đề cập đầy đủ trong TODO: link to String Interpolation

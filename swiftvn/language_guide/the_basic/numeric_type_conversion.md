# Numeric Type Conversion

Dùng kiểu `Int` cho các biến và hằng số chung chung ngay cà khi biết trước giá trị của nó là không âm. Dùng kiểu số nguyên mặc định `Int` giúp cho code dễ hoán chuyển hơn. Chỉ nên sử dụng các kiểu số nguyên khác khi nó đặc biệt cần thiết cho một việc cụ thể, ví dụ: đọc dữ liệu từ một nguồn với kích thức cụ thể, khi tối ưu tốc độ hoặc bộ nhớ...

## Chuyển đổi kiểu số nguyên

Dải giá trị có thể được lưu bởi một số nguyên của mổi kiểu số nguyên là khác nhau. Một số `Int8` có dải giá trị `-128` đến `127`, trong khi đó `UInt8` có dải giá trị từ `0` đên `255`. Khi giá trị của một số nằm ngoải dải giá trị của kiều, sẽ có lỗi biên dịch:

```swift
let cannotBeNagative: UInt8 = -1 // Lỗi biên dịch, UInt8 không thể có giá trị âm
let tooBig : Int8 = Int8.max +1 // Int8 không thể chứa giá trị lớn hơn giá trị lớn nhắt
```

Vì mỗi kiểu chỉ có thể chứa giá trị trong một dải nhất định, nên cần phải chuyển đổi kiều. Việc chuyển đội kiểu được viết rõ ràng trong code, Swift không tự động chuyển kiểu như một số ngôn ngữ khác. Ví dụ:

```swift
let twoThousand : UInt16 = 2_000
let one: UInt8 = 1
let twoThousandAndOne = twoThousand + UInt16(one)
```

Trong ví dụ trên, ta không thể cộng trực tiếp `twoThousand` vad `one` vì kiểu của chung khác nhau. Để thực hiện phép cộng, cần phải chuyển về chung một kiểu `UInt16` (kiểu có dải giá trị lớn hơn). Kết quả có kiểu `Uint16` vì nó là kết quá phép cộng của hai số `UInt16`

Cú pháp `Kiểu(giá trị)` là cú pháp để tạo một giá trị của kiểu với giá trị ban đầu. Trong ví dụ trên, `UInt16` được khởi tạo với giá trị ban dâu là một số kiểu `UInt8`. Ta không thể khởi tạo `UInt16` vói giá trị ban đầu bất kì, chỉ có kiểu mà `UInt16` cung cấp hàm khởi tạo. Để thêm các hàm khởi tạo khác, xem **Extension** (need link)

## Chuyển đổi số nguyên và số dấu phảy động

Chuyển đổi kiều giữa số nguyên và số dấu phảy động cần được viêt rõ ràng:

```swift
let three = 3
let pointOneFourOneFiveNine = 0.14159
let pi = Double(three) + pointOneFourOneFiveNine
```

Ở ví dụ trên, số `3` phải được đổi sang kiểu số dấu phảy động trước khi thực hiện phép công, nều không sẽ có lỗi biên dich.

Đổi kiểu từ số dấu phảy động sang số nguyên cũng phải được viết rõ ràng. Một số nguyển có thể được khởi tạo từ một số `Double` hoặc `Float`:

```swift
let integerPi = Int(pi)
```

Phần lẻ của số phẩy động sẽ bị cắt khi chuyển kiểu sang số nguyên, ví dụ: `4.75` khi chuyển sang số nguyên sẽ là `4`, `-3.9` khi chuyển sang số nguyên sẽ là `-3`

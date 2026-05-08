# Enumertation and Structures

Khai báo kiểu enumeration bằng `enum`. Giống như lớp hay các kiểu khác, enumeration có thể chứa phương thức (method) riêng.

```swift
enum Rank: Int {
    case Ace = 1
    case Two, Three, Four, Five, Six, Seven, Eight, Nine, Ten
    case Jack, Queen, King
    func simpleDescription() -> String {
        switch self {
        case .Ace:
            return "Ace"
        case .Jack:
            return "Jack"
        case .Queen:
            return "Queen"
        case .King:
            return "King"
        default:
            return String(self.rawValue)
        }
    }
}

let ace = Rank.Ace
let aceRawValue = ace.rawValue
```

> **THỰC HÀNH**
>
> Viết hàm để so sánh hai giá trị kiểu `Rank` bằng cách so sánh (giá trị thô) raw-value

Trong ví dụ trên, kiểu của raw-value là `Int`, nên chỉ cần gán giá trị thô cho định nghĩa đầu tiên - `Ace` - các định nghĩa tiếp sau tự động nhận giá trị theo thứ tự xuất hiện: `Two` = 2, `Three` = 3, ... Khác với các C/C++, Swift cho phép enumeration được định nghĩa dựa trên các kiểu khác `Int`. `rawValue` cho phép truy xuất giá trị thô của enumeration.

Sử dụng `init?(rawValue:)` để khởi tạp một đối tượng kiểu enumeration từ raw-value. Ví dụ:

```swift
if let convertedRank = Rank(rawValue:3) {
    let threeDescription = convertedRank.simpleDescription()
}
```

Các tên được định nghĩa trong enumeration được dùng như giá trị của enumeration, nó không phải chỉ là tên thay thế cho giá trị thô. Điểu này có nghĩa, nêu raw-value không có ý nghĩa thì không cần phải viết ra trong định nghĩa của enumeration. Ví dụ sau, `Suit` không cần phải có raw-value vì raw-value không có nghĩa ở đây.

```swift
enum Suit {
    case Spades, Hearts, Diamonds, Clubs
    func simpleDescription()->{
        switch self {
            case .Spades:
                return "spades"
            case .Hearts:
                return "hearts"
            case .Diamonds:
                return "diamonds"
            case .Clubs:
                return "clubs"
        }
    }
}
let hearts = Suit.Hearts;
let heartsDescription = hearts.simpleDesciption()
```

> **THỰC HÀNH**
>
> Thêm phương thức `color()` cho `Suit` để trả về màu.

Chú ý cách mà `Hearts` được dùng ở trên: khi ta gán giá trị cho `hearts`, ta dùng tên đầy đủ `Suit.Hearts`. Khi mà kiểu của biến đã biết thì có thể lược bỏ thành `.Hearts` giống như trong `simpleDescription`

Structure được khai báo bằng `struct`. Structure (cấu trúc) cho phép có các thanh phần giống như class: methods, hàm khởi tạo. Sự khác biệt quan trọng nhất giữa structure và class là: giá trị structure được sao chép khi dùng lại (pass by value), còn class thì chí sao chép lại tham chiếu (reference).

```swift
struct Card {
    var rank: Rank
    var suit: Suit
    func simpleDescription() -> String {
        return "The \(rank.simpleDescription()) of \(suit.simpleDescription())"
    }
}

func isNumberCard(card: Card) -> Bool {
    if ((card.rank.rawValue < 11) && (card.rank.rawValue != 0)){
        return true
    }
    return false
}
let threeOfSpades = Card(rank: .Three, suit: .Spades)
let threeOfSpadesDescription = threeOfSpades.simpleDescription()
let isNumber = isNumberCard(threeOfSpades)
```

Trong ví dụ trên, khi gọi hàm `isNumberCard`, một bản sao của `threeOfSpades` được tạo ra vả truyển cho hàm số `isNumberCard`

> **THỰC HÀNH**
>
> Thêm phương thức vào `Card` để trả về một bộ bài đầy đủ

Enumeration trong swift có thể có giá trị đi kèm (associated value). Một case của enumeration có thể được gán các associated-value khác nhau. associated-value khác với raw-value. Raw-value của một case không thay đổi trong khi associated-value có thể.

Trong ví dụ sau, định nghĩa enumeration với hai case: `Result` và `Error` thể hiện kết quả trả về từ một server.

```swift
enum ServerResponse {
    case Result(String, String)
    case Error(String)
}

let success = ServerResponse.Result("6:00 am", "8:09 pm")
let failure = ServerResponse.Error("Out of cheese.")

switch success {
case let .Result(sunrise, sunset):
    print("Sunrise is at \(sunrise) and sunset is at \(sunset).")
case let .Error(error):
    print("Failure...  \(error)")
}
```

Kết quả trả về có hai trường hợp, như mỗi trường hợp lại có associated-value cung cấp thêm thông tin về nó.

> **THỰC HÀNH**
>
> Thêm case `ServerResponse` vào đoạn code trên

Chú ý các associated-value được truy xuất tử enumeration.

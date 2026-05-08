# Comparison Operators

Swift có tất cả các toán tử so sánh có trong C:

* Bằng (a == b)
* Không bằng (a != b)
* Lớn hơn (a > b)
* Nhỏ hơn (a < b)
* Lớn hơn hoặc bằng (a >= b)
* Nhỏ hơn hoặc bằng (a <= b)

> CHÚ Ý
>
> Swift cũng có hai toán tử identity operator (=== và !==), được dùng để kiểm tra nếu hai tham chiếu đến đối tượng là giống nhau. [Chi tiết xem thêm Classes and Structures](../class_and_structures/)

Mỗi một toán tử so sánh trả vê một giá trị `Bool` đẻ chỉ phép so sánh là đúng (true) hay sai (false):

```swift
1 == 1 // true
2 != 1 // true
2 > 1  // true
1 < 2  // true
1 >= 1 // true
2 <= 1 // false
```

Phép so sánh thường được dùng trong lênh `if`:

```swift
let name = "world"
if name == "world" {
    print("hello, world")
} else {
    print("I'm sorry \(name), but i don't recognize you"
}

// prints "hello, world"
```

Chi tiết về các lệnh điều khiển xem [Control Flow](../../welcometoswift/aswift_tour/controlflows.md).

Ta cũng có thể so sánh các giá trị của kiểu ghép (tuple) nếu mỗi giá trị đơn trong nó có thể so sánh được với nhau. Ví dụ vả số `Int` và `String` đểu là kiểu có thể so sánh được nên tuple `(Int, String)` là kiểu dữ liệu so sánh được. Ngược lại, `Bool` là kiểu không thể so sánh được nên tuple chứa kiểu `Bool` cũng không phải là kiểu so sánh được.

Tuple được so sánh từ trái qua phải, tưng giá trị môt, các giá trị được so sánh cho đển khi tìm được hai giá trị trong tuples không bằng nhau. Nếu tát cả các giá trị trong hai tuples bằng nhau thì hai tuple đó bằng nhau. ví dụ

```swift
(1, "zebra") < (2, "zebra") // Ví 1 < 2

(3, "apple") < (3, "bird") // Vì 3 bằng 3, nhưng "apple" nhỏ hơn "bird" trong phép so sánh chuỗi

(4, "dog") == (4, "dog")
```

> CHÚ Ý
>
> Swift chỉ hỗ trơ phép so sánh cho tuple có dưới 7 giá trị, để so sánh tuple có từ 7 giá trị trở lên, ta phải tự viết code.

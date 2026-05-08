# Error Handling

Xư lí lỗi hay Error Handling là cách mà chúng ta xử lí khi có lỗi xảy ra trong khi chương trình đang chạy.

Trái ngược với kiểu "optional", kiểu mà có thể dùng sự tồn tại hay không tồn tại của giá trị để chỉ định sự thành công hay thất bại của một hàm, xử lí lỗi cho phép ta xác định được nguyên nhân gây ra lỗi và nếu cần có thể chuyển lỗi đó sang bộ phận khác của chương trình.

Khi một hàm rơi vào trạng thái lỗi, hàm sẽ ném ra (throws) một lỗi. Hàm mà thực hiện lời gọi tới hàm bị lỗi đó có thể bắt (catch) lỗi này và xử lí một cách hợp lí.

```swift
func canThrowAnError() throws {
  // Hàm này có thể hoặc không ném ra một lỗi
}
```

Một hàm khai báo có khả năng ném ra lỗi bằng từ khoá `throws`. Khi gọi một hàm có khả năng ném ra lỗi, dùng từ khoá `try` phía trước lời gọi hàm.

Swift tự động truyền lỗi ra khỏi phạm vi hiện tại cho tới khi nó được bắt lại bới `catch`

```swift
do {
  try canThrowAnError()
  // Thực hiện tiếp nếu không có lỗi
} catch {
  // Lỗi được bắt ở đây
}
```

Cú pháp `do` tạo ra một phạm vi (scope), nó cho phép lỗi được chuyển ra một hoặc nhiều lệnh `catch`

Dưới đây là một ví dụ minh hoạ cách xử lí lỗi với nhiều trạng thái lỗi khác nhau

```swift
func makeASandwich() throws {
  // ...
}

do {
  try makeSandwich()
  eatASandwich()
} catch SandwichError.outOfCleanDishes {
  washDishes()
} catch SandwichError.missingIngredient(let ingredients){
  buyGroceries(ingredient)
}
```

Trong ví dụ này, hàm `makeSandwich()` sẽ ném ra lỗi nếu không có đĩa sạch (outOfCleanDishes) hoặc không đủ nguyên liệu (missingIngredient). Do hàm `makeSandwich()` có thể ném ra lỗi nên lời gọi hàm được đặt trong lệnh `do`.

Nếu không có lỗi, `eatASandwich()` sẽ được gọi. Nễu có lỗi được ném ra, tuỳ thuọc loại lỗi, các lệnh trong cú pháp `catch` sẽ được gọi.

Chi tiết về xử li lỗi (throw, catch) và chuyển lỗi ra ngoài hàm gọi được mô tả chi tiết trong [Error Handling](../error_handling/)

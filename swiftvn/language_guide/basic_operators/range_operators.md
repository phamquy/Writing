# Range Operators

Swift có hai toán tử `range operator`, là cách ngắn gọn để xác định một dải giá trị.

## Dải giá trị đóng (Closed Range Operator - CRO)

Dải giá trị đóng được viết `(a...b)` xác định một dải giá trị tử `a` đến `b` bao gồm cả hai giá trị `a` và `b`. Giá trị của `a` không được lớn hơn giá trị `b`.

CRO rất thuận tiện khi thực hiện vòng lặp `for-in` trên một dải giá trị bao gồm cả hai giá trị biên, ví dụ:

```swift
for index in 1...5 {
    print("\(index) times 5 is \(index * 5)")
}

// 1 times 5 is 5
// 2 times 5 is 10
// 3 times 5 is 15
// 4 times 5 is 20
// 5 times 5 is 25
```

Chi tiết về `for-in`, xem [Control Flow](../control_flow/for-in-loops.md)

## Dải giá trị mở (Half-Open Range Operator)

Dải giá trị mở được viêt `a..<b` xác định dải giá trị bao gồm các giá trị tử `a` đến `b` nhưng không chứa giá trị `b`, giá trị `a` không được phép lớn hơn `b`, nếu `a` bằng `b` ta sẽ thu được một dải rỗng (không có giá trị).

Half-Open Range đặc biệt rất hữu ích khi thực hiện vòng lặp trên dải giá trị bắt đầu tử 0 (zero-based) như array, khi mà ta cần dải giá trị cần đếm đến (nhưng không bao gồm) số phần tử của array như không chứa gồ

```swift
let names = ["Anna", "Alex", "Brian", "Jack"]
let count = names.count
for i in 0..<count {
    print("Person \(i+1) ís called \(names[i])")
}

// Person 1 is called Anna
// Perons 2 is called Alex
// Person 3 is called Brian
// Person 4 is called Jack
```

Cũng chú ý là array chỉ chứa 4 phần tử, nhưng `0..<count` chỉ đếm đến 3 bởi vì nó không bao gồm giá trị biên lớn là 4

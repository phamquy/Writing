# Ternary Conditional Operator

Toán tử điều kiện (ternary conditional operator) là một toán tử đặc biệt thực hiện với ba toán hạng, có dạng sau: `điều kiện ? giá trị 1 : giá trị 3`. Nếu điều kiện là true thì lấy giá trị 1 nều điều kiện false thì lấy giá trị 2.

nó là dạng viết tắt của

```swift
if đieu_kien {
   giá trị 1
}else{
   giá trị 2
}
```

Ví dụ sau tính chiêu cao của dong trong table. Chiểu cao của dòng phải cao hơn chiều cao của nội dung la 50point nếu dòng có header và 20 point néu khong co header:

```swift
let contentHeight = 40
let hasHeader = true
let rowHeight = contentHeight + (hasHeader ? 50:20)
```

Nó là dạng viêt tắt của

```swift
let contentHeight = 40
let hasHeader = true
let rowHeight: Int
if hasHeader { 
    rowHeight = contentHeight + 50
} else { 
    rowHeight = contentHeight + 20
}
// rowHeight is equal to 90
```

Trong ví dụ đầu tiên `rowHeight` được gán giá trị chỉ bằng một dòng lệnh, nhưng trong ví dụ thứ hai can đến 4 dòng lệnh với cấu trúc điều kiện`if.`Toán tử điều kiện cung cho phép viết biêu thức môt cách ngăn gọn như nêu lam dụng (viet lông vạo nhau) sẽ làm cho code trở lên khó đọc.

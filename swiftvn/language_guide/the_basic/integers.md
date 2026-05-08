# Integers

Số nguyên là số không có phần lẻ, ví dụ 42 and -23. Số nguyên có thể là số có dấu (số dương, số âm, hoặc 0) hoặc là số không có dấu (số dương hay số 0).

Swift có các kiểu số có dấu và không dấu với độ dài bit 8, 16, 32 và 64. Các kiểu này có cách đặt tên giống trong C: số nguyên không dấu 8-bit có tên `UInt8`, số nguyên có dấu 32 bit tên `Int32`. Giống như các tên kiểu trong Swift, tên các kiểu nguyên có chữ cái đầu tiên viêt hoa.

## Giải giá trị của số nguyên

Ta có thể sử dụng giá trị lớn nhát và nhỏ nhất của các kiểu số nguyên bằng cách truy cap thuộc tính `min` và `max` của kiểu đó:

```swift
let minValue = UInt8.min // giá trị nhỏ nhát của số nguyên không dấu 8 bit, tức giá trị 0
let maxValue = UInt8.max // Giá trị lớn nhất 255
```

giá trị `max` và `min` có cùng kích thước bit nên có thể dùng trong các biểu thức với các giá trị cùng kiều

## Int

Trong hầu hết các trường hợp, ta không cần phải chọn kiểu số nguyên cụ thể. Swift cung cấp kiểu `Int` là kiểu số nguyên có cùng kích thước với kích thước bit của nền tảng hiện tại mà swift chay trên đó:

* Trên nền tảng 32 bit, `Int` có kích thước bit là 32, tức giống kiểu `Int32`
* Trên nền tảng 64 bit, `Int` có kích thước bit là 64, túc giống kiểu `Int64`

Trừ khi ta phải làm việc với một số nguyên với kích thước bit cụ thể, nếu không luôn luôn nên dùng `Int` cho các giá trị số nguyên. Điều này giúp cho code chương trình chặt chẽ và khả chuyển trên nhiều nền tảng khác nhau. Kể cả khi dùng `Int32`, Int có thể lấy giá trị từ -2,147,483,648 đến 2,147,483,647, đủ lớn để chứa giá trị số nguyên trong hầu hết các trường hợp.

## UInt

Swift cung cấp kiểu số nguyên không dấu có kích thước bit giống nhu kích thuóc bit cuả nền tảng hiện tại:

* Trên nền tảng 32 bit, `UInt` có kích thước bit là 32, tức giống kiểu `UInt32`
* Trên nền tảng 64 bit, `UInt` có kích thước bit là 64, túc giống kiểu `UInt64`

> CHÚ Ý
>
> Chỉ nên dùng `UInt` khi cần số nguyên không dấu có kích thước bit giống nền tảng, còn không nên dùng `Int`. Dùng `Int` một cách thống nhất giúp cho mã nguồn khả dụng trên nhiều nền tảng, tránh được nhừng rắc rối liên quan đến chuyển đổi giữa các kiểu.

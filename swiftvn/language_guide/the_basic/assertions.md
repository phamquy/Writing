# Assertions

Trong một số trường hợp, không thể thực hiện tiếp chương trình khi một điều kiện nào đó không được thoả mãn. Trong trường hợp đó, chúng ta có thể dùng lệnh xác nhận `assertion` để xác nhận điều kiện và kết thúc chương trình khi đièu kiện không được thoả mãn (lỗi), `assertion` cho phép chúng ta debug và tìm ra nguyên nhân của lỗi.

## Gỡ lỗi với Assertions

`Assertion` kiểm tra điều kiện trong thời gian chạy. Ta dùng assertions để xác nhận một điều kiện là `true` trước khi thực hiện các lệnh tiếp theo. Nếu điều kiện được thoả mãn, chương trình được tiếp tục thực hiện như bình thường, nếu điều kiện không được thoả mãn, chương trình kết thúc.

Nếu chương trình thực hiện xác nhận trong khi debug, ta có thế biết chính xác vị trí nơi chương trình bị kết thúc và xem trạng thái của chương trình. `Assertion` cũng cho phép in ra thông báo khi xác nhận không thoả mãn.

Ta thực hiện lệnh xác nhận với bằng hàm `assert(_:_:file:line:)` của thư viện chuẩn của Swift. Biểu thức logic được truyển vào để xác nhận sẽ được kiểm tra, nếu biểu thức truyển vào cho giá trị `false`, thông báo sẽ được in ra:

```swift
let age = -3
assert(age >=0, "a persion age cannot be less than zero")

// Đoạn code trên, lệnh assert sẽ in ra nội dung thông báo
```

Trong ví dụ trên, đoạn code sẽ chỉ tiếp tục khi `age >= 0` là `true`, nghĩa là chỉ khi nào `age` có giá trị không âm. Nếu giá trị của `age` là âm, lênh `assert` sẽ dừng chương trình và in thông báo.

Nội dung thông báo trong lệnh `assert` có thể được bỏ qua, ví dụ như:

```swift
assert(age > 0)
```

> CHÚ Ý
>
> Lệnh xác nhận bị bỏ qua khi code được dịch ở với chế độ tối ưu hoá `optimization`, ví dụ như khi ta build ứng dụng với cấu hình cho `release` trong xcode

## Khi nào dùng xác nhận lỗi

Sử dụng xác nhận lỗi khi có một điều kiện nào đó có thể không thoả mãn, nhưng trương trình yêu cầu nó phải được thoả mãn để có thể thực hiện tiếp. Một số trường hợp ví dụ:

* Chỉ số của một subscript phải nằm trong một dải gía trị nhất định, không được quá lớn hoạc quá nhỏ
* Tham số truyển vào cho một hàm phải có giá trị mà hàm số có thể xử lí được

> CHÚ Ý
>
> Assertions dừng chương trinh khi điều kiện không thoả mãn, do đó chỉ nên dung trong quá trình phát triển. Nó không phải là giải pháp thay thế để xử lí khi điều kiện đó.

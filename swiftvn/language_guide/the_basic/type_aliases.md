# Type Aliases

_Type aliases_ định nghĩa tên cho một kiểu đã có. Từ khoá `typealias` được dùng để định nghĩa tên.

Tên định danh hứu ích khi ta muốn gán cho kiểu một ý nghĩa nào đó phù hợp với hoàn cảnh sử dụng. Ví dụ khi xử lí dữ liệu với kich thước cụ thể từ một nguồn ngoài:

```swift
typealias AudioSample = UInt16
```

Sau khi định nghĩa tên, ta có thể dùng tên này thay thế cho tên gốc:

```swift
var maxAmplitudeFound = AudioSample.min // maxAmplitudeFound lấy giá trị 0
```

Ở đây `AudioSample` là tên định danh cho `UInt16`, `AudioSample.min` thực chất sẽ gọi tới hàm `UInt16.min`.

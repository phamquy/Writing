# The Basic

Swift là ngôn ngữ lập trình mới cho iOS, OS X, watchOS và tvOS. Có rất nhiều phần của Swift tương tự C và Objective-C.

Swift có kiểu cơ bản riêng tương tự với C và Objective-C bao gồm: `Int` cho số nguyên, `Double` và `Float` cho số dấu phẩy động, `Bool` cho giá trị kiêu boolean, và `String` cho dữ liệu dang text. Swift cũng cung cấp 3 kiểu dữ liệu tập hợp rất tiện ích: `Array`, `Set` và `Dictionary`. \[TODO: link to CollectionType]

Giống như C, Swift sử dụng biến dể chứa và tham chiếu đến giá trị bằng tên. Swift cũng dùng rất nhiều kiểu biến mà giá trị của nó không thay đổi, gọi là hằng số. Hằng số (constants) trong Swift hữu dụng hơn rất nhiều so với hằng số trong C. Hằng số được sử dụng rất nhiều xuyên suốt code trong Swift giúp cho các mã nguồn Swift an toàn hơn và dễ hiểu hơn.

Bênh cạnh các kiểu dữ liệu cơ bản, Swift có thêm kiểu dữ liệu `tuples`. `Tuples` cho phép tạo và sử dụng một nhóm giá trị như một giá trị. Ta có thể dung tuple để trả về một nhiều giá trị cùng lúc từ một hàm số dưới dạng một cụm giá trị.

Swift cũng cung cấp kiểu giá trị tuỳ trọn `optional type`, là kiểu để biểu hiện trạng thái _không có giá trị_. Optionals nghĩa là: biến có chữa giá trị bằng `x`, hoăc không chứa giá trị gì. Sử dụng kiểu optional tương tự như sử dụng `nil` với con trỏ trong Objective-C, tuy nhiên kiểu optional có thể dùng cho mọi kiểu dữ liệu chứ không chỉ kiểu lớp (class) như trong Objective-C. Kieu Optionals không chỉ giúp cho viết code an toàn hơn mà nó còn giúp viết code dễ hiểu hơn, kiểu optional là nền tảng trong rất nhiểu tính năng ưu việt trong Swift.

Swift là ngôn ngử `type-safe`, nghĩa là kiểu của biến được dung luôn phải được xác định rõ ràng. Nếu code yêu cầu kiểu `String`, type-safe sẽ không cho phép bạn truyền vào kiểu `Int`. Tương tự vậy, nó cũng ngăn không cho truyền một optional `String` vào chỗ cần kiểu non-optional `String`. Type-safe giúp phát hiện sớm lỗi trong quy trình phát triển.

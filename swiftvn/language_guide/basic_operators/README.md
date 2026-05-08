# Basic Operators

Một toán tử (operator) là một kí tự hoặc một nhóm kí tự đặc biệt dùng để kiểm tra, thay đổi hay ghép các giá trị. Ví dụ như phép cộng `+` dùng để cộng giá trị của hai số, như `let i = 1 + 2`. Phép logic AND được thực hiện bởi toán tử `&&` để ghép hai giá trị `Boolean` như `if enteredDoorCode && passRetinaScan`

Swift hỗ trợ hầu hết các toán tử trong C và cải thiện một số để loại bỏ các lỗi lập trình phổ biển. Phép gán giá trị `=` không trả vể gía trị, điều này ngăn chặn việc vô tình sử dụng toán tử này thay vì `==`. Toán tử số học (`+`, `-`, `/`, `%` ...) kiểm tra và không cho phép các giá trị tràn (overflow), điều này giúp tránh các lỗi do làm việc với các giá trị quá lớn hoặc quá nhỏ so với dải giá tri cho phép của kiểu khai báo. Ta có thể dùng toán tử liên quan đến overflow của Swift trong trường hợp này, xem chi tiết [Overflow Operators.](../advanced_operators/overflow-operators.md)

Swift cũng cung cấp hai toán tử tạo dải giá trị (range operators) `a..<b` và `a...b`, hai toán tử này không có trong C.

Chương này sẽ bàn chi tiết về các toán tử thông dụng trong Swift, trong chương [Advanced Operator](../advanced_operators/) ta sẽ tìm hiểu thêm về các toán từ khó hơn, và học các để định nghĩa toán tử mới, hay định nghĩa lại các toán tử đã có cho các kiểu giá trị mới.

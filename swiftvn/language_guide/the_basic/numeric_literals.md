# Numeric Literals

Số cỏ thể được viết như sau:

* Số thập phân (hệ số 10), không có tiền tố
* Số nhị phân (hệ số 2), với tiền tố `0b`
* Số bát phân (hệ số 8), với tiền tố `0o`
* Số thập lục (hệ số 16), với tiền tố `0x`

Ví dụ khi viết số `17` bằng các cách viết trên:

```swift
let decimalInteger = 17
let binaryInteger = 0b10001
let octalInteger = 0o21
let hexadecimalInteger = 0x11
```

Số dấu phẩy động có thể là số thập phân, hoặc thập lục (hexa, với prefix 0x). Khi viểt số dấu phảy động, phải luôn có chữ sô trước và sau dấu chấm thập phân. Số thập phân động (thập phân dấu phảy động) có thể có hoặc không có phần số mũ (exponent), phần này được đánh dấu bằng e hoặc E. Với số hệ hexa động, phần số mũ đánh dấu bằng p hoặc P.

Đối với số thập phân, phần mũ được tính theo số mũ của 10:

* `1.25e2` có giá trị `1.25 x 10^2`, hay `125.0`
* `1.25e-2` có giá trị `1.25 x 10^-2`, hay `0.0125`

Đối với số hexa, phần mũ được tính theo hệ só 2

* `0xFp2` có giá trị `15 x 2^2` hay `60.0`
* `0xFp-1` có giá trị `15 x 2^-2` hay `3.75`

Tất cả các cách viết dấu phẩy động với giá trị `12.1875`:

```swift
let decimalDouble = 12.1875
let exponentDouble = 1.21875e1
let hexadecimalDouble = 0xC.3p0
```

Cách viết số có thể có các kí tự để format làm cho dễ đọc hơn. Cả số nguyen và số phẩy động có thể được thêm các số `0` hoặc `_` để cho dễ đọc hơn. Giá trị của số không bị ảnh hưởng bới các kí tự thêm này:

```swift
let paddedDouble = 000123.456
let oneMillion = 1_000_000
let justOverOneMillion = 1_000_000.000_000_1
```

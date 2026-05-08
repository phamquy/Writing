# Generics

Viết têm trong `<>` để khai báo một hàm hoặc kiểu generic

```swift
func repeatItem<Item>(item: Item, numberOfTimes: Int) -> [Item] {
    var result = [Item]()
    for _ in 0..<numberOfTimes {
        result.append(item)
    }
    return result
}
repeatItem("knock", numberOfTimes:4)
```

Ta có thể định nghĩa dạng generic cho: function, method, class, enumeration và structure

```swift
enum OptionalValue<Wrapped> {
    case None
    case Some(Wrapped)
}
var possibleInteger: OptionalValue<Int> = .None
possibleInteger = .Some(100)
```

Dùng `where` để khai báo ràng buộc, chẳng hạn như tham sô generic phải thuốc kiểu protocol, hoặc hai tham số generic phải giống nhau, hay tham số generic phải kế thừa từ một lớp cha nào đó...

```swift
func anyCommonElements <T: SequenceType, U: SequenceType where T.Generator.Element: Equatable, T.Generator.Element == U.Generator.Element> (lhs: T, _ rhs: U) -> Bool {
    for lhsItem in lhs {
        for rhsItem in rhs {
            if lhsItem == rhsItem {
                return true
            }
        }
    }
    return false
}
anyCommonElements([1, 2, 3], [3])
```

> **THỰC HÀNH**
>
> Sửa `anyCommonElements(_:_:)` sao cho no trả về mảng chung của hai giá trị kiểu `SequenceType`

Viết `<T: Equatable>` cũng giống `<T where T: Equatable>`

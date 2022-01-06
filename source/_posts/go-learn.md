---
title: go入门和避坑指南
date: 2021-10-21 22:30:02
tags:
- Go
- 后端
categories: 后端
---

### nil在接口和值类型的区别
Go的interface底层是通过`(type, value)`来实现的，
`value`被称为接口的动态值，它是一个任意的具体值，而该`type`则为该值的类型。对于 int 值 3， 一个接口值包含 (int, 3)。
只有当value和type都为nil，interface才为nil。下面举例来说明：

```go
func main() {
    var a interface{}
    var p *int = nil
    a = p
    fmt.Println(a == nil)        // false
    fmt.Println(a.(*int) == nil) // true
}
```

```go
func main() {
    // 永远返回err != nil
    if err := returnError(); err != nil {
        fmt.Println(err)
    }
}
func returnsError() error {
    var p *MyError = nil
    if bad() {
        p = ErrBad
    }
    return p // Will always return a non-nil error.
}
```

未完待续..

---
title: "浅谈 Go 语言的 unsafe 包"
date: 2020-06-02T09:29:44+08:00
draft: false
---

![unsafe](/go/unsafe.png)
<!--more-->

> Unsafe 是 Go 语言最底层的元语。

## 安全的指针

Go 的指针是安全，具体是指：

1. Go 指针不能进行算术运算
对于一个指针`p`，`p++` 和 `p+10` 都是非法的。
```go
i := 1
p := &i
p++ // [compiler] [E] invalid operation: p++ (non-numeric type *int)
```

2. Go 指针赋值是类型安全的[^1]

值 x （类型 V ）赋值给类型 T 必须满足以下任意条件
* 类型 V 和类型 T 相同
* 类型 V 和类型 T 的底层类型相同，并且 V 或者 T 至少一个是非自定义类型

```go
type MyInt int64
type Ta *int64
type Tb *MyInt
```

这里
* MyInt 和 int64 可以相互隐式转换和赋值（基于第二条原则，底层类型都是 int64 ），
Ta 和 `*int64`，Tb 和 `*MyInt` 也是如此。
* Ta 和 Tb 不可以相互转换和赋值（Ta 的底层类型是 `*int64`，Tb 的底层类型是 `*MyInt`），
但可以通过多次的显式类型转换`(*MyInt)((*int64)(Ta))`进行赋值。

## 不安全的指针

unsafe 包打破了以上安全限制，利用 unsafe 包可以算术运算也能绕过类型系统做类型转换和赋值。
正所谓能力越大，责任越大。
unsafe 包容易被错误的使用并且其代码不受 Go 兼容性保证[^2]。
因此建议只有在特定的场景下才考虑使用 unsafe 包。
在理解了 unsafe 包的适用场景下，我们来看看 unsafe 包提供了哪些工具。

## unsafe 工具箱

unsafe 工具箱提供了三个方法两个类型，分别是：

### Alignof
方法`Alignof` 接受任意类型返回类型内存对齐大小。返回值与 `reflect.TypeOf(x).Align()` 相同，如果是 struct x 的一个字段 f，返回值与 `reflect.TypeOf(x.f).FieldAlign()`。

Go 语言规定[^3]:
* 对于一个变量 x ，`unsafe.Alignof(x)` 最小值为 1 。
* 对于一个结构体变量 x ，`unsafe.Alignof(x)` 返回值是各个字段 f 中的最大值 `reflect.Alignof(x.f)`，最小值是 1 。
* 对于一个数组变量 x ，`unsafe.Alignof(x)` 与数组元素的内存对齐大小相同。

### Offsetof
方法`Offsetof` 接受任意类型结构体 x 的字段 f，返回内存的偏移值。

### Sizeof
方法`Sizeof` 接受任意类型 x，返回内存大小。

### Pointer
类型`Pointer` 支持四种特殊的操作：
* 任意类型的指针都可以转换成 Pointer
* Pointer 可以转换成任意类型的指针
* uintptr 可以转换成 Pointer
* Pointer 可以转换成 uintptr

### uintptr
类型`uintptr` 是内建[^4]类型，用于保存地址的整数。

## unsafe 最佳实践

1. 类型 T1 转换成类型 T2

如果 T1 和 T2 有相同的内存布局，那么可以将 T1 转换成 T2。
```go
// package math
func Float64bits(f float64) uint64 {
	return *(*uint64)(unsafe.Pointer(&f))
}
```

2. 指针算术运算

指针的算术运算的基本用法是：
```go
p = unsafe.Pointer(uintptr(p) + offset)
```

一个获取结构体字段 Pointer 的例子：
```go
package main

import (
	"fmt"
	"unsafe"
)

type Foo struct {
	a int64
	b int16
}

func main() {
    f := &Foo{b: 10}
	p := unsafe.Pointer(uintptr(unsafe.Pointer(f)) + unsafe.Offsetof(f.b))

	fmt.Println(*(*int16)(p))

	// 注意以下代码是不合法的，这里有两个问题：
	// 1. f 转换成 uintptr 后原数据没有引用，有可能会被垃圾回收清理掉
	// 2. f 换成成 uintptr 后是常量，go 运行时有可能更改内存地址导致常量 uintptr 指向错误的地址
	// 所以转成 uintptr 和转回 Pointer 必须在同一行。
	u := uintptr(unsafe.Pointer(f))
	p = unsafe.Pointer(u + 8)
	fmt.Println(*(*int16)(p))
}
```

3. 结合反射使用

一个访问结构体私有字段的例子：
```go
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

type Foo struct {
	name string
}

func main() {
	f := &Foo{}
	// 通常使用反射比直接通过指针运算更加健壮
	fv := reflect.ValueOf(f).Elem().FieldByName("name")
	name := *(*string)(unsafe.Pointer(fv.UnsafeAddr()))
	fmt.Println(name)
}
```

4. Pointer 转换成 uintptr 调用 syscall.Syscall

```go
// package syscall
func Acct(path string) (err error) {
	var _p0 *byte
	_p0, err = BytePtrFromString(path)
	if err != nil {
		return
	}
	_, _, e1 := Syscall(SYS_ACCT, uintptr(unsafe.Pointer(_p0)), 0, 0)
	if e1 != 0 {
		err = errnoErr(e1)
	}
	return
}
```

最后（unsafe）能力越大，（开发者的）责任越大！
![](/go/power-responsibility.gif)

引用链接：
[^1]: https://golang.org/ref/spec#Assignability
[^2]: https://golang.org/pkg/unsafe/#pkg-overview
[^3]: https://golang.org/ref/spec#Size_and_alignment_guarantees
[^4]: https://golang.org/pkg/builtin/#uintptr

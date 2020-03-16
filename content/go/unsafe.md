---
title: "Unsafe"
date: 2020-03-07T09:29:44+08:00
draft: true
---

> Go 指针在绝大多数下是安全，剩下的那部分就在 `unsafe` 包里。


## `Alignof`
unsafe.Alignof(x) 和 reflect.TypeOf(x).Align() 相同

unsafe.Alignof(s.f) 和 reflect.TypeOf(s.f).FieldAlign() 相同

## `Offsetof`

```go
type S struct {
  f string
}
unsafe.Offsetof(s.f)
```

## `Sizeof`
```go
func stringArr() {
	arr := [5]string{"H", "e", "l", "l", "o"}
	p := unsafe.Pointer(&arr)
	n := unsafe.Sizeof("")

	println(*(*string)(unsafe.Pointer(uintptr(p) + n)))
}

func stringSlice() {
	slice := []string{"H", "e", "l", "l", "o"}
	p := unsafe.Pointer(&slice[0]) // 这边取第一个值，其实是为了拿到内部array的地址，参考 slice 的内存结构
	n := unsafe.Sizeof("")

	println(*(*string)(unsafe.Pointer(uintptr(p) + n)))
}
```

runtime/slice.go
```go
// slice 内存结构
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

runtime/string.go
```go
// string 内存结构
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```

reflect/value.go
```go
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}

type StringHeader struct {
	Data uintptr
	Len  int
}
```

Conversion of a Pointer to a uintptr (but not back to Pointer). Conversion of a uintptr back to Pointer is not valid in general.
uintptr 保存的是整数不是指针，做类型转化到 unsafe.Pointer 时，对象的实际地址可能已经转移。

指针地址计算
f := unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.f))





Reference:
* https://golang.org/pkg/unsafe/
* https://go101.org/article/unsafe.html
* https://halfrost.com/go_slice/






  def seller_timezone_name
    @order.seller_timezone_name.presence || 'Pacific Time (US & Canada)' # Default to GOAT Time of timezone not set.
  end



Go errors
=========

errors.Is 类似于值比较，但是会检查整个链上的错误
errors.As 类似于类型推断，但是会检查整个链上的错误

要使用 Is As，必须实现 Unwrap 方法
要更换 Is 的行为，需要自定义 Is 方法
要更换 As 的行为，需要自定义 As 方法

1.13 之前只有 string

1.13 之后可以包含(wrap) error

```go
func Unwrap(err error) error {
	u, ok := err.(interface {
		Unwrap() error
	}) // 有没有实现 Unwrap 方法, duck type?
	if !ok {
		return nil
	}
	return u.Unwrap()
}
```



```go
func Is(err, target error) bool {
	if target == nil {
		return err == target
	}

	isComparable := reflectlite.TypeOf(target).Comparable()
	for {
		if isComparable && err == target { // 如果能比较，并且 err 与 target 相等
			return true
		}
		if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) { // 如果实现了 Is 方法，用 Is 来比较
			return true
		}
		if err = Unwrap(err); err == nil { // Unwrap 返回上一层 err
			return false
		}
	}
}
```

```go
func As(err error, target interface{}) bool {
	if target == nil { // target 不能是nil
		panic("errors: target cannot be nil")
	}
	val := reflectlite.ValueOf(target)
	typ := val.Type()
	if typ.Kind() != reflectlite.Ptr || val.IsNil() {
		panic("errors: target must be a non-nil pointer")
	}
	if e := typ.Elem(); e.Kind() != reflectlite.Interface && !e.Implements(errorType) {
		panic("errors: *target must be interface or implement error")
	}
	targetType := typ.Elem()
	for err != nil {
		if reflectlite.TypeOf(err).AssignableTo(targetType) {
			val.Elem().Set(reflectlite.ValueOf(err))
			return true
		}
		if x, ok := err.(interface{ As(interface{}) bool }); ok && x.As(target) {
			return true
		}
		err = Unwrap(err)
	}
	return false
}
```

io.ErrUnexpectedEOF
errors.Is






### Reference
1. https://blog.golang.org/go1.13-errors

Go errors
========

> Go 一直把错误作为一个普通的值，一个包含一段文本信息的类型为错误的值，而这在 1.13 版本之后发生了改变。

## 在 1.13 之前
Go 中任意类型只要实现了 `error` 接口的都可以称为错误。
```go
type error interface {
	Error() string
}
```

程序中通常会直接用 `errors.New` 生成错误，调用者拿到错误的时候可以直接用 `==` 做错误判断：
```go
var ErrNotFound = errors.New("not found")
if err == ErrNotFound {
	// 错误处理
}
```

当简单的错误不能满足业务要求时，也会自己定义 `struct` 来实现 `error` 接口，然后需要通过类型断言 (type assertion or type switch) 做错误判断：
```go
type NotFoundError struct {
    Name string
}

func (e *NotFoundError) Error() string { return e.Name + ": not found" }

if e, ok := err.(*NotFoundError); ok {
	// 错误处理
}
```

当要传递额外信息的错误时，通常是用 `fmt.Errorf` 来包含原来的错误：
```go
if err != nil {
	return fmt.Errorf("登陆失败: %v", err)
}
```
但上述方法只能包含原有错误中的文本信息 (message)，失去了原有错误的其他信息，包括类型以及其他自定义的信息。调用者也失去了处理原有错误的机会，只能基于文本做分析处理。

## 新的 API

问题出在了 `fmt.Errorf` 返回的时候失去了内部错误的信息，保留的文本信息在简单的 debug 场景下是够用了，到复杂系统中，这样的信息是远远不够的。

Go 在 1.13 版本中带来了 error wrapping，简单的说就是在错误传递的过程中，新生成的错误中可以包含完整的老的错误，就像洋葱一样的层层包裹。
![](./../images/error_wrapping.png)

 `fmt.Errorf` 新的 `%w` 生成包裹的错误:
```go
newErr := fmt.Errorf("access error: %w", permissionErr)
```

`errors.Unwrap` 剥开一层返回内部的错误，这个方法要求包裹的错误实现 `Unwrap` 方法，用 `fmt.Errorf` 生成的错误已经实现了该方法：
```go
permissionErr := errors.Unwrap(newErr)
```

`errors.Is` 代替 `==` 判断错误，会层层剥开去判断错误是否相等
```go
// 类似 if err == ErrNotFound {
if errors.Is(err, ErrNotFound) {
	// 错误处理
}
```

`errors.As` 代替类型推断判断错误，会层层剥开去判断类型是否相同，如果相同，将 err 赋值给目标类型
```go
// if e, ok := err.(*NotFoundError); ok {
var e *NotFoundError
if errors.As(err, &e) {
	// 错误处理
}
```

## 扩展

可以通过实现 `Is(err error) bool` 方法扩展 `errors.Is` 的比较逻辑，考虑下面的例子
```go
type Error struct {
    Path string
    User string
}

func (e *Error) Is(target error) bool {
    t, ok := target.(*Error)
    if !ok {
        return false
    }
    return (e.Path == t.Path || t.Path == "") &&
           (e.User == t.User || t.User == "")
}

if errors.Is(err, &Error{User: "someuser"}) {
    // err's User field is "someuser".
}
```
`errors.As` 方法也是同样可以用 `As` 方法来扩展。


## 源码解析

```go
func Unwrap(err error) error {
	u, ok := err.(interface {
		Unwrap() error
	}) // duck type 判断错误有没有实现 Unwrap 方法
	if !ok {
		return nil // 没有实现返回 nil
	}
	return u.Unwrap() // 调用 Unwrap 方法返回内部错误
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
		if err = Unwrap(err); err == nil { // Unwrap 返回内部错误
			return false
		}
	}
}
```

```go
func As(err error, target interface{}) bool {
	if target == nil { // target 不能是 nil
		panic("errors: target cannot be nil")
	}
	val := reflectlite.ValueOf(target)
	typ := val.Type()
	if typ.Kind() != reflectlite.Ptr || val.IsNil() { // target 不能是个空指针
		panic("errors: target must be a non-nil pointer")
	}
	if e := typ.Elem(); e.Kind() != reflectlite.Interface && !e.Implements(errorType) { // target 必须是接口或者是实现了 error 接口的 struct
		panic("errors: *target must be interface or implement error")
	}
	targetType := typ.Elem()
	for err != nil {
		if reflectlite.TypeOf(err).AssignableTo(targetType) { // https://golang.org/ref/spec#Assignability
			val.Elem().Set(reflectlite.ValueOf(err))
			return true
		}
		if x, ok := err.(interface{ As(interface{}) bool }); ok && x.As(target) { // 如果定义 As 方法，通过 As 方法转换
			return true
		}
		err = Unwrap(err) // Unwrap 返回内部错误
	}
	return false
}
```

### Reference
1. https://blog.golang.org/go1.13-errors
2. https://golang.org/ref/spec#Assignability
3. https://github.com/golang/go/wiki/ErrorValueFAQ
4. https://golang.org/doc/go1.13
Go errors
=========

errors.Is 类似于值比较，但是会检查整个链上的错误
errors.As 类似于类型推断，但是会检查整个链上的错误

要使用 Is As，必须实现 Unwrap 方法
要更换 Is 的行为，需要自定义 Is 方法
要更换 As 的行为，需要自定义 As 方法

1.13 之前只有 string

1.13 之后可以包含(wrap) error



### Reference
1. https://blog.golang.org/go1.13-errors

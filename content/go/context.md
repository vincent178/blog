---
title: "Context"
date: 2020-02-17T06:37:22+08:00
draft: true
---

Context 接口

例子1

timeout


```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)

	Done() <-chan struct{}

	Err() error

	Value(key interface{}) interface{}
}
```
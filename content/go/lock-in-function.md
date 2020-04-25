---
title: "Lock in Function"
date: 2020-04-26T05:26:13+08:00
draft: true
---

锁在 go 的代码中很常见。并发编程中锁保护对共享内存的读取和改动，不会出现竞争（race condition）。

```go
type person struct {
  mux  sync.Mutex
  name string 
  // ...
}

func (s *person) UpdateName(name string) error {
  s.mux.Lock()
  defer s.mux.Unlock()

  // ...

  s.name = name

  // ...
}
```

高效使用锁的一个要点就是锁变量不锁过程，也就是只锁需要在并发下获取或改变的变量。

根据这个优化一下上面的代码，移除 `defer` 代码，只在变量的上下调用锁。

```go
func (s *person) UpdateName(name string) error {
  // ...

  s.mux.Lock()
  s.name = name
  s.mux.Unlock()

  // ...
}
```

但因为移除了 `defer` 代码块，提高了维护成本。在复杂的代码情景下，会有可能忘了调用 `Unlock` 或者在 `Lock` 和 `Unlock` 中间插入了不应该由锁保护的代码导致性能下降。

那么有没有办法保留 `defer` 同时只保护 `name` 变量呢？引入一个新的立即执行函数。
```go
func (s *person) UpdateName(name string) error {
  // ...

  func() {
    s.mux.Lock()
    defer s.mux.Unlock()

    s.name = name
  }()

  // ...
}
```


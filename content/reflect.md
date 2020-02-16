---
title: "Reflect"
date: 2020-02-15T13:35:28+08:00
draft: true
---

`CanSet`比CanAddr只加了一个限制，就是struct类型的unexported的字段不能Set，所以我们这节主要介绍CanAddr。

https://blog.golang.org/laws-of-reflection

---
title: "Versioning"
date: 2020-03-16T16:15:05+08:00
draft: true
---

Semantic version 通过版本号描述兼容性

Go 中的定义
"If an old package and a new package have the same import path, the new package must be backwards compatible with the old package."


如果老的包和新的包的import路径一致，那么新的包必须兼容老的包

go 不会主动的升级旧版本到新版本。

为了保持 import 兼容性，go 要求模块在 v2 和以后的版本使用大版本作为路径的最后元素






* https://semver.org/lang/zh-CN/

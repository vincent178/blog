---
title: "Kubernetes CRD"
date: 2020-04-08T06:35:04+08:00
draft: true
---

Finalizer: implement asynchronous pre-delete hooks

必须保证 finalizer 都执行完毕后才算删除完成。
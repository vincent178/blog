---
title: "API"
date: 2020-02-29T20:25:31+08:00
draft: true
---

```go
type Object interface {
	GetObjectKind() schema.ObjectKind
	DeepCopyObject() Object
}

// 用来保存 plugin 信息
// 因为 plugin 的类型是未知的
// 所以 RawExtsion 
type RawExtension struct {
  // 保存序列化内容
  Raw []byte `json:"-" protobuf:"bytes,1,opt,name=raw"`
  // 保存 Object 信息
	Object Object `json:"-"`
}

type Unknown struct {
  TypeMeta `json:",inline" protobuf:"bytes,1,opt,name=typeMeta"`

  Raw []byte `protobuf:"bytes,2,opt,name=raw"`

  ContentEncoding string `protobuf:"bytes,3,opt,name=contentEncoding"`

	ContentType string `protobuf:"bytes,4,opt,name=contentType"`
}
```
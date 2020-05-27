---
title: "Collection_cachekey"
date: 2020-03-16T11:54:58+08:00
draft: true
---

to_sql 计算 signatuere

计算 cache version

如果已经加载到内存 就直接计算 size 和 timestmap

如果没有加载到内存

如果有 limit 或者 offset
那么从collection 中 select timestmap

如果没有直接count`"COUNT(*) AS #{connection.quote_column_name("size")}, MAX(%s) AS timestamp"` 


 "#{key}-#{compute_cache_version(timestamp_column)}"

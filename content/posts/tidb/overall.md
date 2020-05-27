---
title: "Overall"
date: 2020-03-07T20:06:22+08:00
draft: true
---

tiup
```
tiup run playground nightly
```

TiKV 节点（Store）

调度
https://zhuanlan.zhihu.com/p/27275483
https://zhuanlan.zhihu.com/p/24809131
https://asktug.com/t/tidb-pd/1388
https://asktug.com/t/tidb-pd/1669
https://cloud.tencent.com/developer/news/303534


TiFlash
https://en.wikipedia.org/wiki/Log-structured_merge-tree



qdisc: queueing disciplines
tbf: token bucket filter


apt-get install iproute2

https://www.tldp.org/HOWTO/Adv-Routing-HOWTO/lartc.qdisc.classless.html
https://www.tldp.org/HOWTO/Adv-Routing-HOWTO/lartc.qdisc.classful.html
http://man7.org/linux/man-pages/man8/tc-tbf.8.html


```c
struct tc_tbf_qopt {
	struct tc_ratespec rate;
	struct tc_ratespec peakrate;
	__u32		limit;
	__u32		buffer;
	__u32		mtu;
};

```

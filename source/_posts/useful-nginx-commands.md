title: 好用的 nginx 命令
date: 2015-10-15 09:16:36
tags:
thumbnailImage: http://res.cloudinary.com/dudqzxsbb/image/upload/v1444872133/nginx-logo-big_q94mmh.png
---
以默认方式启动
```
$ /usr/local/nginx/sbin/nginx
```

使用-c参数指定配置文件
```
$ /usr/local/nginx/sbin/nginx -c /tmp/nginx.conf
```

测试配置文件是否有错误
```
$ /usr/local/sbin/nginx -t
```

强制停止nginx
```
$ /usr/local/nginx/sbin/nginx -s stop
or
$ kill -s SIGTERM 10800 #=> 10800 是nginx master的进程ID
```

优雅停止nginx
```
$ /usr/local/nginx/sbin/nginx -s quit
```

使运行中的nginx重读配置项并生效
```
$ /usr/local/nginx/sbin/nginx -s reload
```

配置项的单位
* K或者k千字节(KiloByte, KB)
* M或者m兆字节(MegaByte, MB)
* 时间单位: ms, s, m, h, d(day), w(week), M(month), y(year)

匹配"/search"全部跳转404
```
location ^~ /search {
  return 404;
}
```




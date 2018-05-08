---
title: Node 网络(tcp)编程笔记
date: 2018-05-08 09:02:46
tags:
---

<div style="display: flex; justify-content: center; align-items: center;">
  ![](images/16-sockets-logical.png)
</div>

<!-- more -->

> 最近看 WebSocket，把涉及到的 tcp socket 这一层Node源代码梳理了一下
>
> 有说的不对的地方，希望大家指正


# 代码
Node 版本： 8.9.4
```javascript
const server = new net.Server((socket) => {
  console.log(socket);
});

server.listen('1234', () => {
  console.log('server started at 1234');
});
```

# Socket

服务器 socket 是用于监听连接而不是发起连接，其生命周期通常由创建，绑定，监听，连接，关闭组成的。

为了便于说明，把上面代码划分为四块：
```
1) new net.Server()
2) socket => console.log(socket)
3) server.listen()
4) () => console.log('server started at 1234')
```

# 服务器启动流程

运行代码, 我们可以通过 `lsof` 看到一个 socket 处于 LISTEN 状态, 这就是服务器 socket.

```
$ lsof -i tcp:1234

COMMAND   PID    USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
node    64852 i312714   14u  IPv6 0xe781f0e6b2c52e5b      0t0  TCP *:search-agent (LISTEN)
```

在代码块1 我们初始化了 net.Server()，具体代码在 net.js L1176, 这里为 精简文章，只贴出了关键代码，后面都会只贴出关键代码。

```javascript
function Server(options, connectionListener) {
  this.on('connection', connectionListener);
  this._connections = 0;

  this[async_id_symbol] = -1;
  this._handle = null;
  this._usingWorkers = false;
  this._workers = [];
  this._unref = false;

  this.allowHalfOpen = options.allowHalfOpen || false;
  this.pauseOnConnect = !!options.pauseOnConnect;
}
util.inherits(Server, EventEmitter);
```

这里先是注册了 connection 事件监听回调connectionListener，也就是代码块2

然后初始化了 _handle, _usingWorkers, _workers, _unref 等属性

并且让 Server 继承 EventEmitter

# 服务器监听

按照代码运行顺序，下一步是执行代码块3，调用了server.listen， 源代码代码在 net.js L1411
```javascript
Server.prototype.listen = function(...args) {
  var normalized = normalizeArgs(args);
  var options = normalized[0];
  var cb = normalized[1];

  var backlog;
  if (typeof options.port === 'number' || typeof options.port === 'string') {
    if (!isLegalPort(options.port)) {
      throw new ERR_SOCKET_BAD_PORT(options.port);
    }
    backlog = options.backlog || backlogFromArgs;
    // start TCP server listening on host:port
    if (options.host) {
      lookupAndListen(this, options.port | 0, options.host, backlog,
                      options.exclusive);
    } else { // Undefined host, listens on unspecified address
      // Default addressType 4 will be used to search for master server
      listenInCluster(this, null, options.port | 0, 4,
                      backlog, undefined, options.exclusive);
    }
    return this;
  }
};
```

省略中间代码追述，调用了 setupListenHandle
```javascript
function setupListenHandle(address, port, addressType, backlog, fd) {
  rval = createServerHandle(address, port, addressType, fd);
  this._handle = rval;
  this[async_id_symbol] = getNewAsyncId(this._handle);
  this._handle.onconnection = onconnection;
  this._handle.owner = this;

  this._handle.listen(backlog || 511);
}
```
关键代码第一步是 createServerHandle, 源代码如下
```javascript
function createServerHandle(address, port, addressType, fd) {
  handle = new TCP(TCPConstants.SERVER);
  isTCP = true;
  err = handle.bind(address, port);
  return handle;
}
```
重点来了:

`TCP` 是从 `tcp_wrap` 模块中引入的，`tcp_wrap` 是一个C++写的Node模块, TCPWrap 的继承链是 TCPWrap < ConnectionWrap < LibuvStreamWrap < HandleWrap, StreamBase < HandleWrap

在 tcp_wrap 的 Initialize 中定义了 TCP 的 open, bind, listen 等方法

`createServerHandle` 中的 new TCP() 是调用 tcp_wrap TCPWrap::New, 最后会返回一个 TCPWrap 的实例, 这里主要是初始化了 libuv 中 `uv_handle_t`

`handle.bind()` 调用 TCPWrap::Bind, 这里调用了 libuv 中 uv__tcp_bind 这里给之前初始化的 handle 设置了 socket, 并且绑定了host，port等信息

这时候在js中访问 `handle.fd` 就会返回真实的文件描述符，应该与 `lsof` 中看到的 `FD` 一致的

回到 `setupListenHandle` 中

`handle.onconnection = onconnection` 这个稍后解释

`handle.listen()` 调用了 TCPWrap::Listen, 这里调用了 libuv 中的 uv_listen(uv_tcp_listen)，最终调用系统的 `listen` 方法, 并且注册了 `uv__server_io` 回调

以上完成了Socket 的创建，绑定和监听。

# 服务器连接

服务器接收连接之后就会触发代码块2，在代码块2中就可以拿到连接socket，从中可以读取数据或者写入数据。

在 new net.Server() 时，这个回调作为 connection 事件绑定到 server 上。

在创建服务器socket的时候，`this._handle.onconnection = onconnection` 这个onconnection 就会最终 emit connection 事件。
```javascript
function onconnection(err, clientHandle) {
  var handle = this;
  var self = handle.owner;

  var socket = new Socket({
    handle: clientHandle,
    allowHalfOpen: self.allowHalfOpen,
    pauseOnCreate: self.pauseOnConnect
  });
  socket.readable = socket.writable = true;


  self._connections++;
  socket.server = self;
  socket._server = self;
  self.emit('connection', socket);
}
```
回到调用 uv_listen 的时候的代码
```
  int err = uv_listen(reinterpret_cast<uv_stream_t*>(&wrap->handle_),
                      backlog,
                      OnConnection);
```

这里的 OnConnection 其实是 ConnectionWrap<WrapType, UVType>::OnConnection 这个方法，这个方法里面会调用 js 的 `onconnection` 回调。

OnConnection 这个方法里面 生成了新的 socket 并且赋值给了 argv 最终传递给了 `onconnection`

```cpp
template <typename WrapType, typename UVType>
void ConnectionWrap<WrapType, UVType>::OnConnection(uv_stream_t* handle,
                                                    int status) {
  ...
  ...

  if (status == 0) {
    // Instantiate the client javascript object and handle.
    Local<Object> client_obj = WrapType::Instantiate(env,
                                                     wrap_data,
                                                     WrapType::SOCKET);

    WrapType* wrap;
    ASSIGN_OR_RETURN_UNWRAP(&wrap, client_obj);
    uv_stream_t* client_handle =
        reinterpret_cast<uv_stream_t*>(&wrap->handle_);

    if (uv_accept(handle, client_handle))
      return;

    argv[1] = client_obj;
  }
  // connection_string 就是 "onconnection"
  wrap_data->MakeCallback(env->onconnection_string(), arraysize(argv), argv);
}
```
到这里基本说明了Node在连接时候的代码逻辑。后面也会继续整理一下关于socket close的代码。
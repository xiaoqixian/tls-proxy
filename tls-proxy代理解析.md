# tls-proxy代理解析

[tls-proxy](https://github.com/TanNang/tls-proxy.git)是我在github找到的一个展示代理原理的软件，总共不到两千行代码。所以用于学习代理的原理非常合适。

tls-proxy分为服务端和客户端。服务端的任务是负责转发来自客户端的请求，而客户端则负责将用户的请求发送给服务端。tls-proxy同时**支持TCP连接和UDP连接**，但是UDP流量使用TCP进行传输，和V2ray一样，这点作者在项目介绍上也提到了。

### event2 lib

event2为Libenvent库的头文件，Libevent为C语言下一个著名的事件库。所谓事件库，可以用于完成一条线程的从创建到赋予任务，到事件通知，再到循环处理。当你需要用线程去完成一些随机到达的任务时，事件库就是最好的选择。显然，接受用户的请求并完成转发是典型的需要用到事件库的地方。

### uthash lib

uthash是一个用于将C结构体存储于哈希表的哈希库，这里是它的[官网介绍界面](https://troydhanson.github.io/uthash/)。

当需要将某个结构体存储于哈希表时，需要在该结构体定义一个`UT_hash_handle` 类型的属性。在哈希表中添加某个结构体实例时，需要自己确定具体的id, 并通过该id来进行查询。

具体的添加项和查询项的方法可以查看官网。

### tls-client

从客户端来看，当你在操作系统上配置了在某个端口的代理之后，你所进行的一切网络连接都会从该端口发出。

因此客户端需要监听所有从该端口发出的连接，由于TCP和UDP的连接属于两套不同的逻辑，因此我们分开讨论。

#### TCP connections

TCP和UDP的发起函数都是 `service` 函数，`service` 接受一个参数，用于指定接收哪种类型的连接。

对于TCP连接来说，会首先创建一个`tcplistener`，`tcplistener`用于监听某个地址到来的TCP连接，这里的地址当然就是本地地址 `127.0.0.1`。

当有新的TCP连接到来时，`tcplistener`会调用回调函数 `tcp_newconn_cb`。

##### `tcp_newconn_cb`

`tcp_newconn_cb`的类型是 `evconnlistener_cb(struct evconnlistener*, evutil_socket_t, struct sockaddr*, int socklen, void*)`

其中第二个参数`evutil_socket_t`其实就是`int`的socket fd, 这个socket fd属于新到来的TCP连接创建时建立的socket fd. 

有了这个socket fd, 我们就能通过 `getsockname` 函数获取其原本想要建立连接的目的地地址。比如，浏览器想要通过google搜索东西，就建立了与google的TCP连接，但是因为用户打开了代理，所以连接从这个代理端口发出，之后被 `tcplistener` 获取之后，就通过对应socket fd 知道用户想要建立与google的连接，之后代理软件的客户端才能告诉服务端需要与哪个地址建立连接，所以这个参数是必不可少的。


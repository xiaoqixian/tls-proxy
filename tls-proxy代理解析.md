# tls-proxy代理解析

[tls-proxy](https://github.com/TanNang/tls-proxy.git)是我在github找到的一个展示代理原理的软件，总共不到两千行代码。所以用于学习代理的原理非常合适。

tls-proxy分为服务端和客户端。服务端的任务是负责转发来自客户端的请求，而客户端则负责将用户的请求发送给服务端。tls-proxy同时**支持TCP连接和UDP连接**，但是UDP流量使用TCP进行传输，和V2ray一样，这点作者在项目介绍上也提到了。

**阅读本文章最好具有计算机网络和socket编程的基础。**

另外，tls-proxy 对于客户端与服务器的连接在应用层上其实是建立的 https 的安全连接，其中会涉及到许多 ssl 的知识，如果掺杂在一起讲会显得比较乱，所以我在本篇去除ssl相关的部分讲，将其看成是http的连接，也无大碍。

[TOC]

### event2 lib

event2为Libenvent库的头文件，Libevent为C语言下一个著名的事件库。所谓事件库，可以用于完成一条线程的从创建到赋予任务，到事件通知，再到循环处理。当你需要用线程去完成一些随机到达的任务时，事件库就是最好的选择。显然，接受用户的请求并完成转发是典型的需要用到事件库的地方。

如果想要深入了解libevent可以阅读这本[《libevent深入浅出》](https://aceld.gitbooks.io/libevent/content/)，对于学习并发模式下的I/O模型有很大的帮助。

#### bufferevent lib

libevent是一个非常庞大的库，bufferevent是其下面的一个用于监视缓存的库。基本原理也与libevent整体的思路非常相似，通过监视一片缓存的读与写，但可读或可写或者有其它事件发生时，分别调用不同的回调函数进行处理。

这里说的缓存并不一定是指用户分配的一片内存，也可以指一个文件描述符。譬如bufferevent也可以监视一个socket fd, 当其有内容到达时，就可以调用相应的回调函数。

### uthash lib

uthash是一个用于将C结构体存储于哈希表的哈希库，这里是它的[官网介绍界面](https://troydhanson.github.io/uthash/)。

当需要将某个结构体存储于哈希表时，需要在该结构体定义一个`UT_hash_handle` 类型的属性。在哈希表中添加某个结构体实例时，需要自己确定具体的id, 并通过该id来进行查询。

具体的添加项和查询项的方法可以查看官网。

### proxy principles on your local machine

首先我们要了解一下代理与正常的网络连接在本机上的不同。很多人都知道最简单的代理在互联网层面就是通过一台中间服务器进行请求转发，但是在本机上是怎么实现所有程序发出的请求都通过代理端口发出去呢？（这里仅关注linux平台，windows平台暂时未了解）

这里就要提到linux平台上的TAP/TUN设备文件，TAP/TUN设备是一种虚拟网络设备，它以文件的形式模拟出网卡，当我们在本机上打开代理功能时，所有程序会默认通过这个虚拟网卡发出。

下图是我在网上找到的一个基于UDP的代理软件的原理图。

![img](https://upload-images.jianshu.io/upload_images/1918847-ac4155ec7e9489b2.png?imageMogr2/auto-orient/strip|imageView2/2/w/840/format/webp)

最右边的Socket API就是正常的程序建立的socket, 其通过三种协议中的一种将数据发送到内核的网络协议栈之后，这些数据会到达tunX设备。如果你在本机上打开了代理软件并且端口与操作系统上设置的一致。则代理软件就可以从/dev/tunX设备文件读取到这些数据，然后再通过自己建立的socket正式从内核网络协议栈再到真正的网卡eth0发送出去。这样就完成了所有程序的请求的转发。

### tls-client

从客户端来看，当你在操作系统上配置了在某个端口的代理之后，你所进行的一切网络连接都会从该端口发出。

因此客户端需要监听所有从该端口发出的连接，由于TCP和UDP的连接属于两套不同的逻辑，因此我们分开讨论。

#### TCP connections

TCP和UDP的发起函数都是 `service` 函数，`service` 接受一个参数，用于指定接收哪种类型的连接。

对于TCP连接来说，会首先创建一个`tcplistener`，`tcplistener`用于监听某个地址到来的TCP连接，这里的地址当然就是本地地址 `127.0.0.1`。

当有新的TCP连接到来时，`tcplistener`会调用回调函数 `tcp_newconn_cb`。

##### `tcp_newconn_cb`

`tcp_newconn_cb`的类型是 `evconnlistener_cb(struct evconnlistener*, evutil_socket_t, struct sockaddr*, int socklen, void*)`

注意第二个参数`evutil_socket_t` 在linux平台下其实就是`int`类型的socket fd, 只不过libevent为了跨平台兼容所以进行一层包装，这个socket fd属于新到来的TCP连接创建时建立的socket fd. 

有了这个socket fd, 我们就能通过 `getsockname` 函数获取其原本想要建立连接的目的地地址。比如，浏览器想要通过google搜索内容，就创建了一个socket, 目的地址是google的ip地址，但是还没有正式建立TCP连接，因为用户打开了代理，所以TCP连接从这个代理端口发出，之后就被 `tcplistener` 获取，`tcplistener`就调用`tcp_newconn_cb`， `tcp_newconn_cb`就通过这个socket fd 知道用户想要建立与google的连接，之后代理软件的客户端才能告诉服务端需要与哪个地址建立连接，所以这个参数是必不可少的。

接着看一下当一个新的TCP连接到来时需要做什么。

首先需要确定的一点是当我们使用两个 `bufferevent` 来监督一个socket fd。一个用于监听来自本地要发送的数据，`tls-proxy` 将其命名为 `clntbev`，对应上面的例子即浏览器需要发送给google的数据；另一个用于监听来自目的地址发送过来的数据，对应上面的例子即google发送过来的数据，`tls-proxy` 将其命名为 `destbev`。

`tls-proxy` 为这两个bufferevent设置的回调函数都是 `tcp_events_cb`，但是对应传入的参数不同。

对于回调函数，如果想要传入自定义的参数的话，一般都是 `void*` 类型，并且只有一个。所以如果想要传入多种类型的参数的话，则需要将这些参数包裹在一个结构体中传入，`tls-proxy` 中对应的这个结构体就是 `TCPArg`.

```c
typedef struct {
    char                addr[16];
    char                port[6];
    struct bufferevent *bev;
    char                type;
} TCPArg;
```

可以看到，`TCPArg` 中也有一个 `bufferevent` 类型的`bev`，当为 `clntbev` 设置回调函数的参数时，对应的 `bev` 就是 `destbev`；同样，`destbev` 对应的 `bev` 就是 `clntbev`。

这样做的好处是，不过对于读写处理函数还是事件处理函数，都能够同时获得两个 `bufferevent`，从而能够随时进行数据交换。

![](char.png)

上图是我画的单个TCP连接的数据流向，单条线程可以建立多个TCP连接，但是对于每个TCP连接，都有两个bufferevent负责完成数据交换。可以看到，client bufferevent 和 dest bufferevent 之间是通过内存复制的方式进行数据交换的，而其两端则是通过socket管道进行数据交换。

对于 `addr` 和 `port`， 两个参数都是填入的远端服务器的地址和端口。

在 `tcp_newconn_cb` 的最后，通过 `bufferevent_socket_connect` 为 `destbev` 建立TCP连接，并通过 `setscoopt_tcp` 对TCP连接进行一系列配置。

##### `tcp_events_cb`

前面提到，在还没有正式建立连接的阶段，两个bufferevent设置的回调函数都是 `tcp_events_cb` 函数，这个函数用于处理buffer有可读数据或buffer可写之外的事件。

先看一下函数的结构：

```c
void tcp_events_cb(struct bufferevent* bev, short events, void* arg)
```

`short` 类型的events参数表明事件的类型，不看错误处理的部分代码，则只需要关注 `BEV_EVENT_CONNECTED` 事件。

当建立连接之后，客户端需要做的就是发送请求信息建立连接，具体信息就是http报文的请求头。

发送完请求之后为 `destbev` 可读数据到达时设置的回调函数 `tcp_recvres_cb`，这个函数后面会提到。同时事件到达的处理函数不变。

##### `tcp_recvres_cb`

从名字就可以看出，`tcp_recvres_cb` 函数用于接收在调用 `connect` 后服务器返回的信息，因为可能存在连接未成功建立的情况，主要是确认连接已经成功建立。

忽略真的发生错误连接的代码，当确认连接已经成功建立之后，将 `destbev` 的可读的回调函数设置为 `tcp_read_cb`，事件回调函数不变。

同时我们要开始从应用程序的socket开始接收数据，所以需要设置 `clntbev` 的可读回调函数为 `tcp_read_cb`，事件回调函数为 `tcp_events_cb`，同时设置 `clntbev` 可读（原来之前都是不可读的吗？）。

##### `tcp_read_cb`

如果你通过阅读上面的内容已经对 `tls-proxy` 有了一定的了解，那你应该知道在数据到达 bufferevent 时应该做什么。

首先看一下 `tcp_read_cb` 的函数签名：

```c
void tcp_read_cb(struct bufferevent* bev, void* arg)
```

如果是从服务器到达的数据可达，则这里的 `bev` 是 `destbev`，`arg` 就可以看作是 `TCPArg`类型，其中所包含的 `bev` 就是 `clntbev`。而我们要做的事情就是将 `destbev` buffer里的数据移动到 `clntbev` 中。

在源码中buffer 会分为 input 端和 output端, 我个人认为比较难以区分，你可以简单地认为你读取数据的一端就是 input端，你写入数据的一端就是 output 端。

对于 `destbev`， input 端可以读取到的数据由服务端发送过来，而如果你要写入数据，就将数据写入 output端；对于 `clntbev`，input 端的数据就是应用程序向socket管道发送的数据，当你要将服务器来的数据转发给应用程序时，就将其写入 output 端。

需要注意的是，这里的数据交换用的是 `evbuffer_add_buffer(dst, src)` 函数，调用后 `src` 的数据会直接转移到 `dst` 中，`src` 的数据将不复存在，这样的好处是不需要对 `src` 的内存进行管理了。

`tcp_read_cb` 为了防止 bufferevent 中的数据过多，当 buffer 中的数据量大于 `BUFSIZ_FOR_BEV` 时，将禁止 input 端对应的 bufferevent 的可读性，这样即使数据到达也不会调用回调函数。

接着会将 output 端对应的 bufferevent 的可写的回调函数设置为 `tcp_write_cb`。而该 bufferevent 可写的watermark 在上一步已经被设置为 `BUFSIZ_FOR_BEV/2` ，这说明当 bufferevent 中的数据量小于 这个值时回调函数才会被调用。

##### `tcp_write_cb`

要看懂 `tcp_write_cb` 的功能，需要将 `tcp_read_cb` 和 `tcp_write_cb` 一起看。

```c
void tcp_read_cb(struct bufferevent *bev, void *arg) {
    TCPArg *thisarg = arg;
    struct evbuffer *input = bufferevent_get_input(bev);
    struct evbuffer *output = bufferevent_get_output(thisarg->bev);
    evbuffer_add_buffer(output, input); 
    if (evbuffer_get_length(output) >= BUFSIZ_FOR_BEV) {     
        TCPArg *othrarg = NULL;
        bufferevent_getcb(thisarg->bev, NULL, NULL, NULL, (void **)&othrarg);
        bufferevent_disable(bev, EV_READ);
        bufferevent_setwatermark(thisarg->bev, EV_WRITE, BUFSIZ_FOR_BEV / 2, 0);
        bufferevent_setcb(thisarg->bev, tcp_read_cb, tcp_write_cb, tcp_events_cb, othrarg);
    }
}

void tcp_write_cb(struct bufferevent *bev, void *arg) {
    TCPArg *thisarg = arg;
    bufferevent_enable(thisarg->bev, EV_READ);
    bufferevent_setwatermark(bev, EV_WRITE, 0, 0);
    bufferevent_setcb(bev, tcp_read_cb, NULL, tcp_events_cb, arg);
}

```

数据的交换其实只发生在 `tcp_read_cb` 之中，但是当 output 端的 bufferevent 中的数据没有被及时读出时，将会造成缓存的堆积，这时必须要禁止数据交换的继续进行。

因此我们可以看到当两个 bufferevent 中有一个 output buffer 的数据量超过 `BUFSIZ_FOR_BEV` 时，input buffer 的读取将会被禁止，同时设置 output buffer 的可写watermark, 只有当 output buffer 中的数据量少于 `BUFSIZ_FOR_BEV/2` 时才会调用 `tcp_write_cb`。

而在 `tcp_write_cb` 内部，则负责将 input buffer 的可写禁止解除，同时将可写的watermark限制也解除，对应的回调函数也进行解除，即可写时不需要调用回调函数。这样数据交换又可以继续进行。

综上所述，这两个函数共同构成一个很巧妙的I/O模型，能够实现流量的灵活控制，保证当数据流向一方阻塞时，数据不会持续堆积在buffer 中。

TCP连接的部分就暂时分析到这里，有问题的话可以在评论去留言。

#### UDP connections

同样从 `service` 函数开始看起，首先创建了一个socket `udplsock`. 这个socket绑定的是本地地址和本地监听的udp端口。说明udp与本地程序通信的方式都是通过socket管道进行数据交换。

接着创建了两个event, 一个是超时后释放udp相关资源 `udptev`，另一个是当 `udplsock` 可读时唤醒 `udp_request_cb`的  `udplev`。

接着创建与远程服务器进行通信的socket, 通过 `bufferevent_opnessl_socket_new` 函数建立安全的 ssl 连接，并且传输层是属于安全的tcp连接，前面已经提到了，不管是TCP还是UDP流量，tls-proxy都通过tcp传输出去。

##### `udp_sendreq_cb`

向服务器发送连接申请后，通过该函数进行处理。当连接成功后，发送初始的HTTP请求消息，只是将 "ConnectionType" 改为 "udp".

接着通过 `udp_recvres_cb` 接收服务器的返回消息。

##### `udp_recvres_cb`

用于接收第一次发送请求消息后的服务器返回消息。

当数据到达后，首先读取第一行的 statusline, 检查正常后，将上面提到的两个事件 `udptev` 和 `udplev` 加入事件队列中，加入后两个事件就开始正常工作后。

同时将 proxy-server 端的bufferevent 的可读回调函数设置为 `udp_response_cb`，当有事件到达时还是唤醒 `udp_sendreq_cb`，用于在出现相关错误事件时关闭udp并释放资源，毕竟tcp还得工作。

##### `udp_request_cb `

前面提到了 `udp_request_cb` 是当本地socket `udplsock` 可读时的回调函数，当其可读时，说明本地程序需要发送请求，tls-proxy则需要从socket管道接收数据，解析出目的地址，并将其发送往 bufferevent.

udp传输发送和接收数据的函数分别是 `sendto` 和 `recvmsg` ，并不是配套的两个函数，可能是为了方便。

`udp_request_cb` 使用 `recvmsg` 从socket管道接收数据，接收到的数据存储在 `struct msghdr` 结构体中。

```c
struct iovec {                    /* Scatter/gather array items */
   void  *iov_base;              /* Starting address */
   size_t iov_len;               /* Number of bytes to transfer */
};

struct msghdr {
   void         *msg_name;       /* optional address */
   socklen_t     msg_namelen;    /* size of address */
   struct iovec *msg_iov;        /* scatter/gather array */
   size_t        msg_iovlen;     /* # elements in msg_iov */
   void         *msg_control;    /* ancillary data, see below */
   size_t        msg_controllen; /* ancillary data buffer len */
   int           msg_flags;      /* flags on received message */
};
```

其中 `msg_control` 指向一些辅助数据，利用 `CMSG_FIRSTHDR` 宏可以找到目的地址，具体用法可以查看源代码。

接着作者将本地地址、目的地址以及远程端口都变成字符串形式发送给 `udpbev`，远程端口的获得则通过 uthash 实现的一个哈希表，key为本地地址和本地端口组成的 string, value 则为远程端口。

此外，作者还对报文内容进行了 base64 的加密，暂时不确定这么做的理由，应该是为了安全，避免明文发送报文。

注意，发送给 `udpbev` 的内容并非正式的udp报文，而是作者自己定义的格式，在 server 还会再做处理。

##### `udp_response_cb`

该函数用于在


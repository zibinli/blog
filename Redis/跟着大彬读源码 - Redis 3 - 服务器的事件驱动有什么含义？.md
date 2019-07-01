众所周知，Redis 服务器是一个事件驱动程序。那么事件驱动对于 Redis 而言有什么含义？源码中又是如何实现事件驱动的呢？今天，我们一起来认识下 Redis 服务器的事件驱动。

对于 Redis 而言，服务器需要处理以下两类事件：
- **文件事件（file event）**：Redis 服务器通过套接字与客户端进行连接，而文件事件就是服务器对套接字操作的抽象。服务器与客户端的通信会产生相应的文件事件，而服务器则通过监听并处理这些事件来完成一系列的网络通信操作。
- **时间时间（time event）**：Redis 服务器中的一些操作（比如 serverCron 函数）需要在给定的时间点执行，而时间事件就是服务器对这类定时操作的抽象。

接下来，我们分别来认识下文件事件和时间事件。

### 1 文件事件
Redis 基于 Reactor 模式开发了自己的网络事件处理器，这个处理器被称为**文件事件处理器（file event handler）：
- 文件事件处理器使用 IO 多路复用程序来同时监听多个套接字，并根据套接字目前执行的任务来为套接字关联不同的事件处理器。
- 当被监听的套接字准备好执行连接应答（accept）、读取（read）、写入（write）、关闭（close）等操作时，与操作相对应的文件事件就会产生，这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件。

虽然文件处理器以单线程方式运行，但通过 IO 多路复用程序监听多个套接字，既实现了高性能的网络通信模型，又可以很好的与 Redis 服务器中其它同样以单线程运行的模块进行对接，保持了 Redis 内部单线程设计的简洁。

#### 1.1 文件事件处理器的构成
图 1 展示了文件事件处理器的四个组成部分：
- 套接字；
- IO 多路复用程序；
- 文件事件分派器（dispatcher）；
- 事件处理器；

![文件事件处理器的四个组成部分](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190613124505742_9741.png)

文件事件是对套接字的抽象。每当一个套接字准备好执行连接应答（accept）、写入、读取、关闭等操作时，就好产生一个文件事件。因为一个服务器通常会连接多个套接字，所以多个文件事件有可能会并发的出现。

而 IO 多了复用程序负责监听多个套接字，并向文件事件分派器分发那些产生事件的套接字。

尽管多个文件事件可能会并发的出现，但 IO 多路复用程序总是会将所有产生事件的套接字都放到一个队列里面，然后通过这个队列，以**有序、同步**的方式，把每一个套接字传输给文件事件分派器。当上一个套接字产生的事件被处理完毕之后（即，该套接字为事件所关联的事件处理器执行完毕），IO 多路复用程序才会继续向文件事件分派器传送下一个套接字。如图 2 所示：

![图 2 - IO 多路复用程序通过队列向文件事件分派器传送套接字](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190613125101059_19351.png)

文件事件分派器接收 IO 多路复用程序传来的套接字，并根据套接字产生的事件类型，调用相应的事件处理器。

服务器会为执行不同任务的套接字关联不同的事件处理器。**这些处理器本质上就是一个个函数**。它们定义了某个事件发生时，服务器应该执行的动作。

#### 1.2 IO 多路复用程序的实现
Redis 的 IO 多路复用程序的所有功能都是通过包装常见的 **select、epoll、evport 和 kqueue** 这些 IO 多路复用函数库来实现的。每个 IO 多路复用函数库在 Redis 源码中都对应一个单独的文件，比如 ae_select.c、ae_poll.c、ae_kqueue.c 等。

由于 Redis 为每个 IO 多路复用函数库都实现了相同的 API，所以 IO 多路复用程序的底层实现是可以互换的，如图 3 所示：

![图 3 - Redis 的 IO 多路复用程序有多个 IO 多路复用库实现可选](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190613125627879_24040.png)

Redis 在 IO 多路复用程序的实现源码中用 ```#include``` 宏定义了相应的规则，**程序会在编译时自动选择系统中性能最高的 IO 多路复用函数库来作为 Redis 的 IO 多路复用程序的底层实现，这保证了 Redis 在各个平台的兼容性和高性能。对应源码如下：
```
/* Include the best multiplexing layer supported by this system.
 * The following should be ordered by performances, descending. */
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

#### 1.3 事件的类型
IO 多路复用程序可以监听多个套接字的 ```ae.h/AE_READABLE``` 和 ```ae.h/AE_WRITABLE``` 事件，这两类事件和套接字操作之间有以下对应关系：
- **当服务器套接字变得可读时，套接字会产生 AE_READABLE 事件**。此处的套接字可读，是指客户端对套接字执行 write、close 操作，或者有新的可应答（acceptable）套接字出现时（客户端对服务器的监听套接字执行 connect 操作），套接字会产生 AE_READABLE 事件。
- **当服务器套接字变得可写时，套接字会产生 AE_WRITABLE 事件。**

IO 多路复用程序允许服务器同时监听套接字的 AR_READABLE 事件和 AE_WRITABLE 事件。如果一个套接字同时产生了两个事件，那么文件分派器会优先处理 AE_READABLE 事件，然后再处理 AE_WRITABLE 事件。简单来说，如果一个套接字既可读又可写，那么服务器将先读套接字，后写套接字。

#### 1.4 文件事件处理器
Redis 为文件事件编写了多个处理器，这些事件处理器分别用于实现不同的网络通信需求。比如说：
- 为了对连接服务器的各个客户端进行应答，服务器要**为监听套接字关联连接应答处理器**。
- 为了接收客户端传了的命令请求，服务器要**为客户端套接字关联命令请求处理器**。
- 为了向客户端返回命令执行结果，服务器要**为客户端套接字关联命令回复处理器**。
- 当主服务器和从服务器进行复制操作时，**主从服务器都需要关联复制处理器**。

在这些事件处理器中，服务器最常用的是**与客户端进行通信的连接应答处理器、命令请求处理器和命令回复处理器**。

**1）连接应答处理器**

```networking.c/acceptTcpHandle``` 函数是 Redis 的连接应答处理器，这个处理器用于对连接服务器监听套接字的客户端进行应答，具体实现为 ```sys/socket.h/accept``` 函数的包装。

当 Redis 服务器进行初始化的时候，程序会将这个连接应答处理器和服务器监听套接字的 AE_READABLE 事件关联。对应源码如下
```
# server.c/initServer
...
/* Create an event handler for accepting new connections in TCP and Unix
 * domain sockets. */
for (j = 0; j < server.ipfd_count; j++) {
    if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
        acceptTcpHandler,NULL) == AE_ERR)
        {
            serverPanic(
                "Unrecoverable error creating server.ipfd file event.");
        }
}
...
```

当有客户端用 ```sys/scoket.h/connect``` 函数连接服务器监听套接字时，套接字就会产生 AE_READABLE 事件，引发连接应答处理器执行，并执行相应的套接字应答操作。如图 4 所示：

![图 4 - 服务器对客户端的连接请求进行应答](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190614123503033_20651.png)

**2）命令请求处理器**
```networking.c/readQueryFromClient``` 函数是 Redis 的命令请求处理器，这个处理器负责从套接字中读入客户端发送的命令请求内容，具体实现为 ```unistd.h/read``` 函数的包装。

当一个客户端通过连接应答处理器成功连接到服务器之后，服务器会将客户端套接字的 AE_READABLE 事件和命令请求处理器关联起来（```networking.c/acceptCommonHandler``` 函数）。

当客户端向服务器发送命令请求的时候，套接字就会产生 AR_READABLE 事件，引发命令请求处理器执行，并执行相应的套接字读入操作，如图 5 所示：

![图 5 - 服务器接收客户端发来的命令请求](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190614124918037_1589.png)

在客户端连接服务器的整个过程中，服务器都会一直为客户端套接字的 AE_READABLE 事件关联命令请求处理器。

**3）命令回复处理器**
```networking.c/sendReplToClient``` 函数是 Redis 的命令回复处理器，这个处理器负责将服务器执行命令后得到的命令回复通过套接字返回给客户端。

当服务器有命令回复需要发给客户端时，服务器会将客户端套接字的 AE_WRITABLE 事件和命令回复处理器关联（```networking.c/handleClientsWithPendingWrites``` 函数）。

当客户端准备好接收服务器传回的命令回复时，就会产生 AE_WRITABLE 事件，引发命令回复处理器执行，并执行相应的套接字写入操作。如图 6 所示：

![图 6 - 服务器向客户端发送命令回复](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190614125858054_14856.png)

当命令回复发送完毕之后，服务器就会解除命令回复处理器与客户端套接字的 AE_WRITABLE 事件的关联。对应源码如下：
```
# networking.c/writeToClient
...
if (!clientHasPendingReplies(c)) {
    c->sentlen = 0;
    # buffer 缓冲区命令回复已发送，删除套接字和事件的关联
    if (handler_installed) aeDeleteFileEvent(server.el,c->fd,AE_WRITABLE);

    /* Close connection after entire reply has been sent. */
    if (c->flags & CLIENT_CLOSE_AFTER_REPLY) {
        freeClient(c);
        return C_ERR;
    }
}
...
```

#### 1.5 客户端与服务器连接事件
之前我们通过 debug 的形式大致认识了客户端与服务器的连接过程。现在，我们站在文件事件的角度，再一次来追踪 Redis 客户端与服务器进行连接并发送命令的整个过程，看看在过程中会产生什么事件，这些事件又是如何被处理的。

先来看客户端与服务器建立连接的过程：
1. 先启动我们的 Redis 服务器（127.0.0.1-8379）。成功启动后，服务器套接字（127.0.0.1-8379） AE_READABLE 事件正处于被监听状态，而该事件对应连接应答处理器。（```server.c/initServer()```）。
2. 使用 redis-cli 连接服务器。这是，服务器套接字（127.0.0.1-8379）将产生 AR_READABLE 事件，触发连接应答处理器执行（```networking.c/acceptTcpHandler()```）。
3. 对客户端的连接请求进行应答，创建客户端套接字，保存客户端状态信息，并将客户端套接字的 AE_READABLE 事件与命令请求处理器（```networking.c/acceptCommonHandler()```）进行关联，使得服务器可以接收该客户端发来的命令请求。 

此时，客户端已成功与服务器建立连接了。上述过程，我们仍然可以用 gdb 调试，查看函数的执行过程。具体调试过程如下：
```
gdb ./src/redis-server
(gdb) b acceptCommonHandler    # 给 acceptCommonHandler 函数设置断点
(gdb) r redis-conf --port 8379 # 启动服务器
```

另外开一个窗口，使用 redis-cli 连接服务器：```redis-cli -p 8379```

回到服务器窗口，我们会看到已进入 gdb 调试模式，输入：```info stack```，可以看到如图 6 所示的堆栈信息。

![图 6 - gdb 调试显示客户端连接时服务器的堆栈信息](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190614133035004_12904.png)

现在，我们再来认识命令的执行过程：
1. 客户端向服务器发送一个命令请求，客户端套接字产生 AE_READABLE 事件，引发命令请求处理器（readQueryFromClient）执行，读取客户端的命令内容；
2. 根据客户端发送命令内容，格式化客户端 argc、argv 等相关值属性值；
3. 根据命令名称查找对应函数。```server.c/processCommad()``` 中 ```lookupCommand``` 函数调用；
4. 执行与命令名关联的函数，获得返回结果，客户端套接字产生 。```server.c/processCommad()``` 中 ```call``` 函数调用。
5. 返回命令回复，删除客户端套接字与 AE_WRITABLE 事件的关联。```network.c/writeToClient()``` 函数。

图 7 展示了命令执行过程的堆栈信息。图 8 则展示了命令回复过程的堆栈信息。
![图 7 - 服务器执行命令的堆栈信息](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190614180606647_15447.png)


![图 8 - 命令回复的堆栈信息](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190617195653689_22592.png)

### 2 时间事件
Redis 的时间时间分为以下两类：
- **定时时间**：让一段程序在指定的时间之后执行一次。比如，让程序 M 在当前时间的 60 毫秒后执行一次。
- **周期性事件**：让一段程序每隔指定时间就执行一次。比如，让程序 N 每隔 30 毫秒执行一次。

对于时间事件，数据结构源码（ae.h/aeTimeEvent）：
```
/* Time event structure */
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    aeTimeProc *timeProc;
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *next;
} aeTimeEvent;
```

主要属性说明：
- id：服务器为时间事件创建的全局唯一 ID。ID 号按从小到大的顺序递增。
- when_sec：秒精度的 UNIX 时间戳，记录了时间事件的到达时间。
- when_ms：毫秒精度的 UNIX 时间戳，记录了时间事件的到达时间。
- timeProc：时间事件处理器，对应一个函数。当时间事件发生时，服务器就会调用相应的处理器来处理事件。

时间事件进程执行的函数为 ```ae.c/processTimeEvents()```。

此外，对于时间事件的类型区分，取决于时间事件处理器的返回值：
- 返回值是 ``ae.h/AE_NOMORE``，为**定时事件**。该事件在到达一次后就会被删除；
- 返回值不是 ``ae.h/AE_NOMORE``，为**周期事件**。当一个周期时间事件到达后，服务器会根据事件处理器返回的值，对时间事件的 when_sec 和 when_ms 属性进行更新，让这个事件在一段时间之后再次到达，并以这种方式一致更新运行。比如，如果一个时间事件处理器返回 30，那么服务器应该对这个时间事件进行更新，让这个事件在 30 毫秒后再次执行。
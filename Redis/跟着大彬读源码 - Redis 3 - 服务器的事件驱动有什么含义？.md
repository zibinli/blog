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
- **当套接字变得可读时，套接字会产生 AE_READABLE 事件**。此处的套接字可读，是指客户端对套接字执行 write、close 操作，或者有新的可应答（acceptable）套接字出现时（客户端对服务器的监听套接字执行 connect 操作），套接字会产生 AE_READABLE 事件。
- **当套接字变得可写时，套接字会产生 AE_WRITABLE 事件。**


### 2 时间事件

上一节我们一起认识了文件事件。接下来，让我们再来认识下时间事件。

### 1 时间事件
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

#### 1.1 时间事件之 serverCron 函数
持续运行的 Redis 服务器需要定期对自身的资源和状态进行检查和调整，从而确保服务可以长期、稳定的运行。这些定期操作由 ```server.c/serverCron()``` 函数负责执行。主要操作包括：
- 更新服务器的各类统计信息。比如时间、内存占用、数据库占用情况等。
- 清理数据库中的过期键值对。
- 关闭和清理连接失效的客户端。
- 尝试进行 AOF 或 RDB 持久化操作。
- 如果服务器是主服务器，对从 服务器进行定期同步。
- 如果处于集群模式，对集群进行定期同步和连接测试。

Redis 服务器以周期性事件的方式来运行 serverCron 函数，在服务器运行期间，每隔一段时间，serverCron 就会执行一次，直到服务器关闭为止。

关于执行次数，可参见 ```redis.conf``` 文件中的 **hz** 选项。默认为 10，表示每秒运行 10 次。

### 2 事件调度与执行
由于服务器同时存在文件事件和时间事件，所以服务器必须对这两种事件进行调度，来决定何时处理文件事件，何时处理时间事件，以及花多少时间来处理它们等等。

事件的调度和执行有 ```ae.c/aeProcessEvents()``` 函数负责。源码如下：
```
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;
    /* Nothing to do? return ASAP */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    /* 首先判断是否存在需要监听的文件事件，如果存在需要监听的文件事件，那么通过IO多路复用程序获取
     * 准备就绪的文件事件，至于IO多路复用程序是否等待以及等待多久的时间，依发生时间距离现在最近的时间事件确定;
     * 如果eventLoop->maxfd == -1表示没有需要监听的文件事件，但是时间事件肯定是存在的(serverCron())，
     * 如果此时没有设置 AE_DONT_WAIT 标志位，此时调用IO多路复用，其目的不是为了监听文件事件是否准备就绪，
     * 而是为了使线程休眠到发生时间距离现在最近的时间事件的发生时间(作用类似于unix中的sleep函数),
     * 这种休眠操作的目的是为了避免线程一直不停的遍历时间事件形成的无序链表，造成不必要的资源浪费 */
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

        /* 寻找发生时间距离现在最近的时间事件,该时间事件的发生时间与当前时间之差就是IO多路复用程序应该等待的时间 */
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        if (shortest) {
            long now_sec, now_ms;

            // 创建 timeval 结构
            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;

            /* How many milliseconds we need to wait for the next
             * time event to fire? */
            long long ms =
                (shortest->when_sec - now_sec)*1000 +
                shortest->when_ms - now_ms;

            /* 如果时间之差大于0，说明时间事件到时时间未到,则等待对应的时间;
             * 如果时间间隔小于0，说明时间事件已经到时，此时如果没有
             * 文件事件准备就绪，那么IO多路复用程序应该立即返回，以免
             * 耽误处理时间事件*/
            if (ms > 0) {
                tvp->tv_sec = ms/1000;
                tvp->tv_usec = (ms % 1000)*1000;
            } else {
                tvp->tv_sec = 0;
                tvp->tv_usec = 0;
            }
        } else {
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }

        // 阻塞并等等文件事件产生，最大阻塞事件由 timeval 结构决定
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            // 处理所有已产生的文件事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0; /* Number of events fired for current fd. */
            int invert = fe->mask & AE_BARRIER;

            if (!invert && fe->mask & mask & AE_READABLE) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }

            /* Fire the writable event. */
            if (fe->mask & mask & AE_WRITABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            /* If we have to invert the call, fire the readable event now
             * after the writable one. */
            if (invert && fe->mask & mask & AE_READABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            processed++;
        }
    }
    /* Check time events */
    if (flags & AE_TIME_EVENTS)
        // 处理所有已到达的时间事件
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```

将 ```aeProcessEvents``` 函数置于一个循环里面，加上初始化和清理函数，就构成了 Redis 服务器的主函数 ```server.c/main()```。以下是主函数的伪代码：
```
def main():
    // 初始化服务器
    init_server();
    // 一直处理事件，直到服务器关闭为止
    while server_is_not_shutdown():
        aeProcessEvents();
    // 服务器关闭，执行清理操作
    clear_server()
```

从事件处理的角度来看，Redis 服务器的运行流程可以用流程图 1 来概括：

![图 1：事件处理角度下的服务器运行流程](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190706151711050_31836.png)

以下是事件的调度和执行规则：
1. aeApiPoll 函数的最大阻塞事件由到达时间最接近当前时间的时间事件决定。这个方法既可以避免服务器对时间事件进行频繁的轮询，也可以确保 aeApiPoll 函数不会阻塞过长时间。
2. 因为文件事件是随机出现的，如果等待并处理完一次文件事件之后，仍未有任何时间事件到达，那么服务器将再次等待并处理文件事件。随着文件事件的不断执行，时间会逐渐向时间事件所设置的到达时间逼近，并最终来到，这时服务器就可以开始处理到达的时间事件了。
3. 对文件事件和时间事件的处理都是**同步、有序、原子**地执行。服务器不会中途中断事件处理，也不会对事件进行抢占。因此，不管是文件事件的处理器，还是时间事件的处理器，它们斗殴尽可能的减少程序的阻塞事件，并在有需要时主动让出执行权，从而降低事件饥饿的可能性。举个栗子，在命令回复处理器将一个命令回复写入到客户端套接字时，如果写入字节数超过了一个预设常量，命令回复处理器就会主动用 break 跳出写入循环，将余下的数据留到下次再写。另外，时间事件也会将非常耗时的持久化操作放到子线程或者子进程中执行。
4. 因为时间事件在文件事件之后执行，并且事件之间不会出现抢占，所以时间事件的实际处理时间，通常会比时间事件设定的时间稍晚一些。

### 总结
1. 时间事件分为定时事件和周期事件。定时事件只在指定时间执行一次，而周期事件则每隔指定时间执行一次。
2. 服务器一般情况下只执行 serverCron 函数这一个周期性时间事件。
3. 时间事件和文件事件之间是合作关系。服务器会轮流处理这两种事件，并且处理事件的过程中不会进行抢占。
4. 时间事件的实际处理时间通常比设定的时间晚一些。
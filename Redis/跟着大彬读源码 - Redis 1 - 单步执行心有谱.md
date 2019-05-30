一直很羡慕那些能读 Redis 源码的童鞋，也一直想自己解读一遍，但迫于 C 大魔王的压力，解读日期一拖再拖。

但正所谓，种一棵树的最好时间是十年前，其次就是现在。择日不如撞日，就让我们从现在开始解读 Redis 源码吧。

决定要读了，下一步就是如何读。从 github 上克隆下来源码，一看 src 目录，望天，104 个文件，我该从哪个文件开始呢？一个个文件看？不行不行，这样对我毫无诱惑力，没有诱惑力，怎么能战胜游戏、小说对我的吸引呢？苦苦思考，不得其解。然后突然想起来 HTTP 协议的那个经典面试题：从浏览器输入网址，到页面展示，这个过程发生了什么？

把这个面试题换成 Redis：输入开启 Redis 服务的命令，回车，到启动 Redis 服务，这个过程发生了什么？

很好，这个问题成功吸引到我了。就让我们从源码中找出这个问题的答案吧。

要了解 Redis 命令的执行过程，首先要安装 Redis 服务，搭建 debug 环境。如果我们能一行行的看到命令在代码中的执行过程，解读源码也就没任何阻碍了。

后续所有文章均基于 redis3.2.13 版本。

### 1 搭建 debug 环境
**1、下载编译文件**
在 linux 上，下载源码文件，编译，使用 gdb（cgdb） 进行 debug。
```
# bash
wget https://github.com/antirez/redis/archive/3.2.13.tar.gz
tar -zxvf 3.2.13.tar.gz
mv redis-3.2.13 /opt/
cd redis-3.2.13
make                 # 编译文件，得到可执行文件 redis-server、redis-cli 等
```

**2、开启 debug**
```
# bash
gdb src/redis-server # 在 redis 安装目录，进入 gdb 调试环境
```

按我们平时调试的习惯，找到一个函数设置断点，然后一步步运行调试。对于 Redis 也一样，我们找到 server.c 文件，服务器运行的 main 函数就在此文件中。我们对 main 函数设置断点：
``` 
# gdb
(gdb) b main
Breakpoint 1 at 0x42ed05: file server.c, line 3962.
```

页面会提示我们在 server.c 文件的 3962 行设置了断点，也就是我们指定的 main 函数的位置。

设置好断点，下一步就是启动服务：
```
# 启动服务
(gdb) r ./redis.conf
Starting program: /opt/redis-3.2.13/src/redis-server ./redis.conf
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, main (argc=2, argv=0x7fffffffe5a8) at server.c:3962
3962	int main(int argc, char **argv) {
```

通过页面输出信息，我们会发现程序已经运行到我们设置的断点了。但是我们看不到运行处的代码，这可不行，看不到源码的调试，没法接受使用以下命令”召唤“源码：
```
(gdb) layout src
```

出现下图所示的界面：
![图 1 - gdb 的 src 和 cmd 并存](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190530130922292_10690.png)

到了这一步，我们已经正式开始踏上 Redis 源码解读之路了。

### 2 初始化服务
继续往下走，使用  ```n``` 命令，执行下一步，然后不断回车、回车、回车，好像每一行都看不懂什么意思。不管了，继续走。咦，好像发现个能看懂的 ```initServerConfig()```。没看错的话，这个应该是初始化服务器配置的，让我们进到这个函数里确认下：
```
(gdb) s
```

回车，走你。然后我们就看到了下面这个界面：
![图 2 - 进入初始化服务器配置函数](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190530131828651_24415.png)

提示我们进入了 server.c 1464 行的 ```initServerConfig``` 函数中。 ```n``` 命令，继续走。我们会发现在这个函数里对服务器的各种基础参数进行初始化。这里的参数详见 server.h/redisServer 结构体。

回到 main 函数后，我们继续前进，还会发现一个 ```initServer()``` 的函数。这个函数是进行驱动事件的注册，以及绑定回调函数等。

继续走，直到执行 ```aeMain()```，如下图：
![图 3 - Redis 服务已开启](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190530192740390_18685.png)

程序执行到 4133 行时，Redis 服务已成功开启了。此时服务器处于休眠状态，并使用 ```aeMain()``` 进行事件轮询，等待监听事件的发生。

### 3 gdb 基础使用
|   命令   |                                          解释                                           |         示例         |
| :------: | :------------------------------------------------------------------------------------: | :------------------: |
| gdb file |                                加载被调试的可执行程序文件                                 | gdb src/redis-server |
|    r     |                              Run 的缩写，运行被调试的程序。                               |    r ./redis.conf    |
|    c     |                Continue 的缩写。继续执行被调试程序，直至下一个断点或程序结束                 |          c           |
|    b     |         Breakpoint 缩写。设置断点。可以使用 行号、函数名称、执行地址等方式指定断点位置         |        b main        |
|   s/n    | s 相当于“单步跟踪并进入”，也就是说进入到执行的函数内部。n 相当于“单步跟踪”，不进入到执行函数内部 |         s/n          |
 | p 变量名称| Print 缩写。显示指定变量的值。| p server|

### 总结
1. 搭建环境三步走：下载、编译、gdb。
2. 服务启动包括：初始化基础配置、监听事件等。
3. gdb 基础命令：r c b n p。

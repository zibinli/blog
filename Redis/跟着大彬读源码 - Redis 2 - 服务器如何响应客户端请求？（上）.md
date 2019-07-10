上次我们通过问题“启动服务器，程序都干了什么？”，跟着源码，深入了解了 Redis 服务器的启动过程。

既然启动了 Redis 服务器，那我们就要连上 Redis 服务干些事情。这里我们可以通过 redis-cli 测试。现在客户端和服务器都准备好了，那么**Redis 客户端和服务器如何建立连接？服务器又是如何响应客户端的请求呢？**

### 1 连接服务器
客户端和服务器进行通讯，首先应该就是建立连接。接下来，我们来看下 redis-cli 与服务器的连接过程。

还记得我们上次使用 ```gdb``` 调试程序的步骤吗？让我们对 redis-cli 再来一次，看看源码的执行步骤。在开始之前，记得在编辑器打开 ```redis-cli.c```，定位到 ```main``` 函数的位置，毕竟 gdb 看代码没有编辑器看着舒服。

debug 步骤如下：
```
# bash
cd /opt/redis-3.2.13
// 启动 Redis 服务。Ctrl+c 可推出服务器启动页，同时保持服务器运行
./src/redis-server --port 8379 &
// 调试 redis-clli
gdb ./src/redis-cli
# gdb 
(gdb) b main
(gdb) r -p 8379
(gdb) layout src
(gdb) focus cmd
```

执行完上述步骤，我们会进入如下界面：

![图 1 - 进入 redis-cli main 函数](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190604195940724_8956.png)

这时候我们就可以回到编辑器页，看看对 ```main``` 函数中哪一行比较感兴趣，就停下来研究研究。到了 2618 行，我们会看到有执行 ```parseOptions``` 这个函数，看名字，好像是初始化一些可选项。那就进去看看呗。

#### 1.1 初始化客户端配置
函数执行步骤：```main``` -> ```parseOptions``` -> ```main```。

我们会看到，在执行 ```redis-cli``` 时携带的参数都是在这个函数中解析，比如我们启动的时候带着的 ```-p``` 参数，会在 996 行被解析到，同时赋值给客户端的 hostport 配置项。如下图：

![图 2 - 启动 redis-cli 携带的 -p 参数被赋值给 hostport 配置项](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190604200701003_6835.png)

#### 1.2 客户端启动模式
函数执行步骤：```main```。

回到 ```main``` 函数，会看到后面的代码会出现很多 ```cliConnect``` 函数。要注意的是，这里并不表示 redis-cli 会执行多次 ```cliConnect``` 函数。实际上，每一个 ```if``` 语句块，都代表着客户端的一种连接模式，3.2.13 版本支持以下模式：
1. Latency mode：延迟模式。```redis-cli --latency -p 8379``` 用来测试客户端与服务器连接的延迟。还有 ```--history``` 和 ```--dist``` 可选项，用来展示不同的形式。
2. Slave mode：模拟从节点模式。
3. Get RDB mode：生成并发送 RDB 持久化文件，保存在本地。
4. Pipe mode：管道模式。将命令封装成指定数据格式，批量发送给 redis 服务器执行。
5. Find big keys：统计 bigkey 的分布。
6. Stat mode：统计模式。实时展示服务器的统计信息。
7. Scan mode：扫描指定模式的键，相当于 scan 模式。
8. LRU test mode：实时测试 LRU 算法的命中情况。

#### 1.3 连接服务器
函数执行步骤：```main``` -> ```cliConnect``` -> ```redisConnect``` -> ```redisContextInit``` -> ```redisContextConnectTcp``` -> ```_redisContextConnectTcp``` ->  ```cliConnect```。

我们上面没有使用特殊模式启动，因此，我们会看到在 2687 行真正的去调用 ```cliConnect``` 函数。跟踪进去，让我们看看究竟是如何和服务器进行连接的。

在  ```cliConnect``` 函数中，我们看到，根据 ```hostsocket``` 的配置项，会使用不同的连接模式。从名字上，我们大概可以猜出，一个是 TCP Socket 连接，另一个是本机 Unix Socket 连接。

如果想要使用 Unix Socket 连接，只需按格式配置 ```hostscoket``` 即可：```./src/redis-cli -s /tmp/redis.sock```。

我们这里使用 TCP Scoket 连接，使用 ```redisConnect``` 函数建立连接。

不断追踪，我们会看到上面所示的函数执行步骤，在 ```_redisContextConnectTcp``` 函数中会看到 ```getaddrinfo``` 和 ```connect``` 函数的调用，这里就是建立 TCP 连接的地方。

#### 1.4 校验权限及选择数据库
函数执行步骤：```cliConnect``` -> ```anetKeepAlive``` -> ```cliAuth``` -> ```cliSelect``` -> ```main```。

回到  ```cliConnect``` 函数，如果正常连接上服务器后，还会将我们上面创建的 TCP 连接设置为长连接，然后校验权限，选择连接数据库。
```
...
/* Set aggressive KEEP_ALIVE socket option in the Redis context socket
 * in order to prevent timeouts caused by the execution of long
 * commands. At the same time this improves the detection of real
 * errors. */
anetKeepAlive(NULL, context->fd, REDIS_CLI_KEEPALIVE_INTERVAL);
/* Do AUTH and select the right DB. */
if (cliAuth() != REDIS_OK)
    return REDIS_ERR;
if (cliSelect() != REDIS_OK)
    return REDIS_ERR;
...
```

至此，我们已经跑完客户端与服务器建立连接的全过程。感兴趣的小伙伴可以尝试连接不存在的 IP 或 端口，观察程序抛出异常的时机，熟悉整个连接过程。

客户端与 服务器建立连接后，就可以使用相关命令操作数据库中的 key 了。下面我们以 ```SET KEY VALUE``` 命令为例，来看看命令的执行过程。

### 2 发送命令请求
当用户在客户端键入一个命令请求时，客户端会将这个命令请求按协议格式转换，然后通过连接到服务器的套接字，将转换后的命令请求发送给服务器，如图 3 所示：

![图 3 - 客户端接收并发送命令请求的过程](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190605200620062_13527.png)

因此，对于我们上面的命令请求，客户端会转成：
```
"*3\r\n$3\r\nSET\r\n$3\r\nKEY\r\n$5\r\nVALUE\r\n"
```

然后发给服务器。

以上是客户端发送命令给服务器的过程，在下一节中，我们再来认识服务器是如何响应客户端请的。

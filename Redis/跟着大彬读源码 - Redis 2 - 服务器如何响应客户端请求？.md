上次我们通过问题“启动服务器，程序都干了什么？”，跟着源码，深入了解了 Redis 服务器的启动过程。

既然启动了 Redis 服务器，那我们就要连上 Redis 服务干些事情。这里我们可以通过 redis-cli 测试。现在客户端和服务器都准备好了，那么**怎么 Redis 客户端和服务器如何建立连接？服务器又是如何响应客户端的请求呢？**

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

### 3 服务器处理
服务器读取到命令请求后，会进行一系列的处理。

#### 3.1 读取命令请求
当客户端与服务器之间的套接字因客户端的写入变得可读时，服务器将调用命令请求处理器执行以下操作：
1. 读取套接字中的命令请求，并将其保存到客户端状态的输入缓冲区。
2. 对输入缓冲区的命令请求进行分析，提取出命令请求中包含的命令参数及参数个数，然后分别将参数和参数个数保存到客户端状态的 argv 属性和 argc 属性里。
3. 调用命令执行器，执行客户端指定的命令。

上面的 ```SET``` 命令保存到客户端状态的输入缓存区之后，客户端状态如图 4。

![图 4 - 客户端状态中的命令请求](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190605220440144_29575.png)

之后，分析程序将对输入缓冲区中的协议进行分析，并将得出的结果保存的客户端的 argv 和 argc 属性中，如图 5 所示：

![图 5 - 客户端状态中的 argv 和 argc 属性](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190605220637329_1161.png)

之后，服务器将通过调用**命令执行器**来完成执行命令的余下步骤。

#### 3.2 查找命令实现
命令执行器要做的第一件事就是根据 argv[0] 参数，在命令表（commandtable）中查找参数所指定的命令，并将找到的命令保存到 cmd 属性中。

命令表是一个字典，字典的键是一个个命令名称，比如 "SET"、"GET" 等。而字典的值则是一个个 redisCommand 结构，每个 redisCommand 结构记录了 Redis 命令的实现信息。源码如下：
```
# server.h/redisCommand
struct redisCommand {
    char *name;   // 命令名称。如 "SET"
    redisCommandProc *proc; // 对应函数指针，指向命令的实现函数。比如 SET 对应的 setCommand 函数
    int arity;    // 命令参数的格个数。用来检查命令请求的格式是否合法。
                        // 要注意的命令的名称也是一个参数。像我们上面的 SET KEY VALUE 命令，实际上有三个参数。
    char *sflags; // 字符串形式的标识值。记录了命令的属性。
    int flags;    // 对 sflags 标识分析得出的二进制标识，由程序自动生成。检查命令时，实际上使用的是此字段
    redisGetKeysProc *getkeys_proc; // 指针函数，通过此方法来指定 key 的位置。
    int firstkey; // 第一个 key 的位置
    int lastkey;  // 最后一个 key 的位置
    int keystep;  // key 之间的间距
    long long microseconds, calls; // 命令的总调用时间及调用次数
};
```

另外，对于 sflags 属性，可使用的标识值及含义如下表：
| 标识     |  意义   |  带有此标识的命令   |
| :-: | :-: | :-: |
|   w  |  这是一个写入命令，可能会修改数据库   |  SET、RPUSH、DEL 等   |
|  r   |  这是一个只读命令，不会修改数据库   | GET、STRLEN 等    |
|  m   | 此命令可能会占用大量内存，执行器需先检查内存使用情况，如果内存紧缺就禁止执行此命令    | SET、APPEND、RPUSH、SADD 等    |
|   a  |  这是一个管理命令   | SAVE、BGSAVE 等    |
|   p | 这是一个发布与订阅功能的命令    |  PUBLISH、SUBSRIBE 等   |
|   f |     |     |
|    s |  这个命令不可以在 lua 脚步中使用   |   BPOP、BLPOP 等  |
|   R  |   这是一个随机命令。对于相同的数据集和相同的参数，返回结果可能不同  | SPOP、SRANDMEMBER 等    |
|   S  | 当在 lua 脚步中使用此命令时，对返回结果进行排序，使得结果有序    | SINTER、SUNION 等    |
|   l  |  这个命令可以在服务器载入数据的过程中使用   |INFO、PUBLISH 等     |
|   t  |  这个命令允许在从库有过期数据时使用   | SLAVEOF、PING 等    |
|   M  | 这个命令在监视模式下，不会被自动传播    |   EXEC  |
|   k  | 集群模式下，如果对应槽点标记位“导入”，则接受此命令 |   restore-asking  |
|   F  | 这个命令在程序执行时应该立刻执行    | SETNX、GET 等    |

命令表结构如图 6：

![图 6 - 命令表](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190606123755943_21138.png)

对于我们上面的 ```SET KEY VALUE``` 命令，当程序以图 5 中的 argv[0] 作为输入，在命令表中进行查找时，命令表返回 "set" 键对于的 redisCommand 结构，客户端状态的 cmd 指针会指向这个 redisCommand 结构。如图 7 所示：

![图 7 - 设置客户端状态的 cmd 指针](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190606124028494_23495.png)

要注意的是，对于 Redis 而言，**命令名字的大小写不影响命令表的查找结果**，也就是命令名称不区分大小写。执行 SET 和 set、Set 将获得相同结果。

#### 3.3 执行预备操作
到目前为止，服务器已经将执行命令所需要的命令实现函数（客户端 cmd 属性）、参数（客户端 argv 属性）、参数个数（客户端 argc 属性）都初始化完毕。但在真正执行命令之前，程序还会进行一些预备操作，保证命令可以正确、顺利的被执行。预备操作包括：
1. 检查客户端的 cmd 指针是否指向 NULL，如果是的话，说明用户输入的命令名称没有对应的函数，服务器将不再执行后续操作，并向客户端返回一个错误。
2. 根据客户端 cmd 属性指向的 redisCommand 结果的 arity 属性，检查命令请求所给定的**参数个数**是否正确。
3. 检查客户端是否已经通过了身份验证。未通过身份验证的客户端只能执行 ```AUTH``` 命令。否则，将会向客户端返回一个错误。
4. 如果服务器打开了 maxmemory 功能，在执行命令之前，会先检查服务器的内存占用情况，并在有需要时进行内存回收，从而使得接下来的命令可以顺利执行。如果内存回收失败，将不再执行后续步骤，向客户端返回一个错误。
5. 如果服务器上一次执行 ```BGSAVE``` 命令时出错，并且服务器打开了 *stop-writes-on-bgsave-error* 功能，而将要执行的命令是一个写命令，那么服务器将拒绝执行这个鞋命令，并向客户端返回一个错误。
6. 如果客户端正在用 ```SUBSCRIBE``` 和 ```PSUBSCRIBE``` 命令订阅频道或模式，那么服务器只会执行客户端发来的 ```SUBSCRIBE```、```PSUBSCRIBE```、```UNSUBSCRIBE```、```PUNSUBSCRIBE``` 四个命令，其它命令都会被拒绝。
7. 如果服务器正在进行数据载入，那么客户端发送是命令必须带有 ```l``` 标识才会被服务器执行。
8. 如果客户端正在执行事务，那么服务器只会执行 ```EXEC```、```DISCARD```、```MULTI```、```WATCH``` 四个命令，其他命令都会被放进事务队列中。
9. 如果服务器打开了监视器功能，那么服务器会将要执行的命令和参数等信息发送给监视器。

当完成了以上预备操作之后，服务器就开始真正的执行命令了。

要注意的是，上面列出的预备操作只是服务器在单机模式下的检查操作。如果在复制或者集群模式下，预备操作还会更多。

#### 3.4 调用命令的实现函数
在前面的操作中 ，服务器已经将要执行的命令实现、参数、参数个数保存在客户端结构中。

对于我们上面的 ```SET KEY VALUE``` 命令，图 8 包含了命令实现、参数和参数个数结构：

![图 8 - 客户端状态](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190606130340397_25001.png)


当服务器决定要执行命令时，只要执行以下语句即可：
```
// client 是指向客户端状态的指针。server.c/call()
client->cmd->proc(client);
```

上面的执行语句实际上就是调用 ```setCommand``` 函数（t_string.c）。

被调用的命令实现函数会执行指定的操作，并产生相应的命令回复，这些回复会被保存在客户端状态的输出缓冲区中（bug 属性 和 reply 属性），之后实现函数会为客户端的套接字关联命令回复处理器，由命令回复处理器返回给客户端。

回到我们的示例，```setCommand(client)``` 将产生一个 "+OK\r\n" 回复，这个回复被保存在客户端的 buf 属性中。如图 9 所示：

![图 9 - 保存了命令回复的客户端状态](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190606130904312_21491.png)


#### 3.5 执行后续工作
实现函数执行完后，服务器还会执行一些后续工作，主要包括：
1. 如果服务器开启了 slow-log 功能，那么慢查询日志模块将会检查是否需要将刚执行的命令添加到慢查询日志。
2. 更新 ```redisCommand``` 结构的 milliseconds 和 calls 属性。
3. 如果服务器开启了 AOF 持久化功能，那么 AOF 持久化模块会将刚刚执行的命令请求写入到 AOF 缓冲区中。
4. 如果有其它服务器正在复制当前这个服务器，那么服务器将会把刚刚执行的命令传播给所有从服务器。

以上后续操作执行完毕后，一条执行命令也就执行完成了。服务器可以继续处理后续的命令。

#### 3.6 将命令回复发送给客户端
上面过程中，命令实现函数会将命令回复保存到客户端的输出缓冲区中，并为客户端的套接字关联命令回复处理器。当客户端套接字变为可写状态时，服务器就会执行命令回复处理器，将命令回复发送给客户端。

当命令回复发送完毕后，回复处理器会情况客户端的输出缓冲区，为处理下一个命令请求做好准备。

以图 9 所示的客户端状态为例，当客户端的套接字变为可写状态时，命令回复处理器会将协议格式的命令回复 "+OK\r\n" 发送给客户端。

### 4 客户端接收并打印回复
客户端接收到命令回复之后，会将回复转换成我们可读的格式，并打印在屏幕上（对于 redis-cli 客户端），如图 10 所示。

![图 10 客户端接收并打印命令回复的过程](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190612080448952_13253.png)


至此，我们走完了从发起一个命令请求，到收到回复的所有过程。对于我们最开始提的问题，服务器如何响应客户端请求，你有答案了吗？

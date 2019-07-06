Redis 没有直接使用 C 语言传统的字符串表示（以空字符串结尾的字符数组），而是构建了一种名为**简单动态字符串（simple dynamic string）**的抽象类型，并将 SDS 用作 Redis 的默认字符串表示。

在 Redis 中，C 字符串只会作为字符串字面量用在一些无需对字符串进行修改的地方，比如打印日志：
> serverLog(LL_WARNING,"SIGTERM received but errors trying to shut down the server, check the logs for more information");

当 Redis 需要的不仅仅是一个字符串字面量，而是一个可以被修改的字符串值时，Redis 就会适应 SDS 来表示字符串。比如在数据库中，包含字符串值的键值对在底层都是由 SDS 实现的。

还是拿简单的 SET 命令举例，执行以下命令
```
redis> SET msg "hello world"
ok
```

那么，Redis 将在数据中创建一个新的键值对，其中：
- 键值对的键是一个字符串对着，对象的底层实现是一个保存着字符串 "msg" 的 SDS。
- 键值对的值也是一个字符串对象，对象的底层实现是一个保存着字符串 "hello world" 的 SDS。

除了用来保存数据库中的字符串值之外， SDS 还被用作缓冲区。AOF 模块中的 AOF 缓冲区，以及客户端状态中的输入缓冲区，都是由 SDS 实现的。

接下来，我们就来详细认识下 SDS。

### 1 SDS 的定义
在 sds.h 中，我们会看到以下结构：
```
typedef char *sds;
```

可以看到，SDS 等同于 char * 类型。这是因为 SDS 需要和传统的 C 字符串保存兼容，因此将其类型设置为 char *。但是要注意的是，SDS 并不等同 char *，它还包括一个 header 结构，源码如下：
```
struct __attribute__ ((__packed__)) sdshdr5 { // 已弃用
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 { // 长度小于 2^8 的字符串类型
    uint8_t len;         // SDS 所保存的字符串长度
    uint8_t alloc;       // SDS 分配的长度
    unsigned char flags; // 标记位，占 1 字节，使用低 3 位存储 SDS 的 type，高 5 位不使用
    char buf[];          // 存储的真实字符串数据
};
struct __attribute__ ((__packed__)) sdshdr16 { // 长度小于 2^16 的字符串类型
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 { // 长度小于 2^32 的字符串类型
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 { // 长度小于 2^64 的字符串类型
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

#### 1.1 源码解读
上面都是理论知识，SDS 在数据库中究竟是怎么存储的？让我们继续用 gdb 一探究竟 SDS 的数据结构。

对于字符串的操作，基本上所有操作方法都在 ```t_string.c``` 文件中。

根据 ```server.c/redisCommandTable```，我们可以查到 SET 命令对应的函数是 ```setCommand```。因此，我们可以在这个方法上设置断点，启动服务，进行 gdb 调试，步骤如下：
```
gbd src/redis-server
(gdb) layout src   # 显示源码窗口
(gdb) focus cmd    # 焦点指向命令行窗口
(gdb) b setCommand # 设置断点
(gdb) r redis.conf --port 8379 # 指定配置文件与端口号，启动服务
```

再用另一个窗口，使用 redis-cli 连接服务器，并执行 SET 命令：
```
redis-cli -p 8379
redis> SET msg 'hello world'
```

此时会看到客户端处于等待回复状态。切回服务器窗口，我们会看到如下画面：

![图 1-1：服务器捕获断点](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190704194510466_11791.png)

为了便于查看我们执行的 SET 命令中，key 和 value 的结构，我们进入到 ```setGenericCommand()``` 函数，如下图：

![图 1-2：进入setGenericCommand函数查看key和value的结构](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190704194811500_13534.png)

调用此方法中的参数，key 和 value 就是我们执行 SET 命令的参数。找到参数了，

之所以会有 5 种类型的 header，是为了能让不同长度的字符串使用对应大小的 header，提高内存利用率。

一个 SDS 的完整结构，由内存地址上前后相邻的两部分组成：
- header：包括字符串的长度（len），最大容量（alloc）和 flags（不包含 sdshdr5）。
- buf[]：一个字符串数组。这个数组的长度等于最大容量加 1，存储着真正的字符串数据。

### SDS VS C 原生字符串

### SDS 主要 API

### 总结

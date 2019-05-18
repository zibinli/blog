这个月开始学习 Redis 源码。基于 Redis 3.2 版本。

Redis 没有直接使用 C 语言传统的字符串表示（以空字符串结尾的字符数组），而是构建了一种名为**简单动态字符串（simple dynamic string）**的抽象类型，并将 SDS 用作 Redis 的默认字符串表示。

### SDS 的定义
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

之所以会有 5 种类型的 header，是为了能让不同长度的字符串使用对应大小的 header，提高内存利用率。

一个 SDS 的完整结构，由内存地址上前后相邻的两部分组成：
- header：包括字符串的长度（len），最大容量（alloc）和 flags（不包含 sdshdr5）。
- buf[]：一个字符串数组。这个数组的长度等于最大容量加 1，存储着真正的字符串数据。

### SDS VS C 原生字符串

### SDS 主要 API

### 总结
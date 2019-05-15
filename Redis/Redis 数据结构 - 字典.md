字典，是一种用于保存键值对的抽象数据结构。由于 C 语言没有内置字典这种数据结构，因此 Redis 构建了自己的字典实现。

在 Redis 中，就是使用字典来实现数据库底层的。对数据库的 CURD 操作也是构建在对字典的操作之上。

除了用来表示数据库之外，字典还是哈希键的底层实现之一。当一个哈希键包含的键值对比较多，又或者键值对中的元素都是比较长的字符串时，Redis 就会适应字典作为哈希键的底层实现。

### 1 字典的实现
Redis 的字典使用哈希表作为底层实现。一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。

#### 1.1 哈希表
Redis 字典所使用的哈希表结构：
```
typedef struct dictht {
    dictEntry **table;      // 哈希表数组
    unsigned long size;     // 哈希表大小
    unsigned long sizemask; // 哈希表大小掩码，用来计算索引
    unsigned long used;     // 哈希表现有节点的数量
} dictht;
```

- table 属性是一个数组。数组中的每个元素都是一个指向 dictEntry 结构的指针，每个 dictEntry 结构保存着一个键值对。
- size 属性记录了哈希表的大小，也即是 table 数组的大小。
- used 属性记录了哈希表目前已有节点（键值对）的数量。
- sizemask 属性的值总数等于 size-1，这个属性和哈希值一起决定一个键应该被放到 table 数组中哪个索引上。

图1 展示了一个大小为 4 的空哈希表。
![大小为4的空哈希表](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190515203213897_23208.png)

#### 1.2 哈希表节点
哈希表节点使用 dictEntry 结构表示，每个 dictEntry 结构中都保存着一个键值对：
```
typedef struct dictEntry {
    void *key;              // 键
    union {
        void *val;          // 值类型之指针
        uint64_t u64;       // 值类型之无符号整型
        int64_t s64;        // 值类型之有符号整型
        double d;           // 值类型之浮点型
    } v;                    // 值
    struct dictEntry *next; // 指向下个哈希表节点，形成链表
} dictEntry;
```
- key 属性保存着键，而 v 属性则保存着值。
- next 属性是指向另一个哈希表节点的指针。这个指针可以将多个哈希值相同的键值对连接在一起，以此来解决键冲突的问题。

图 2 展示了通过 next 指针，将两个索引相同的键 k1 和 k0 连接在一起的情况。
![连接在一起的键 k1 和 k0](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190515205808155_12978.png)

#### 1.3 字典
字典的结构：
```
typedef struct dict {
    dictType *type; // 类型特定函数
    void *privdata; // 私有数据
    dictht ht[2];   // 哈希表(两个)
    long rehashidx; // 记录 rehash 进度的标志。值为 -1 表示 rehash 未进行
    int iterators;  // 当前正在迭代的迭代器数
} dict;
```

dictType 的结构如下：
```
typedef struct dictType {
    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);
    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);
    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);
    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

type 属性和 privdata 属性是针对不同类型的键值对，为创建多态字典而设置的。其中：
- type 属性是一个指向 dictType 结构的指针，每个 dictType 结构保存了一簇用于操作特定类型键值对的函数。Redis 会为用途不用的字典设置不同的类型特定函数。
- privdata 属性保存了需要传给那些类型特定函数的可选参数。

而  ht 属性是一个包含两个哈希表的数组。一般情况下，字典




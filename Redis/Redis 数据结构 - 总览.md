相信很多人应该都知道 Redis 有五种数据类型：字符串、列表、哈希、集合和有序集合。但这五种数据类型是什么含义？Redis 的数据又是怎样存储的？今天我们一起来认识下 Redis 这五种数据结构的含义及其底层实现。

首先要明确的是，Redis 并没有直接使用这五种数据结构来实现键值对数据库，而是基于这些数据结构创建了一套对象系统，我们常说的数据类型，准确来说，就是 Redis 对象系统的类型。因此，我们认识 Redis 的数据结构，就是认识 Redis 的对象系统。

### 对象结构
对于 Redis 而言，所有键值对的存储，都是将数据存储在对象结构中。所不同的是，**键总是一个字符串对象，值可以是任意类型的对象**。
对象源码结构如下：
```
typedef struct redisObject {
    unsigned type:4;       // 对象类型
    unsigned encoding:4;   // 对象编码
    unsigned lru:LRU_BITS; // LRU
    int refcount;          // 引用统计
    void *ptr;             // 指向底层实现数据结构的指针
} robj;
```

- type 字段：对象类型，就是我们常说的。string、list、hash、set、zset。
- encoding：对象编码。也就是我们上面说的底层数据结构。
- LRU：键值对的 LRU。
- refcount：键值对对象的引用统计。当此值为 0 时，回收对象。
- *ptr：指向底层实现数据结构的指针。就是实际存放数据的地址。

#### 对象类型
对象有五种数据类型，就是我们上面提过的：
1. 字符串类型
2. 列表类型
3. 哈希类型
4. 集合类型
5. 有序集合类型

结合我们上面提到的键值对存储类型的差别，可以了解到，我们常说的“一个列表键或一个哈希键”，本质上指的是：**一个 key 对应的 value 是列表对象或哈希对象**。

对于 type 字段，我们可以使用 ```TYPE``` 命令来查看指定 key 对应 value 值的对象类型。
![图 2 - 设置不同类型的 key](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190522130037441_6914.png)
![图 3 - TYPE 命令对不同类型的 key 的输出](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190522130140504_20872.png)

#### 对象编码
按道理讲，已经有了 type，为什么还要搞个编码呢？

想想看，通过 encoding 属性，我们是不是使用不同编码的对象？这种使用方式可以根据不同的使用场景来为一个对象设置不同的编码，从而优化在某一场景下的效率，极大的提升了 Redis 的灵活性和效率。

举个栗子，在列表对象包含的元素比较少时，Redis 使用压缩列表作为列表对象的底层实现：
- 压缩列表比快速链表更节约内存，并且在元素数量较少时，在内存中以连续块方式报错的压缩列表比起快速列表可以更快的载入到缓存中；
- 随着列表对象包含的元素越来越多，使用压缩列表保存元素的优势消失时，对象就会将底层实现从压缩列表转为功能更强、也更适合保存大量元素的快速链表。

后面介绍完编码类型后，我们详细认识不同类型对应的各个编码方式。

encoding 属性有以下取值：
1. OBJ_ENCODING_RAW
2. OBJ_ENCODING_INT
3. OBJ_ENCODING_HT
4. OBJ_ENCODING_ZIPMAP
5. OBJ_ENCODING_LINKEDLIST
6. OBJ_ENCODING_ZIPLIST
7. OBJ_ENCODING_INTSET
8. OBJ_ENCODING_SKIPLIST
9. OBJ_ENCODING_EMBSTR
10. OBJ_ENCODING_QUICKLIST

对象的编码类型可以由 ```OBJECT ENCODING``` 命令获取。
![图 4 - 获取 key 的编码](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190522131136463_24618.png)

OBJECT ENCODING 命令输出值与 encoding 属性取值对应关系如下：
| 对象使用的底层数据结构    | 编码常量    |  OBJECT ENCODING 输出    |
| :-: | :-: | :-: |
|  简单动态字符串   |OBJ_ENCODING_RAW     |"raw"     |
|  整数   |OBJ_ENCODING_INT     |"embstr"     |
|  embstr 编码的简单动态字符串   |OBJ_ENCODING_EMBSTR     |"embstr"     |
|  字典   |OBJ_ENCODING_HT     |"embstr"     |
|  简单动态字符串   |OBJ_ENCODING_ZIPMAP     |"embstr"     |
|  双端链表   |OBJ_ENCODING_LINKEDLIST     |"embstr"     |
|  压缩列表   |OBJ_ENCODING_ZIPLIST     |"embstr"     |
|  快速列表   |OBJ_ENCODING_QUICKLIST     |"embstr"     |
|  整数集合   |OBJ_ENCODING_INTSET     |"embstr"     |
|  跳跃表和字典   |OBJ_ENCODING_SKIPLIST     |"embstr"     |

总结来看，如下图：
![11 种不同编码的数据对象](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190521202055424_22716.png)

十一种数据对象分别是：
1. 使用双端或快速列表实现的列表对象
2. 使用压缩列表实现的列表对象
3. 使用字典实现的哈希对象
4. 使用压缩列表实现的哈希对象
5. 使用字典实现的集合对象
6. 使用整数集合实现的集合对象
7. 使用压缩列表实现的有序集合对象
8. 使用跳跃表实现的有序集合对象
9. 使用普通 SDS 实现的字符串对象
10. 使用 embstr 编码的 SDS 实现的字符串对象
11. 使用整数值实现的字符串对象

### 对象验证
上面了解了 Redis 的对象系统，但这都是理论知识，有没有方法可以实时查看



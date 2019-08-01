[TOC]

Redis 底层使用了 ziplist、skiplist 和 quicklist 三种 list 结构来实现相关对象。顾名思义，ziplist 更节省空间、skiplist 则注重查找效率，quicklist 则对空间和时间进行折中。

在典型的双向链表中，我们有称为**节点**的结构，它表示列表中的每个值。每个节点都有三个属性：指向列表中的前一个和下一个节点的指针，以及指向节点中字符串的指针。而每个值字符串值实际上存储为三个部分：一个表示长度的整数、一个表示剩余空闲字节数的整数以及字符串本身后跟一个空字符。

可以看到，链表中的每一项都占用独立的一块内存，各项之间用地址指针（或引用）连接起来。这种方式会带来大量的内存碎片，而且地址指针也会占用额外的内存。这就是普通链表的**内存浪费**问题。

此外，在普通链表中执行随机查找操作时，它的时间复杂度为 O(n)，这对于注重效率的 Redis 而言也是不可接受的。这是普通链表的**查找效率太低**问题。

针对上述两个问题，Redis 设计了**ziplist（压缩列表）**、**skiplist（跳跃表）**和**快速链表**进行相关优化。

### 1 ziplist
对于 ziplist，它要解决的就是**内存浪费**的问题。也就是说，它的设计目标就是是**为了节省空间，提高存储效率**。

基于此，Redis 对其进行了特殊设计，使其成为一个经过特殊编码的**双向链表**。将表中每一项存放在前后连续的地址空间内，一个 ziplist 整体占用一大块内存。它是一个表（list），但其实不是一个链表（linked list）。

除此之前，ziplist 为了在细节上节省内存，对于值的存储采用了变长的编码方式，大概意思是说，对于大的整数，就多用一些字节来存储，而对于小的整数，就少用一些字节来存储。

也正是为了这种高效率的存储，ziplist 有很多 bit 级别的操作，使得代码变得较为晦涩难懂。不过不要紧，我们本节的目标之一是为了**了解 ziplist 对比普通链表，做了哪些优化，可以更好的节省空间**。

接下来我们来正式认识下压缩列表的结构。

#### 1.1 压缩列表的结构
一个压缩列表可以包含任意多个节点（entry），每个节点可以保存一个字节数组或者一个整数值。

图 1-1 展示了压缩列表的各个组成部分：

![图 1-1：压缩列表的结构](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190723124326650_29558.png)

相关字段说明如下：
- zlbytes：4 字节，表示 ziplist 占用的字节总数（包括 zlbytes 本身占用的 4 个字节）。
- zltail：4 字节，表示 ziplist 表中最后一项距离列表的起始地址有多少字节，也就是表尾节点的偏移量。通过此字段，程序可以快速确定表尾节点的地址。
- zllen：2 字节：表示 ziplist 的节点个数。要注意的是，由于此字段只有 16bit，所以可表达的最大值为 2^16-1。一旦列表节点个数超过这个值，就要遍历整个压缩列表才能获取真实的节点数量。
- entry：表示 ziplist 的节点。长度不定，由保存的内容决定。要注意的是，列表的节点（entry）也有自己的数据结构，后续会详细说明。
- zlend：ziplist 的结束标记，值固定为 255，用于标记压缩列表的末端。

图 1-2 展示了一个包含五个节点的压缩列表：

![图 1-2：包含五个节点的压缩列表](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190725202858638_4824.png)

- 列表 zlbytes 属性的值为 0xd2（十进制 210），表示压缩列表的总长为 210 字节。
- 列表 zltail 属性的值为 0xb3,（十进制 179），表示尾结点相比列表的起始地址有 179 字节的距离。假如列表的起始地址为 p，那么指针 p + 179 = entry5 的地址。
- 列表 zllen 属性的值为 0x5（十进制 5），表示压缩列表包含 5 个节点。

#### 1.2 列表节点的结构
节点的结构源码如下（ziplist.c）：
```
typedef struct zlentry {
    unsigned int prevrawlensize, prevrawlen;
    unsigned int lensize, len;
    unsigned int headersize;
    unsigned char encoding;
    unsigned char *p;
} zlentry;
```

如图 1-3，展示了压缩列表节点的结构。

![图 1-3：压缩列表节点结构](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190729125748351_27862.png)

- prevrawlen：表示前一个节点占用的总字节数。此字段是为了让 ziplist 能够从后向前遍历。
- encoding：此字段记录了节点的 content 属性中所保存的数据类型。
- lensize：此字段记录了节点所保存的数据长度。
- headersize：此字段记录了节点 header 的大小。
- *p：此字段记录了指向节点保存内容的指针。

#### 1.3 压缩列表如何节省了内存
回到我们最开始对普通链表的认识，普通链表中，每个节点包：
- 一个表示长度的整数
- 一个表示剩余空闲字节数的整数
- 字符串本身
- 结尾空字符。

以图 1-4 为例：

![图 1-4：普通链](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190730095415325_22348.png)

图 1-4 展示了一个普通链表的三个节点，这三个节点中，每个节点实际存储内容只有 1 字节，但是它们除了实际存储内容外，还都要有：
- 3 个指针 - 占用 3 byte
- 2 个整数 - 占用 2 byte
- 内容字符串 - 占用 1 byte
- 结尾空字符的空间 - 占用 1 byte

这样来看，存储 3 个字节的数据，至少需要 21 字节的开销。可以看到，这样的存储效率是很低的。

另一方面，普通链表通过前后指针来关联节点，地址不连续，多节点时容易产生内存碎片，降低了内存的使用率。

最后，普通链表对存储单位的操作粒度是 byte，这种方式在存储较小整数或字符串时，每个字节实际上会有很大的空间是浪费的。就像上面三个节点中，用来存储**剩余空闲字节数的整数**，实际存储空间只需要 1 bit，但是有了 1 byte 来表示剩余空间大小，这一个 byte 中，剩余 7 个 bit 就被浪费了。

那么，Redis 是如何使用 ziplist 来改造普通链表的呢？通过以下两方面：

一方面，ziplist **使用一整块连续内存，避免产生内存碎片，提高了内存的使用率**。

另一方面，ziplist **将存储单位的操作粒度从 byte 降低到 bit**，有效的解决了存储较小数据时，单个字节中浪费 bit 的问题。

### 2 skiplist
skiplist 是一种**有序**数据结构，它通过在每个节点中维持多个指向其他节点的指针，来达到快速访问节点的目的。

skiplist 本质上是一种查找结构，用于解决算法中的查找问题。即根据指定的值，快速找到其所在的位置。

此外，我们知道，"查找" 问题的解决方法一般分为两大类：**平衡树**和**哈希表**。有趣的是，skiplist 这种查找结构，因为其特殊性，并不在上述两大类中。但在大部分情况下，它的效率可以喝平衡树想媲美，而且跳跃表的实现要更为简单，所以有不少程序都使用跳跃表来代替平衡树。

本节没有介绍跳跃表的定义及其原理，有兴趣的童鞋可以参考[这里](https://lotabout.me/2018/skip-list/)。

认识了跳跃表是什么，以及做什么的，接下来，我们再来看下在 redis 中，是怎么实现跳跃表的。

在 ```server.h``` 中可以找到跳跃表的源码，如下：
```
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;

typedef struct zskiplistNode {
    robj *obj;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned int span;
    } level[];
} zskiplistNode;
```

Redis 中的 skiplist 和普通的 skiplist 相比，在结构上并没有太大不同，只是在一些细节上有以下差异：
- 分数（score）可以重复。就是说 Redis 中的 skiplist 在分值字段上是允许重复的，而普通的 skiplist 则不允许重复。
- 第一层链表不是单向链表，而是双向链表。这种方式可以用倒序方式获取一个范围内的元素。
- 比较时，除了比较 score 之外，还比较数据本身。在 Redis 的 skiplist 中，数据本身的内容是这份数据的唯一标识，而不是由 score 字段做唯一标识。此外，当多个元素分数相同时，还需要根据数据内容进行字典排序。

### 3 quicklist
对于 quicklist，在 ```quicklist.c``` 中有以下说明：
> A doubly linked list of ziplists

它是一个双向链表，并且是一个由 ziplist 组成的双向链表。

相关源码结构可在 ```quicklist.h``` 中查找，如下：
```
/* quicklistNode is a 32 byte struct describing a ziplist for a quicklist.
 * We use bit fields keep the quicklistNode at 32 bytes.
 * count: 16 bits, max 65536 (max zl bytes is 65k, so max count actually < 32k).
 * encoding: 2 bits, RAW=1, LZF=2.
 * container: 2 bits, NONE=1, ZIPLIST=2.
 * recompress: 1 bit, bool, true if node is temporarry decompressed for usage.
 * attempted_compress: 1 bit, boolean, used for verifying during testing.
 * extra: 12 bits, free for future use; pads out the remainder of 32 bits */
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

/* quicklistLZF is a 4+N byte struct holding 'sz' followed by 'compressed'.
 * 'sz' is byte length of 'compressed' field.
 * 'compressed' is LZF data with total (compressed) length 'sz'
 * NOTE: uncompressed length is stored in quicklistNode->sz.
 * When quicklistNode->zl is compressed, node->zl points to a quicklistLZF */
typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;

/* quicklist is a 32 byte struct (on 64-bit systems) describing a quicklist.
 * 'count' is the number of total entries.
 * 'len' is the number of quicklist nodes.
 * 'compress' is: -1 if compression disabled, otherwise it's the number
 *                of quicklistNodes to leave uncompressed at ends of quicklist.
 * 'fill' is the user-requested (or default) fill factor. */
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned int len;           /* number of quicklistNodes */
    int fill : 16;              /* fill factor for individual nodes */
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
} quicklist;
```

上面介绍链表的时候有说过，链表由多个节点组成。而对于 quicklist 而言，它的每一个节点都是一个 ziplist。quicklist 这样设计，其实就是我们篇头所说的，是一个空间和时间的折中。

ziplist 相比普通链表，主要优化了两个点：**降低内存开销**和**减少内存碎片**。正所谓，事物总是有两面性。ziplist 通过连续内存解决了普通链表的内存碎片问题，但与此同时，也带来了新的问题：**不利于修改操作**。

由于 ziplist 是一整块连续内存，所以每次数据变动都会引发一次内存的重分配。当在 ziplist 很大的时候，每次重分配都会出现大批量的数据拷贝操作，降低性能。

于是，结合了双向链表和 ziplist 的优点，就有了 quicklist。

quicklist 的基本思想就是，给每一个节点的 ziplist 分配合适的大小，避免出现因数据拷贝，降低性能的问题。这又是一个需要找平衡点的难题。我们先从存储效率上分析：
- 每个 quicklist 节点上的 ziplist 越短，则内存碎片越多。而内存碎片多了，就很有可能产生很多无法被利用的内存小碎片，降低存储效率。
- 每个 quicklist 节点上的 ziplist 越长，则为 ziplist 分配大块连续内存的难度就越大。有可能出现，内存里有很多较小块的内存，却找不到一块足够大的空闲空间分配给 ziplist 的情况。这同样会降低存储效率。

可见，一个 quicklist 节点上的 ziplist 需要保持一个合理的长度。这里的合理取决于实际应用场景。基于此，Redis 提供了一个配置参数，让使用者可以根据情况，自己调整：
> list-max-ziplist-size -2

这个参数可以取正值，也可以取负值。

当取正值的时候，表示按照**数据项个数**来限定每个 quicklist 节点上 ziplist 的长度。比如配置为 2 时，就表示 quicklist 的每个节点上的 ziplist 最多包含 2 个数据项。

当取负值的时候，表示按照**占用字节数**来限定每个 quicklist 节点上 ziplist 的长度。此时，它的取值范围是 [-1, -5]，每个值对应不同含义：
- -1：每个 quicklist 节点上的 ziplist 大小不能超过 4Kb；
- -2：每个 quicklist 节点上的 ziplist 大小不能超过 8Kb（默认值）；
- -3：每个 quicklist 节点上的 ziplist 大小不能超过 16Kb；
- -4：每个 quicklist 节点上的 ziplist 大小不能超过 32Kb；
- -5：每个 quicklist 节点上的 ziplist 大小不能超过 64Kb；

### 总结
1. 普通链表存在两个问题：**内存利用率低**和**容易产生内存碎片**。
2. ziplist 使用连续内存，减少内存碎片，提供内存利用率。
3. skiplist 可以用相对简单的实现，达到和平衡树相同的查找效率。
4. quicklist 汲取了普通链表和压缩链表的优点，保证性能的前提下，尽可能的提高内存利用率。

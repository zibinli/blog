继续撸我们的对象和数据类型。

上节我们一起认识了字符串和列表，接下来还有哈希、集合和有序集合。

### 1 哈希对象
哈希对象的可选编码分别是：ziplist 和 hashtable。

#### 1.1 ziplist 编码的哈希对象
ziplist 编码的哈希对象使用压缩列表作为底层实现。每当有新的键值对要加入到哈希对象时，程序会先将保存了**键**的压缩列表节点推入到表尾，然后再将保存了**值**的压缩列表节点推入到表尾。因此：
- 保存了键值对的两个节点总是紧挨在一起，保存键的节点在前，保存值的节点在后；
- 先添加到哈希对象中的键值对会被仿造压缩列表的表头方向，后添加的键值对会被放在压缩列表的表尾方向。

执行以下 HSET 命令，服务器将创建一个如图 9 所示的列表对象作为 profile 键的值：
```
127.0.0.1:6379> HSET profile name "Tom"
(integer) 1
127.0.0.1:6379> HSET profile age "25"
(integer) 1
127.0.0.1:6379> HSET profile career "Programer"
(integer) 1
127.0.0.1:6379> OBJECT ENCODING profile
"ziplist"
```

![图 9 - ziplist 编码的哈希对象](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190524123000055_32164.png)

其中对象所使用的压缩列表如图 10 所示：

![图 10 - ziplist 编码的哈希对象中压缩列表结构](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190524123202076_12872.png)

#### 1.2 hashtable 编码的
hashtable 编码的哈希对象使用字典作为底层实现。哈希对象中的每个键值对都使用一个字典键值对来保存：
- 字典中的每个键都是一个字符串对象，对象中保存了键值对的键；
- 字典中的每个值都是一个字符串对象，对象中保存了键值对的值。

如果前面的 profile 键使用的是 hashtable 编码的哈希对象，那么这个哈希对象应该如图 11 所示：

![图 11 - hashtable 编码的哈希对象](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190524124313495_9362.png)

#### 1.3 编码转换
当哈希对象同时符合下面两个条件时，将使用 ziplist 编码：
1. 哈希对象保存的所有键值对中，键和值的字符串长度都小于 64 个字节；
2. 哈希对象保存的键值对数量小于 512 个。

上述条件中的临界值对应 redis.conf 文件中的配置：```hash-max-ziplist-value``` 和 ```hash-max-ziplist-entries```。

在 3.2 版本中，新增一个哈希键值对时，实际上总是先创建一个 ziplist 编码的哈希对象，然后再进行转换检查。
关于何时进行编码转换，有两种情况发生：
1. 更新或新增键值对时，如果**值的字节数**大于 ```hash-max-ziplist-value```，将从 ziplist 编码转成 hashtable 编码；
2. 新增键值对时，如果哈希中的**键值对数量**大于 ```hash-max-ziplist-entries```，将从 ziplist 编码转成 hashtable 编码。

要注意的是，上述发生转换的情况，都不会出现从 hashtable 转成 ziplist 的情况，即使符合条件。

关于哈希编码转换的函数，可以参考 t_hash.c/hashTypeConvert，源码如下：
```
# o 是原始对象，enc 是目标编码。
void hashTypeConvert(robj *o, int enc) {
    if (o->encoding == OBJ_ENCODING_ZIPLIST) { // 原始编码是 OBJ_ENCODING_ZIPLIST 才进行转换
        hashTypeConvertZiplist(o, enc);
    } else if (o->encoding == OBJ_ENCODING_HT) {
        serverPanic("Not implemented");
    } else {
        serverPanic("Unknown hash encoding");
    }
}
```

### 2 集合对象
集合对象的可选编码有：intset 和 hashtable。

#### 2.1 intset 编码的集合对象
intset 编码的集合对象使用整数集合作为底层实现，集合对象包含的所有元素都被保存在整数集合里面。

执行以下 SADD 命令，将创建一个如图 12 所示的 intset 编码的集合对象：
```
127.0.0.1:6379> SADD numbers 1 3 5
(integer) 3
127.0.0.1:6379> OBJECT ENCODING numbers
"intset"
```
![图 12 - intset 编码的集合对象](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190524131821117_3903.png)

#### 2.2 hashtable 编码的集合对象
hashtable 编码的集合对象使用字典作为底层实现，字典的每个键都是一个字符串对象，每个字符串对象中又包含了一个集合元素，而字典的值则全部设置为 NULL。

执行以下 SADD 命令，将创建一个如图 13 所示的 hashtable 编码的集合对象：
```
127.0.0.1:6379> SADD fruits "apple" "banana" "cherry"
(integer) 3
127.0.0.1:6379> OBJECT ENCODING fruits
"hashtable"
```
![图 13 - hashtable 编码的集合对象](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190524131844230_29702.png)

#### 2.3 编码转换
当集合对象同时满足以下两个条件时，对象使用 intset 编码：
1. 集合对象保存的所有元素都是可以被 long double 表示整数值；
2. 集合对象保存的元素数量不超过 512 个。

上述条件中的临界值对应 redis.conf 文件中的配置：```set-max-intset-entries```。

对于集合对象，在新增第一个键值对时，就会对键值对中的值进行检查，如果是符合条件的整数值，就会创建一个 intset 编码的集合对象，否则，则创建 hashtable 编码的集合对象。

关于何时进行编码转换，有两种情况发生：
1. 更新或新增键值对时，如果**值**不能用 long double 表示，将从 intset 编码转成 hashtable 编码；
2. 新增键值对时，如果集合中的**键值对数量**大于 ```set-max-intset-entries```，将从 intset 编码转成 hashtable 编码。

同样，上述发生转换的情况，都不会出现从 hashtable 转成 intset 的情况，即使符合条件。

关于哈希编码转换的函数，可以参考 t_set.c/setTypeConvert，源码如下：
```
# setobj 是原始对象，enc 是目标编码。
hvoid setTypeConvert(robj *setobj, int enc) {
    setTypeIterator *si;
    serverAssertWithInfo(NULL,setobj,setobj->type == OBJ_SET && setobj->encoding == OBJ_ENCODING_INTSET);
    if (enc == OBJ_ENCODING_HT) { // 只能转成 OBJ_ENCODING_HT 编码
        int64_t intele;
        dict *d = dictCreate(&setDictType,NULL);
        robj *element;
        /* Presize the dict to avoid rehashing */
        dictExpand(d,intsetLen(setobj->ptr));
        /* To add the elements we extract integers and create redis objects */
        si = setTypeInitIterator(setobj);
        while (setTypeNext(si,&element,&intele) != -1) {
            element = createStringObjectFromLongLong(intele);
            serverAssertWithInfo(NULL,element,
                                dictAdd(d,element,NULL) == DICT_OK);
        }
        setTypeReleaseIterator(si);
        setobj->encoding = OBJ_ENCODING_HT;
        zfree(setobj->ptr);
        setobj->ptr = d;
    } else {
        serverPanic("Unsupported set conversion");
    }
}
```

### 3 有序集合对象
有序集合对象的可选编码有：ziplist 和 skiplist。

#### 3.1 ziplist 编码的有序集合对象
intset 编码的集合对象使用压缩列表作为底层实现。每个集合元素使用两个紧挨在一起的压缩列表节点来保存。第一个节点保存元素的成员（member），第二个成员保存元素的分值（score）。

压缩列表内的集合元素按分值从小到大排序，分值较小的元素被放置在表头的方向，而分值较大的元素则被放置在靠近表尾的方向。

执行以下 SADD 命令，将创建一个如图 14 所示的 ziplist 编码的集合对象：
```
127.0.0.1:6379> ZADD price 8.5 apple 5.0 banana 6.0 cherry
(integer) 3
127.0.0.1:6379> OBJECT ENCODING price
"ziplist"
```

![图 14 - ziplist 编码的有序集合对象](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190527122407446_19652.png)

底层结构 ziplist 如图 15 所示：
![图 15 - ziplist 编码的有序集合，数据在压缩列表中按分值从小到大排列](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190527122602034_6298.png)

#### 3.2 skiplist 编码的集合对象
skiplist 编码的集合对象使用 zset 作为底层实现。一个 zset 结构同时包含一个字典和一个跳跃表。结构源码如下：
```
# server.h
typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```

zset 结构中的 zsl 跳跃表按分值从小到大保存了所有集合元素，每个跳跃表节点都保存了一个集合元素。

跳跃表节点的 object 属性保存了元素的成员，而跳跃表节点的 score 属性则保存了元素的分支。**程序通过这个跳跃表，对有序集合进行范围型操作。比如 ZRANK、ZRANGE 等命令就是基于跳跃表 API 来实现的。

除此之外，zset 结构中的 dict 字典为有序集合创建了一个从成员到分值的映射。字典中的每个键值对都保存了一个集合元素：字典中的键保存了元素的成员，而字典的值则保存了元素的分值。通过这个字典，程序用 O(1) 复杂度查找给定成员的分值。

有序集合每个元素的成员都是一个字符串对象，而每个元素的分值都是一个 double 类型的浮点数。值得一提的是，虽然 zset 结构同时使用跳跃表和字典保存了有序集合的元素，但这两种数据结构都会通过指针来共享相同元素的成员和分值，所以不会产生任何重复成员和分值，也不会因此而浪费额外的内存。

如果前面 price 键创建的不是 ziplist 编码的有序集合对象，而是 skiplist 编码，那么这个有序集合对象将会如图 16 所示，而对象所使用的 zset 结果将会如图 17 所示：
![图 16 - skiplist 编码的有序集合对象](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190527124054221_23339.png)

![图 17 - 有序集合元素同时被存放在字典和跳跃表中](https://raw.githubusercontent.com/zibinli/blog/master/Redis/_v_images/20190527124127930_234.png)

图 17 中，为了展示方便，重复展示了各个元素的成员和分值。实际上，它们是共享元素的成员和分值。

#### 3.3 编码转换
当有序集合对象同时满足以下两个条件时，对象使用 ziplist 编码：
1. 有序集合对象保存的元素数量不超过 128 个。
2. 有序集合中保存的所有元素成员的长度都小于 64 个字节。

上述条件中的临界值对应 redis.conf 文件中的配置：```zset-max-ziplist-entries``` 和 ```zset-max-ziplist-value```。

对于集合对象，在新增键值对时，就会对集合元素以及键值对中的值进行检查，如果是符合条件，就会创建一个 ziplist 编码的集合对象，否则，则创建 skiplist 编码的集合对象。对应源码如下：
```
# t_zset.c/zaddGenericCommand
...
zobj = lookupKeyWrite(c->db,key);
if (zobj == NULL) {
    if (xx) goto reply_to_client; /* No key + XX option: nothing to do. */
    if (server.zset_max_ziplist_entries == 0 ||
        server.zset_max_ziplist_value < sdslen(c->argv[scoreidx+1]->ptr))
    {
        # 对象元素数量为 0，或者
        zobj = createZsetObject();
    } else {
        zobj = createZsetZiplistObject();
    }
    dbAdd(c->db,key,zobj);
} else {
    if (zobj->type != OBJ_ZSET) {
        addReply(c,shared.wrongtypeerr);
        goto cleanup;
    }
}
```

### 总结
1. 哈希对象有 **ziplist** 和 **hashtable** 编码。
2. 集合对象有 **intset** 和 **hashtable** 编码。
3. 有序集合对象有 **ziplist** 和 **skiplist** 编码。
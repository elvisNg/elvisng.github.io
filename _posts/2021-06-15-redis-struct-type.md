---
layout: post
title: "Redis-Struct-Type"
subtitle: 'redis基础数据类型'
author: "Elvis"
header-style: text
mermaid: true
tags:
  - Redis
---



### Redis的数据结构与实现

#### String

> **常用命令:** set,get,EXISTS,DEL,SETEX,decr,incr,mget 等。 Redis 官方提供了在线的调试器，可以在里面敲入命令进行操作：http://try.redis.io/#run

String数据结构是简单的key-value类型，value其实不仅可以是String，也可以是数字。
常规key-value缓存应用；
常规计数：微博数，粉丝数等。



##### String的底层数据结构

Redis 中的字符串是一种 **动态字符串**，这意味着使用者可以修改，它的底层实现有点类似于 Java 中的 **ArrayList**，有一个字符数组，从源码的 **sds.h/sdshdr 文件** 中可以看到 Redis 底层对于字符串的定义 **SDS**，即 *Simple Dynamic String* 结构：

```c
COPY/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

你会发现同样一组结构 Redis 使用泛型定义了好多次，**为什么不直接使用 int 类型呢？**

因为当字符串比较短的时候，len 和 alloc 可以使用 byte 和 short 来表示，**Redis 为了对内存做极致的优化，不同长度的字符串使用不同的结构体来表示。**



##### 为什么SDS不直接用C的String

- **获取字符串长度为 O(N) 级别的操作** → 因为 C 不保存数组的长度，每次都需要遍历一遍整个数组；
- 不能很好的杜绝 **缓冲区溢出/内存泄漏** 的问题 → 跟上述问题原因一样，如果执行拼接 or 缩短字符串的操作，如果操作不当就很容易造成上述问题；
- C 字符串 **只能保存文本数据** → 因为 C 语言中的字符串必须符合某种编码（比如 ASCII），例如中间出现的 `'\0'` 可能会被判定为提前结束的字符串而识别不了；



##### Redis怎么杜绝内存泄露做字符串追加？

Redis String 扩容源码如下：

```c
COPY/* Append the specified binary-safe string pointed by 't' of 'len' bytes to the
 * end of the specified sds string 's'.
 *
 * After the call, the passed sds string is no longer valid and all the
 * references must be substituted with the new pointer returned by the call. */
sds sdscatlen(sds s, const void *t, size_t len) {
    // 获取原字符串的长度
    size_t curlen = sdslen(s);

    // 按需调整空间，如果容量不够容纳追加的内容，就会重新分配字节数组并复制原字符串的内容到新数组中
    s = sdsMakeRoomFor(s,len); 
    if (s == NULL) return NULL;   // 内存不足
    memcpy(s+curlen, t, len);     // 追加目标字符串到字节数组中
    sdssetlen(s, curlen+len);     // 设置追加后的长度
    s[curlen+len] = '\0';         // 让字符串以 \0 结尾，便于调试打印
    return s;
}
```

- **注：Redis 规定了字符串的长度不得超过 512 MB**



#### List

> **常用命令:** lpush,rpush,lpop,rpop,lrange等

list 就是链表，Redis list 的应用场景非常多，也是Redis最重要的数据结构之一，比如微博的关注列表，粉丝列表，消息列表等功能都可以用Redis的 list 结构来实现。

Redis list 的实现为一个双向链表，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销。

另外可以通过 lrange 命令，就是从某个元素开始读取多少个元素，可以基于 list 实现分页查询，这个很棒的一个功能，基于 redis 实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西（一页一页的往下走），性能高。



Redis 的列表相当于 Java 语言中的 **LinkedList**，注意它是链表，而且是双向链表。这意味着 list 的插入和删除操作非常快，时间复杂度为 O(1)，但是索引定位很慢，时间复杂度为 O(n)。

我们可以从源码的 `adlist.h/listNode` 来看到对其的定义：

```c
COPY/* Node, List, and Iterator are the only data structures used currently. */

typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct listIter {
    listNode *next;     //迭代器当前指向的节点（名字叫next有点迷惑）
    int direction;      //迭代方向，可以取以下两个值：AL_START_HEAD和AL_START_TAIL
} listIter

#define AL_START_HEAD 0 //正向迭代：从表头向表尾进行迭代
#define AL_START_TAIL 1 //反向迭代：从表尾到表头进行迭代

typedef struct list {
    listNode *head;     //链表头结点指针
    listNode *tail;     //链表尾结点指针

    //下面的三个函数指针就像类中的成员函数一样
    void *(*dup)(void *ptr);    //复制链表节点保存的值
    void (*free)(void *ptr);    //释放链表节点保存的值
    int (*match)(void *ptr, void *key); //比较链表节点所保存的节点值和另一个输入的值是否相等
    unsigned long len;      //链表长度计数器
} list;
```

可以看到，多个 listNode 可以通过 `prev` 和 `next` 指针组成双向链表：

![image-20210615151511734](https://raw.githubusercontent.com/elvisNg/elvisng.github.io/master/img/in-post/post-redis/redis-listnode.png)

虽然仅仅使用多个 listNode 结构就可以组成链表，但是使用 `adlist.h/list` 结构来持有链表的话，操作起来会更加方便：

![image-20210615151736043](https://raw.githubusercontent.com/elvisNg/elvisng.github.io/master/img/in-post/post-redis/list.png)

* `listDup()`用于复制链表, 如果用户实现了dup函数, 则会使用它复制链表结点的value。
* `listRelease(list *list)`用于释放表头和列表，如果用户实现了free函数，则会使用它来释放该节点的值
*  `listSearchKey()`通过循环的方式在O(N)的时间复杂度下查找值, 若用户实现了match函数, 则用它进行匹配, 否则使用按引用匹配。



##### list 实现队列

队列是先进先出的数据结构，常用于消息排队和异步逻辑处理，它会确保元素的访问顺序：

```bash
COPY> RPUSH books python java golang
(integer) 3
> LPOP books
"python"
> LPOP books
"java"
> LPOP books
"golang"
> LPOP books
(nil)
```

##### list 实现栈

栈是先进后出的数据结构，跟队列正好相反：

```bash
COPY> RPUSH books python java golang
> RPOP books
"golang"
> RPOP books
"java"
> RPOP books
"python"
> RPOP books
(nil)
```



### Hash

> **常用命令：** hget,hset,hgetall,hmset 等。

hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象，后续操作的时候，你可以直接仅仅修改这个对象中的某个字段的值。 比如我们可以 hash 数据结构来存储用户信息，商品信息等等



Redis 中的字典相当于 Java 中的 **HashMap**，内部实现也差不多类似，都是通过 **“数组 + 链表”** 的链地址法来解决部分 **哈希冲突**，同时这样的结构也吸收了两种不同数据结构的优点。源码定义如 `dict.h/dictht` 定义：

```c
typedef struct dictht {

    //桶
    dictEntry **table;

    //指针数组大小
    unsigned long size;

    //指针数组掩码，用于计算索引值
    unsigned long sizemask;

    //hash表现有节点数量
    unsigned long used;
} dictht;

typedef struct dict {

    //类型处理函数
    dictType *type;

    //类型处理函数私有值
    void *privdata;

    //两个hash表
    dictht ht[2];

    //rehash标示，为－1表示不在rehash，不为0表示正在rehash的桶
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

    //当前正在运行的安全迭代器数量
    int iterators; /* number of iterators currently running */
} dict;
```

`table` 属性是一个数组，数组中的每个元素都是一个指向 `dict.h/dictEntry` 结构的指针，而每个 `dictEntry` 结构保存着一个键值对：

```c
/*
 * hash节点
 */

typedef struct dictEntry {

    //键
    void *key;

    //值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    //指向下一个节点 链地址法来解决hash冲突
    struct dictEntry *next;
} dictEntry;
```

`iterators`Hash的迭代器

```c
typedef struct dictIterator {
    dict *d;                    //被迭代的字典
    long index;                 //迭代器当前所指向的哈希表索引位置
    int table, safe;            //table表示正迭代的哈希表号码，ht[0]或ht[1]。safe表示这个迭代器是否安全。
    dictEntry *entry, *nextEntry;   //entry指向当前迭代的哈希表节点，nextEntry则指向当前节点的下一个节点。
    /* unsafe iterator fingerprint for misuse detection. */
    long long fingerprint;      //避免不安全迭代器的指纹标记
} dictIterator;
```

迭代器分为安全迭代器和不安全迭代器：

* 非安全迭代器只能进行Get等读的操作, 而安全迭代器则可以进行iterator支持的任何操作；
* 由于dict结构中保存了safe iterators的数量，如果数量不为0， 是不能进行下一步的rehash的; 因此安全迭代器的存在保证了遍历数据的准确性；
* 在非安全迭代器的迭代过程中, 会通过fingerprint方法来校验iterator在初始化与释放时字典的hash值是否一致; 如果不一致说明迭代过程中发生了非法操作；



#### dict的结构如下：

![image-20210615154035183](https://raw.githubusercontent.com/elvisNg/elvisng.github.io/master/img/in-post/post-redis/hash-struct.png)



#### 往HashDict添加元素的过程：

![image-20210615155553317](https://raw.githubusercontent.com/elvisNg/elvisng.github.io/master/img/in-post/post-redis/hash-dict-add-entry.png)





#### Rehash

当哈希表的大小不能满足需求，就可能会有两个或者以上数量的键被分配到了哈希表数组上的同一个索引上，于是就`发生冲突（collision）`，在Redis中解决冲突的办法是`链接法（separate chaining）`。但是需要`尽可能避免冲突`，希望哈希表的负载因子（load factor），维持在一个合理的范围之内，就需要对哈希表进行扩展或收缩。



#### Rehash扩缩容的条件

正常情况下，当 hash 表中 **元素的个数等于第一维数组的长度时**，就会开始扩容，扩容的新数组是 **原数组大小的 2 倍**。不过如果 Redis 正在做 `bgsave(持久化命令)`，为了减少内存也得过多分离，Redis 尽量不去扩容，但是如果 hash 表非常满了，**达到了第一维数组长度的 5 倍了**，这个时候就会 **强制扩容**。



当 hash 表因为元素逐渐被删除变得越来越稀疏时，Redis 会对 hash 表进行缩容来减少 hash 表的第一维数组空间占用。所用的条件是 **元素个数低于数组长度的 10%**，缩容不会考虑 Redis 是否在做 `bgsave`。



#### 渐进式 rehash

大字典的扩容是比较耗时间的，需要重新申请新的数组，然后将旧字典所有链表中的元素重新挂接到新的数组下面，这是一个 O(n) 级别的操作，作为单线程的 Redis 很难承受这样耗时的过程，所以 Redis 使用 **渐进式 rehash** 小步搬迁：

![image-20210615160238957](https://raw.githubusercontent.com/elvisNg/elvisng.github.io/master/img/in-post/post-redis/hash-append-newone.png)

1. 创建一个比 ht[0]->table 更大的 ht[1]->table ， size为大于used*2的2的指数, 开始值为4；

2. 将 ht[0]->table 中的所有键值对迁移到 ht[1]->table ；

3. 将原有 ht[0] 的数据清空，并将 ht[1] 替换为新的 ht[0] ；

##### Hash查询

往Hash里面查询数据时，查询时会同时查询两个 hash 结构，然后在后续的定时任务以及 hash 操作指令中，循序渐进的把旧字典的内容迁移到新字典中。当搬迁完成了，就会使用新的 hash 结构取而代之。



### Set

> **常用命令：**
> sadd,spop,smembers,sunion 等

set 对外提供的功能与list类似是一个列表的功能，特殊之处在于 set 是可以自动排重的。

当你需要存储一个列表数据，又不希望出现重复数据时，set是一个很好的选择，并且set提供了判断某个成员是否在一个set集合内的重要接口，这个也是list所不能提供的。

可以基于 set 轻易实现交集、并集、差集的操作。

比如：在微博应用中，可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。Redis可以非常方便的实现如共同关注、共同粉丝、共同喜好等功能。这个过程也就是求交集的过程，具体命令如下：

```bash
sinterstore key1 key2 key3     ##将交集存在key1内
```



### ZSet

> **常用命令：** zadd,zrange,zrem,zcard等

和set相比，sorted set增加了一个权重参数score，使得集合中的元素能够按score进行有序排列。

**举例：** 在直播系统中，实时排行信息包含直播间在线用户列表，各种礼物排行榜，弹幕消息（可以理解为按消息维度的消息排行榜）等信息，适合使用 Redis 中的 Sorted Set 结构进行存储。



`ZSet`类似于 Java 中 **SortedSet** 和 **HashMap** 的结合体，一方面它是一个 set，保证了内部 value 的唯一性，另一方面它可以为每个 value 赋予一个 score 值，用来代表排序的权重。

它的内部实现用的是一种叫做 **「跳跃表」** 的数据结构

跳跃表节点 zskiplistNode

```c
typedef struct zskiplistNode {
    robj *obj;                          //保存成员对象的地址
    double score;                       //分值
    struct zskiplistNode *backward;     //后退指针
    struct zskiplistLevel {
        struct zskiplistNode *forward;  //前进指针
        unsigned int span;              //跨度
    } level[];                          //层级，柔型数组
} zskiplistNode;


```

跳跃表表头 zskiplist（记录跳跃表信息）

```c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;//header指向跳跃表的表头节点，tail指向跳跃表的表尾节点
    unsigned long length;       //跳跃表的长度或跳跃表节点数量计数器，除去第一个节点
    int level;                  //跳跃表中节点的最大层数，除了第一个节点
} zskiplist;
```



通俗点讲，跳跃表其实就是把元素用多级指针连起来的形成的金字塔结构的平衡树


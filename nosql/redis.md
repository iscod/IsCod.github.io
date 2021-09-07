# Redis

Redis包含了六个主要的底层数据结构：`动态字符串` `链表` `字典` `跳跃表` `整数集合` `压缩列表`

通过底层数据结构实现了五个基本对象：`字符串` `列表` `哈希` `集合` `有序集合`

## 数据结构

底层数据结构：`动态字符串` `链表` `字典` `跳跃表` `整数集合` `压缩列表`

### 动态字符串

`src/sds.h`文件定义了`sds`结构：

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

* `len`记录sds已使用的字节数量
* `buf`用于保存字符串的字节数组 

### 链表(list)
`链表(list)`提供高效的节点重排能力，以及顺序的节点访问，可以通过增删节点灵活调整链表长度

链表应用于`列表键`，在`列表键`元素数量比较多，或者元素成员是比较长的字符串时，Redis会使用`链表(list)`作为`列表键`的底层实现

`src/adlist.h`文件定义了`listNode`和`list`结构：

```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);//节点释放函数
    int (*match)(void *ptr, void *key);
    unsigned long len;//链表包含的节点数量
} list;
```

* `listNode`定义了链表节点
* `list`定义了链表结构

### 字典(dict)

Redis中的字典使用哈希表作为底层实现，一个哈希表包含多个哈希节点，每个哈希表节点就保存了字典中的一个键值对。

`src/dict.h`文件定义了`dictht`、`dictEntry`、`dict`结构：

`dictht`结构:
```c
//哈希表
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

//哈希节点
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

//字典
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

### 跳跃表(zskiplist)

`跳跃表(zskiplist)`是一个有序数据结构, 它通过维持多个指向其他节点的指针，从而达到快速访问节点的目的。
`跳跃表(zskiplist)`应用于`有序集合`，在有序集合元素数量比较多，或者元素成员是比较长的字符串时，Redis会使用`跳跃表(zskiplist)`作为`有序集合键`的底层实现

`src/server.h`文件定义了`zskiplistNode`和`zskiplist`两个结构：

`zskiplist`结构:

```c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;//记录跳跃表表头节点和尾节点
    unsigned long length;//记录跳跃表长度, 既跳跃表节点数量
    int level;//记录跳跃表内, 层数最大的节点层数
} zskiplist;

typedef struct zskiplistNode {
    sds ele;
    double score;//分值
    struct zskiplistNode *backward;//后退指针
    struct zskiplistLevel {
        struct zskiplistNode *forward;//前进指针
        unsigned long span;//层的跨度
    } level[];//层
} zskiplistNode;
```
### 整数集合

### 压缩列表

## 对象

Redis通过上述章节中介绍的底层数据结构，构建一个对象系统来实现`键值对`数据库

Redis实现了`字符串` `列表` `哈希` `集合` `有序集合` 这五类对象，每一种对象都至少使用了一种数据结构来实现

`src/server.h`文件定义了`RedisObject`结构:

```c
typedef struct RedisObject {
    unsigned type:4;//对象类型
    unsigned encoding:4;//编码
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;//引用计数
    void *ptr;//指向底层实现数据结构的指针
} robj;
```

#### type 记录了对象的类型

`src/server.h`文件定义了`type`对象类型:

```c
/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```

可通过Redis`TYPE`命令查看对象`key`的`type`对象类型

```sh
127.0.0.1:6379[1]> TYPE A
string
```

#### encoding 记录了对象所使用的底层编码

`src/server.h`文件定义了`encoding`编码类型:

```c
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```

可通过Redis命令`OBJECT ENCODING`命令查看对象`key`的底层数据结构

```sh
127.0.0.1:6379[1]> OBJECT ENCODING user_score_rank
"ziplist"
127.0.0.1:6379[1]> set A string
OK
127.0.0.1:6379[1]> OBJECT ENCODING A
"embstr"
```

#### lru

lru 记录了对象最后一次被程序访问的时间

可通过Redis命令`OBJECT IDLETIME`查看对象`key`的空转时长，既当前的时间减去`lru`的时间

```sh
127.0.0.1:6379[1]> OBJECT IDLETIME A
(integer) 604
```

#### 字符串对象

字符串对象的编码对象可以是`int` `emstr` `raw` 

#### 列表对象

列表对象的编码对象可以是`ziplist` `linkedlist`

#### 哈希对象

列表对象的编码对象可以是`ziplist` `hashtable`

#### 集合对象

列表对象的编码对象可以是`intset` `hashtable`

#### 有序集合对象

有序集合对象的编码对象可以是`ziplist` `skiplist`

## 内存回收

由于C语言不具备自动内存回收功能，所以`Redis`在自身的对象系统中构建了`应用计数(refcount)`来实现内存的回收机制，通过`refcount`记录跟踪对象的引用计数，实现在适当时机的自动释放对象和内存回收。

在`src/server.h`文件可以看到`RedisObject`结构:

```c
typedef struct RedisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;//引用计数
    void *ptr;
} robj;
```

`应用计数(refcount)`会随着对象的使用状态而不断变化

* 创建对象时，引用计数会被初始化为 1
* 对象没一个程序使用时，引用计数会被增加 1
* 对象不被一个程序使用时，引用计数会被减 1
* 对象的引用计数变为 0 时，对象所占用内存会被释放

通过Redis命令`OBJECT REFCOUNT`可以查看对象的引用计数

```sh
127.0.0.1:6379> set A iscod
OK
127.0.0.1:6379> OBJECT REFCOUNT A
(integer) 1
```

### 对象共享

Redis在内存处理上除了`refcount`引用计数之外，还设计了对象共存

Redis在初始化服务器时，创建了一万个字符串对象，这些对象包含了从 0 到 9999 的所有整数值，当有其它程序或对象需要使用到这些字符串对象时，服务器就会共享这些对象，而不是新创建对象。

共享字符串的对象数量由`server.h`中`OBJ_SHARED_INTEGERS`指定

```c
#define OBJ_SHARED_INTEGERS 10000
```

如以下例子：

```sh
127.0.0.1:6379[1]> set A 1
OK
127.0.0.1:6379[1]> OBJECT REFCOUNT A
(integer) 2147483647
127.0.0.1:6379[1]> set A 100001
OK
127.0.0.1:6379[1]> OBJECT REFCOUNT A
(integer) 1
```

对象`A` 的键值为 `1`时, `A`的对象引用计数是 `1`。当A的键值设置 `100001` 时，引用计数为 `1`

## Lua脚本

Redis 通过`EVAL`命令来执行`lua`脚本, 使用`lua`脚本可以很方便的获取`Redis`多个命令的结果，比如`ZSCORE`获取多个`member`的结果时。

### EVAL

EVAL的第一个参数是一段 Lua 脚本程序。 这段Lua脚本不需要（也不应该）定义函数。

EVAL的第二个参数是`key`的个数，后面的参数（从第三个参数），表示在脚本中所用到的那些 Redis 键(key)，这些键名参数可以在 Lua 中通过全局变量`KEYS`数组引用，用 1 为基址的形式访问( KEYS[1] ， KEYS[2] ，以此类推)。

在命令的最后，是非`key`参数的附加参数 arg [arg …] ，可以在 Lua 中通过全局变量 ARGV 数组访问，访问的形式和 KEYS 变量类似( ARGV[1] 、 ARGV[2] ，诸如此类)。

举例说明：

```sh
127.0.0.1:6379[1]> eval "local res={} for i,v in ipairs(ARGV) do res[i]=Redis.call('ZSCORE', KEYS[1], v); end return res" 1 key member1 member2 member3
```

- Redis cluster 环境下使用要保证所有的`key`在同一个`slot`, 否则会报`ERR 'EVAL' command keys must in same slot`

## 持久化

`Redis`提供多种类型的持久化方式：

- RDB

`RDB`持久化方式能够在指定的时间间隔对你的数据进行快照存储。

- AOF

`AOF`持久化方式是记录每次对服务器的写操作, 当服务器重启的时候会重新执行这些命令来恢复原始数据。
`AOF`命令以Redis协议追加保存每次写的操作到文件末尾。
`Redis`还能对`AOF`文件进行后台重写,使得`AOF`文件的体积不至于过大。

- 无持久化

如果你只希望你的数据在服务器运行的时候存在,你也可以不使用任何持久化方式。

- RDB + AOF

你也可以同时开启两种持久化方式, 在这种情况下, 当`Redis`重启的时候会优先载入AOF文件来恢复原始的数据, 因为在通常情况下`AOF`文件保存的数据集要比`RDB`文件保存的数据集要完整。


最重要是要了解RDB和AOF持久化方式的不同, 让我们以RDB持久化方式开始:


### 如何选择使用哪种持久化方式？

一般来说, 如果想达到足以媲美`PostgreSQL`的数据安全性, 你应该同时使用两种持久化功能。

如果你非常关心你的数据, 但仍然可以承受数分钟以内的数据丢失, 那么你可以只使用`RDB`持久化。

有很多用户都只使用`AOF`持久化, 但我们并不推荐这种方式：因为定时生成`RDB`快照（snapshot）非常便于进行数据库备份, 并且`RDB` 恢复数据集的速度也要比`AOF`恢复的速度要快, 除此之外, 使用`RDB`还可以避免之前提到的`AOF`程序的bug。

`因为以上提到的种种原因， 未来我们可能会将 AOF 和 RDB 整合成单个持久化模型。 （这是一个长期计划。） 接下来的几个小节将介绍 RDB 和 AOF 的更多细节。`


## 集群

Redis集群是由多个Redis节点组成的数据共享的程序集（最少三个节点）

### 数据分片

Redis集群没有使用一致性hash, 而是使用哈希槽的该概念。

Redis 集群有16384个哈希槽, 每个key通过CRC16校验后对16384取模来决定放置哪个槽。
集群的每个节点负责一部分hash槽。

举个例子,比如当前集群有3个节点, 那么:

* 节点 A 包含 0 到 5500号哈希槽.
* 节点 B 包含5501 到 11000 号哈希槽.
* 节点 C 包含11001 到 16384号哈希槽.

* 参考
    * [Redis集群](http://www.Redis.cn/topics/cluster-tutorial)
    * [Redis设计与实现](https://www.bookstack.cn/read/Redisbook/2d294542c86f1acf.md)


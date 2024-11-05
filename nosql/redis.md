# Redis

Redis 内部用**六种主要底层结构**实现多种对外类型：`动态字符串（SDS）`、`链表`、`字典`、`跳跃表`、`整数集合`、`压缩列表`（新版本列表/哈希等逐步过渡到 **listpack**，概念上仍可与 ziplist 对照理解）。

对外常见的**值类型**包括：`String`、`List`、`Set`、`Sorted Set`、`Hash`，以及 **`Stream`**、**`Bitmap`**、**`Bitfield`** 等扩展能力。下文先讲底层结构与对象系统，再涉及事务、Lua、持久化与集群。

**结构导读**：数据结构 → 对象与编码 → 内存与引用计数 → 大 Key / 热 Key → 事务 → Lua / Pipeline → 分布式锁 → 持久化 → 集群 → 性能与阻塞 → 参考。

---

## 数据结构

六种底层结构概览：`SDS`、`链表`、`字典`、`跳跃表`、`整数集合`、`压缩列表`。

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
    unsigned long length;//记录跳跃表长度，即节点数量
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
### 整数集合（intset）

当 **Set** 只包含整数且元素个数较少时，可用整数数组有序存储，在升级（如插入非整数）时再迁移到哈希表等结构，以节省内存。

### 压缩列表（ziplist / listpack）

将多个小元素顺序压紧在一块连续内存里，适合元素少且短的 **List / Hash / ZSet** 场景。Redis 新版本逐步用 **listpack** 替代 ziplist，思路仍是紧凑序列化以省内存，具体编码以当前版本为准。

## 对象

Redis 用上述底层结构封装成**对象系统**，对外表现为键值数据库。

经典五类对象：**字符串、列表、哈希、集合、有序集合**；每种对象在内部会随数据形态切换 **encoding**（编码）。

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

#### 对象类型

`src/server.h`文件定义了`type`对象类型:

```c
/* The actual`Redis`Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```


可通过Redis`TYPE`命令查看对象`key`的`type`对象类型

```bash
127.0.0.1:6379[1]> TYPE A
string
```

#### 编码

encoding 记录了对象所使用的底层编码

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

```bash
127.0.0.1:6379[1]> OBJECT ENCODING user_score_rank
"ziplist"
127.0.0.1:6379[1]> set A string
OK
127.0.0.1:6379[1]> OBJECT ENCODING A
"embstr"
```

* 字符串对象（`OBJ_STRING`）的编码常见为 `int`、`embstr`、`raw`（具体以版本与数据为准）。
* 列表对象（`OBJ_LIST`）在新版中多为 `quicklist`（由多个紧凑节点组成）；老版本曾用 `ziplist` / `linkedlist`。
* 集合对象(OBJ_SET)的编码可以是`intset` `hashtable`
* 有序集合对象(OBJ_ZSET)的编码可以是`ziplist` `skiplist`
* 哈希对象(OBJ_HASH)的编码可以是 `ziplist` `hashtable`

#### lru

lru 记录了对象最后一次被程序访问的时间

可通过 `OBJECT IDLETIME` 查看某 key 的空转时长，**即**当前时间相对对象最近访问记录的间隔（与 LRU/LFU 策略相关）。

```bash
127.0.0.1:6379[1]> OBJECT IDLETIME A
(integer) 604
```

#### 引用计数

`refcount` 用于跟踪对象被引用次数，降至 0 时释放内存；机制见下一节。

## 内存回收

C 语言无自动 GC，Redis 在对象系统里用**引用计数（`refcount`）**在适当时机释放对象内存。

`refcount` 随对象被引用情况变化：

* 创建对象时，引用计数会被初始化为 1
* 对象被一个程序使用时，引用计数会被增加 1
* 对象不被一个程序使用时，引用计数会被减 1
* 对象的引用计数变为 0 时，对象所占用内存会被释放

通过Redis命令`OBJECT REFCOUNT`可以查看对象的引用计数

```bash
127.0.0.1:6379> set A iscod
OK
127.0.0.1:6379> OBJECT REFCOUNT A
(integer) 1
```

### 对象共享

除 `refcount` 外，Redis 还对**小整数字符串**做**对象共享**。

服务器会预先创建一批共享的整数字符串对象（数量由 `OBJ_SHARED_INTEGERS` 等配置决定，常见为 0～9999）。业务 `SET` 到这些值时可能直接引用共享对象，而不是新建一份。

共享字符串的对象数量由`server.h`中`OBJ_SHARED_INTEGERS`指定

```c
#define OBJ_SHARED_INTEGERS 10000
```

如以下例子：

```bash
127.0.0.1:6379[1]> set A 1
OK
127.0.0.1:6379[1]> OBJECT REFCOUNT A
(integer) 2147483647
127.0.0.1:6379[1]> set A 100001
OK
127.0.0.1:6379[1]> OBJECT REFCOUNT A
(integer) 1
```

当值为预共享池内的小整数（如 `1`）时，`OBJECT REFCOUNT` 可能显示为很大的数（共享对象被多处引用时的展示行为，与具体版本有关）。当值为池外整数（如 `100001`）时，一般为普通对象的引用计数（如 `1`）。**判断内存是否共享应结合 `OBJECT ENCODING` 与业务语义，不要单独迷信 REFCOUNT 的绝对数值。**

## 大 Key 与热 Key

### 定义与危害

**大 Key** 常见两类：

1. **Value 体积大**：例如单个 `string` 超过约 10MB，或集合类型（Hash / List / Set 等）**整体**超过约 50～100MB（阈值按团队规范与实例能力调整）。
2. **元素特别多**：例如 Hash 字段数远超万级；一般元素数超过约 5000～10000 即需警惕（同样以业务为准）。

**大 Key 的典型危害**：

1. 集群中各分片内存与访问不均。
2. **单线程执行命令耗时长**，阻塞同实例其它请求。
3. 单次读写**网络与序列化开销大**，易打满带宽。

**热 Key**：单位时间内被频繁访问、或占用大量 CPU / 带宽的少量 key。常结合 QPS、带宽占比判断，例如：

- 某分片每秒 1 万次请求中，约 3000 次落在同一 key。
- 分片总带宽 100 Mbps，其中约 80 Mbps 来自对某 Hash 的 `HGETALL` 等重命令。

> 大 Key / 热 Key 没有统一国标，需结合业务 SLA 与监控自定阈值。

### 如何发现

1. Redis 4.0+ 可用 `redis-cli --bigkeys`、`--hotkeys`（热 key 依赖 LFU 等配置，需按官方说明开启）。
2. 离线分析 RDB 可用 [redis-rdb-tools](https://github.com/sripathikrishnan/redis-rdb-tools) 等工具扫描大 key。

```bash
redis-cli -h 127.0.0.1 --bigkeys
redis-cli -h 127.0.0.1 --hotkeys
```

### 优化思路

**大 Key** 常见三类手段：

1. 进行大Key拆分

    * **String 型**：拆成多个 key，用 `MGET` 或 Pipeline 批量读，在集群上可把压力分散到不同 slot（需注意 key 设计）。
    * **集合型且必须整存整取**：设计上尽量避免；若已存在，宜迁到**对象存储 / 其它 NoSQL**，Redis 仅存索引或元数据。
    * **集合型可部分读写**：按业务分片，例如对 Hash 的 field 做哈希取模映射到多个 key（思路类似按 slot 分片），避免单 key 过大。

1. **迁出 Redis**：无法拆分又不必强依赖内存访问的数据，落到其它存储，Redis 中删除或缩短 TTL。

1. **过期与清理**：合理 TTL，避免冷数据堆积；配合**主动扫描**或后台任务处理过期/残留大 key（注意惰性删除带来的内存占用）。

**热 Key** 常见手段：

1. **读写分离 / 多副本**：读多写少时分散读流量；副本多时注意主节点复制压力。
2. **客户端或本地缓存**：二级缓存承接热点读，写路径需保证一致性策略。
3. **熔断、降级、限流**：防止热点拖垮后端与 Redis，与「缓存击穿」治理配套。

## 事务

将一组命令放在 `MULTI` 与 `EXEC` 之间批量入队，`EXEC` 时一次性执行；`DISCARD` 清空队列、不执行。

> Redis **不是**关系型数据库意义上的 ACID 事务：入队阶段只做**命令解析**；执行阶段某条命令失败时，**已执行的命令不会回滚**。

* 语法错误

```bash
127.0.0.1:6380> set mkey 111
OK
127.0.0.1:6380> MULTI
OK
127.0.0.1:6380(TX)> set mkey 222
QUEUED
127.0.0.1:6380(TX)> sett mkey 333
(error) ERR unknown command 'sett', with args beginning with: 'mkey' '333'
127.0.0.1:6380(TX)> exec
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6380> get mkey
"111"
```
由于语法错误，整个事务都未执行

* 执行错误

```bash
127.0.0.1:6380> set mkey 111
OK
127.0.0.1:6380> MULTI
OK
127.0.0.1:6380(TX)> set mkey 222
QUEUED
127.0.0.1:6380(TX)> SADD mkey 333
QUEUED
127.0.0.1:6380(TX)> exec
1) OK
2) (error) WRONGTYPE Operation against a key holding the wrong kind of value
127.0.0.1:6380> get mkey
"222"
```

对于**运行期类型错误**（如 `WRONGTYPE`），Redis **不会回滚**已成功执行的命令，开发上必须自行保证命令与 key 类型一致。

**语法错误 + 合法命令混排**时，入队阶段即失败，整个事务在 `EXEC` 时被丢弃：

```bash
# 既有语法错误，又有合法命令时，EXEC 整体不执行
127.0.0.1:6380> MULTI
OK
127.0.0.1:6380(TX)> set mkey 222
QUEUED
127.0.0.1:6380(TX)> SADD mkey 333
QUEUED
127.0.0.1:6380(TX)> sett mkey 444
(error) ERR unknown command 'sett', with args beginning with: 'mkey' '444'
127.0.0.1:6380(TX)> exec
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6380> get mkey
"111"
```

## Lua 脚本

用 `EVAL` / `EVALSHA` 在服务端原子执行一段 Lua，适合把**多步读写**合成一次往返，例如批量 `ZSCORE`。

### EVAL 参数约定

1. 第一个参数：脚本源码（一般不要包一层 `function`）。
2. 第二个参数：`KEYS` 的个数 `N`。
3. 随后 `N` 个参数：脚本中的 `KEYS[1]` … `KEYS[N]`（**Cluster 下须落在同一 slot**）。
4. 再往后为 `ARGV[1]`、`ARGV[2]` …

示例（对同一 zset 批量取分）：

```bash
127.0.0.1:6379> eval "local res={} for i,v in ipairs(ARGV) do res[i]=redis.call('ZSCORE', KEYS[1], v); end return res" 1 zset_key m1 m2 m3
```

> **Redis Cluster**：`EVAL` 涉及多个 key 时，必须保证它们在同一 **hash slot**，否则会报类似 `CROSSSLOT` / keys in same slot 错误。多 key 脚本可配合 **hash tag**（如 `{user}:1` 与 `{user}:2`）设计 key。

### Pipeline

**Pipeline**：客户端一次发出多条命令、一次读回多批结果，减少 RTT；与事务不同，**不保证**服务端原子性。

### Lua table 序列化存储

`cjson` 与 `cmsgpack` 等库可把 Lua table 编成二进制或字符串再 `SET`。**cmsgpack** 往往更省空间、更快，但**可读性**不如 JSON。

```bash
127.0.0.1:6379> EVAL "local user={};user[1]={};user[1][100]='hjhj'; return redis.pcall('SET', 'user', cmsgpack.pack(user))" 0
127.0.0.1:6379> get user
"\x91\x81d\xa4hjhj"
127.0.0.1:6379> EVAL "local user=cmsgpack.unpack(redis.call('get', 'user')); return user[1][100]" 0  # 解码后取 user[1][100]
"hjhj"
```

> 使用数字键解码 JSON 对象后必须小心。每个数字键都将存储为 Lua字符串。任何假设类型编号的后续代码都可能会中断。

## Lua 实现分布式锁（思路）

### 推荐：单条原子命令

Redis 2.6.12+ 可用 **`SET key token NX EX seconds`**（或 `PX`）一次完成「不存在则设置 + 过期」，避免 `SETNX` 与 `EXPIRE` 非原子导致的死锁。

```bash
127.0.0.1:6380> SET mylock <unique_token> NX EX 10
```

### 脚本：加锁与续期（示例）

要点：**仅当 key 不存在时占位**；若已存在且 **value 与当前持有者 token 一致**则可续期 TTL（需与业务约定一致）。

```lua
local key = KEYS[1]
local required = KEYS[2]
local ttl = tonumber(KEYS[3])
local result = redis.call('SETNX', key, required)

if result == 1 then
    redis.call('PEXPIRE', key, ttl)
else
    local value = redis.call('GET', key)
    if value == required then
        result = 1
        redis.call('PEXPIRE', key, ttl)
    end
end
return result
```

### 解锁：仅持有者可删

先 `GET` 比对 token，一致再 `DEL`，**整段应用 Lua 脚本保证原子性**（避免误删他人锁）。生产环境建议阅读官方 [Distributed locks with Redis](https://redis.io/docs/manual/patterns/distributed-locks/)（Redlock 等模式与边界条件）。

```lua
--当锁匹配的钥匙相同时才可以删除锁
local key = KEYS[1]
local required = KEYS[2]
local value = redis.call('GET', key)
if value == required then
    redis.call('DEL', key);
    return 1;
end
return 0;
```

## 持久化

Redis 持久化常见四种组合：**仅 RDB**、**仅 AOF**、**RDB + AOF**、**全关**（仅内存）。

- **RDB**：按策略或手动在时间点生成数据快照，恢复大块数据往往较快，适合备份与灾备。
- **AOF**：把写命令以 Redis 协议**追加**记录到文件，可 `rewrite` 压缩体积；重启时重放日志恢复。
- **全关**：只当纯缓存、可丢数据时使用。
- **RDB + AOF**：兼顾快照与日志；**重启时默认优先用 AOF** 恢复（通常更「完整」，以配置为准）。

### RDB 与 AOF 对比

先理解二者差异，再选策略。

- RDB的优点

    - `RDB`是一个非常紧凑的文件，保存**某一时刻**的全量数据，适合做**按时间点备份**与版本回滚。
    - `RDB`是一个紧凑的单一文件, 很方便传送到另一个远端数据中心或者亚马逊的S3（可能加密），非常适用于灾难恢复。
    - `RDB`在保存`RDB`文件时父进程唯一需要做的就是fork出一个子进程, 接下来的工作全部由子进程来做, 父进程不需要再做其他IO操作, 所以`RDB`持久化方式可以最大化redis的性能。
    - 与`AOF`相比, 在恢复大数据集的时候, `RDB`方式会更快一些.

- RDB的缺点

    - 若追求**宕机丢数据尽量少**，单独 RDB 往往不够：快照间隔之间（常见数分钟级）的写入可能丢失。
    - `RDB`需要经常fork子进程来保存数据集到硬盘上, 当数据集比较大的时候, fork的过程是非常耗时的, 可能会导致Redis在一些毫秒级内不能响应客户端的请求。如果数据集巨大并且CPU性能不是很好的情况下, 这种情况会持续1秒, `AOF`也需要fork, 但是你可以调节重写日志文件的频率来提高数据集的耐久度.

- AOF优点

    - 使用`AOF`会让你的`Redis`更加耐久:你可以使用不同的`fsync`策略：无`fsync`, 每秒`fsync`, 每次写的时候`fsync`。使用默认的每秒`fsync`策略, Redis的性能依然很好(fsync是由后台线程进行处理的,主线程会尽力处理客户端请求),一旦出现故障，你最多丢失1秒的数据。
    - AOF 为追加写，损坏时可用 `redis-check-aof` 等工具做修复（视损坏程度而定）。
    - `Redis`可以在`AOF`文件体积变得过大时，自动地在后台对`AOF`进行重写： 重写后的新`AOF`文件包含了恢复当前数据集所需的最小命令集合。 整个重写操作是绝对安全的，因为`Redis`在创建新`AOF`文件的过程中，会继续将命令追加到现有的`AOF`文件里面，即使重写过程中发生停机，现有的`AOF`文件也不会丢失。 而一旦新`AOF`文件创建完毕, `Redis`就会从旧`AOF`文件切换到新`AOF`文件，并开始对新`AOF`文件进行追加操作。
    - `AOF`文件有序地保存了对数据库执行的所有写入操作， 这些写入操作以`Redis`协议的格式保存， 因此`AOF`文件的内容非常容易被人读懂， 对文件进行分析（parse）也很轻松。 导出（export）`AOF`文件也非常简单： 举个例子， 如果你不小心执行了`FLUSHALL`命令， 但只要`AOF`文件未被重写， 那么只要停止服务器， 移除`AOF`文件末尾的`FLUSHALL`命令， 并重启`Redis`， 就可以将数据集恢复到`FLUSHALL`执行之前的状态。

- AOF缺点

    - 对于相同的数据集来说`AOF`文件的体积通常要大于`RDB`文件的体积。
    - 根据所使用的`fsync`策略，AOF 的速度可能会慢于`RDB`。 在一般情况下， 每秒`fsync`的性能依然非常高， 而关闭`fsync`可以让`AOF`的速度和`RDB`一样快， 即使在高负荷之下也是如此。 不过在处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间（latency）。

下面示意一条带过期时间的 `SET` 在 AOF 中的 **RESP** 记录形态（具体与版本、选项有关，仅供理解格式）：

```text
# 对应命令示例: SET iscod hello PXAT <毫秒时间戳>
*5
$3
SET
$5
iscod
$5
hello
$4
PXAT
$13
1687334133386
```

### AOF的fsync策略

Redis AOF 的 `fsync` 策略常见三种：

- **always**：每条写入后 `fsync`，最安全、最慢。
- **everysec**（默认）：每秒刷盘一次，故障时通常最多丢约 **1 秒**数据，性能与安全性较平衡。
- **no**：交给操作系统刷盘，最快、最不安全。

> 生产环境多采用默认的 **everysec**，在延迟与耐久之间折中。


### 如何选择

- 要求接近传统 RDBMS 的耐久：**RDB + AOF** 双开更稳。
- 可接受分钟级丢失、侧重备份体积与恢复速度：可评估**仅 RDB**。
- 仅 AOF：并非不可，但建议仍定期 **bgsave** 做快照备份；大体积恢复通常比 RDB 慢，需结合版本与工具链评估。

## 集群

### Redis Cluster 概述

Redis 自 **3.0** 起内置 **Cluster**：通过分片把数据分布到多节点，缓解单机内存、吞吐与连接压力，并配合副本做故障转移。

#### 相对单机的限制

1. **批量与多 key**：`MGET` / `MSET` / 部分脚本与事务要求 key 落在**同一 slot**，不可随意跨槽。
2. **事务**：跨 slot 时组合受限。
3. **分区粒度是 key（经哈希得到 slot）**：无法把**同一个**大 Hash/List 拆到不同节点，大 key 仍会造成单节点倾斜。
4. **逻辑库**：集群模式通常只用 **db 0**，没有单机多 `SELECT` 的习惯用法。

拓扑由多节点组成，**至少需若干主节点承载 16384 个槽位**（常见为三主，实际部署常加副本），以官方文档为准。

### 数据分片（哈希槽）

Cluster **不用**一致性哈希，而用固定 **16384 个槽（slot）**：对每个 key 计算 CRC16 后对 16384 取模得到槽号，再由节点负责槽区间。

示例（三主均分槽位，数字仅为示意）：

* 节点 A：约 0～5500
* 节点 B：约 5501～11000
* 节点 C：约 11001～16383

> 精确范围以 `CLUSTER SLOTS` 与迁移结果为准；扩缩容用 `redis-cli --cluster` 做 **reshard**。

### 节点增加后的`slot`再分配

```bash
redis-cli --cluster add-node 127.0.0.1:6383 127.0.0.1:6380 #增加新节点到集群 new_host:new_port existing_host:existing_port
redis-cli --cluster reshard 127.0.0.1:6383 # 为新节点划分slot
```


### Redis 为什么快（常见归纳）

1. **纯内存**访问（工作集在 RAM）。
2. **命令执行单线程**（避免锁竞争）；网络 I/O 采用 **epoll/kqueue** 等多路复用。（Redis 6+ 可选 **IO 线程**加速网络读写，命令仍主要在主线程执行，细节见官方说明。）
3. **渐进式 rehash**：字典扩容时把迁移分摊到多次请求，降低单次停顿。
4. **缓存时间戳**：常用时间在内存维护，减少频繁系统调用。

### 典型应用场景

| 场景 | 常见类型 |
|------|----------|
| 缓存、会话、简单 KV | String |
| 计数、限流 | String / Hash |
| 排行榜 | Sorted Set |
| 最新列表、时间线 | List / Stream |
| 去重、集合运算 | Set |
| 轻量消息队列 | List / Stream（复杂场景建议专用 MQ） |
| 分布式锁 | String + Lua / Redlock 等模式 |

### 常见性能问题与对策

1. **持久化拖慢**：可让副本承担 AOF/RDB，主节点侧重吞吐（按架构权衡）。
2. **复制延迟**：主从同机房、专线；避免跨地域同步热路径。
3. **拓扑**：复制链不宜让单主挂过多从节点；可用树状/分层减轻单主复制扇出。

### 可能导致阻塞的操作

1. **O(N) 或大范围扫描**：`KEYS *`、`HGETALL` 大 Hash、`SMEMBERS` 大 Set 等。
2. **删除大 key**：`DEL` 大集合会阻塞（可用 `UNLINK` 异步释放，视版本与支持情况而定）。
3. **`FLUSHALL` / `FLUSHDB`**。
4. **AOF 强同步**：高写入下 `appendfsync always` 或磁盘慢时放大延迟。

---

## 参考

- [如何发现和处理大 Key、热 Key（华为云）](https://support.huaweicloud.com/bestpractice-dcs/dcs-bp-0220411.html)
- [Redis 集群教程（Redis 中文网）](http://www.redis.cn/topics/cluster-tutorial)
- [《Redis 设计与实现》笔记](https://www.bookstack.cn/read/Redisbook/2d294542c86f1acf.md)
- [Distributed locks with Redis（官方）](https://redis.io/docs/manual/patterns/distributed-locks/)


## golang

### 值类型&引用类型传递

引用类型传递: `map`, `slice`, `interface`, `chan`, `pointer`

值类型传递: `array`, `struct` ,`int`, `float`, `bool`, `string`

> 一个简单的判断就是引用类型可以初始化为`nil`

*make可以创建哪些*

```go
var ch, sl, ma = make(chan int), make([]int, 0), make(map[int]int)
```

* make, new 的区别

1. make 只能创建三种类型，new 可以创建任意类型
2. make返回的是Type对象，new 返回的是Type指针

### 控制并发控制的三种模式

1. 通过`chan`进行控制
1. 通过`Waitgroup`进行控制
1. 通过`Context`上下文进行控制

### chan

* chan应用到哪些场景?

1. 协程见的通信
1. 控制并发数
1. 定时任务 time.after

* chan可以传入nil吗？

当chan定义为引用类型时，可以传入nil, 非引用类型如`int`、`string`、`array`, `struct`不行


### RPC

gRPC是golang实现的RPC框架

RPC是一种远程服务调用方案，RPC负责底层的通信协议和序列化方式。服务调用者可以想调用本地函数一样调用远程服务。
gRPC对接口与严格的约束，安全性高，且更适合高并发场景。
gRPC有明确的接口规范，
RCP效率更高，RPC使用自定义的TCP协议，可以让请求报文体积更小，提高传输效率

rest，是网络中的clinet和server的交互形式

### 读写锁

golang的`sync`包提供了 互斥锁`sync.Mutex`和读写锁`sync.RWMutex`两种锁

`sync.Mutex`互斥锁，既不可同时运行。互斥锁有两个方法`Lock`和`Unlock`。通常在加锁有采用`defer`语句解锁。

`sync.RWMutex`读写锁，可以理解为`多读单写锁`。`读`是运行同时运行的，`写`是互斥的。一般有四种情况

读锁之间不互斥，没有写锁的情况下，读锁是无阻塞的，多个协程可以同时获得读锁。
写锁之间是互斥的，存在写锁，其他写锁阻塞。
写锁与读锁是互斥的，如果存在读锁，写锁阻塞，如果存在写锁，读锁阻塞。


sync.map是golang官方包实现的并发安全的map结构

### 协程和线程和进程的区别

* 进程

进程是系统资源分配和调度的最小单位，每个进程有自己的内存空间，不同进程之前通讯是通过进程通讯来完成

* 线程

线程是进程的一个实体，他是比进程更小的能独立运行的基本单位。线程间通讯最要通过共享内存，资源开销较小

* 协程

协程是一种用户态的轻量级线程，协程的调度是通过用户来进行控制的。协程拥有自己的寄存器上下文和栈（P）


线程是进程的一个实体

### GMP

* G: goroutine 调度实体, 既用户代码,   # 在本地队列中不断切换执行
* M: machine 内核线程, 既系统线程, 负责代码执行
* P: processor 逻辑处理器, 保存了调度上下文。也可以理解为局部的一个调度器。P的数量由`runtime.GOMAXPROCS`控制,

每个p都有一个runqueue队列。p负责goroutine调度到`M线程`上运行。

除此之外呢，golang还维护一个全局的runqueue队列。


当某个p的本地runqueue队列运行完。首先 `p`就会从全局队列当中获取新的`goroutine`.

如果全局队列没有`goroutine`.

那么会从其它的`P`本地队列中偷一些`goroutine`(一半)


当某一个进程`M0`被阻塞时，P会找到一个新的`M1`进程上进行运行。
当被阻塞的`M0`返回`syscall`(既可以运行不阻塞了) `M0`会尝试获取一个`P`来继续运行。如果没有获取`P`，那么会将`MO`阻塞的`G`放入全局`runqueue`

### csp

CSP 模型是*以通信的方式来共享内存*，不同于传统的多线程通过共享内存来通信。

主要用于描述两个独立的并发实体, 通过共享的通讯 channel (管道)进行通信的并发模型。


### 如何退出`goroutine`

`goroutine`的退出一般有两种模式： 1.超时模式 2.关闭chan

超时模式是使用`time.After`启动了一个异步的定时器，返回一个 channel，当超过指定的时间后，该 channel 将会接受到信号

2，通过一个chan的关闭通知，进行`goroutine`的关闭

### 什么情况下死锁，如何避免？

单个`goroutine`自读自写一个没有缓冲能力的`chan`

range `chan`时要注意关闭，如果不关闭一直阻塞


### 垃圾回收

#### 垃圾回收的三种算法

1. 引用计数

每个对象维护一个引用计数, 当被引用对象被创建或被赋值给其他对象时引用计数自动加 +1；如果这个对象被销毁，则计数 -1 ，当计数为 0 时，回收该对象。 [redis内存回收](https://iscod.github.io/#/nosql/redis?id=%e5%86%85%e5%ad%98%e5%9b%9e%e6%94%b6)

redis就是采用的该种方法
2. 标记清除, 从根变量开始遍历所有引用的对象，引用的对象标记“被引用”，没有被标记的则进行回收
3. 分代收集

golang的垃圾回收是通过`GC`来实现的。

GC采用的理念是`三色标记`. `三色标记`可以理解成将一个内存标记为三种状态

1，白色对象 — 潜在的垃圾，其内存可能会被垃圾收集器回收
2，黑色对象 — 活跃的对象，包括不存在任何引用外部指针的对象 以及从根对象可达的对象
3，灰色对象 — 活跃的对象，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象

垃圾回收的可以理解为三个步骤

1. 从灰色对象的集合中选择一个灰色对象并将其标记成黑色；
2. 将黑色对象指向的所有对象都标记成灰色，保证该对象和被该对象引用的对象都不会被回收；
3. 重复上述两个步骤直到不存在灰色对象；

当三色的标记清除的标记阶段结束之后，应用程序的堆中就不存在任何的灰色对象，我们只能看到黑色的存活对象以及白色的垃圾对象，垃圾收集器可以回收这些白色的垃圾

##### 垃圾回收演进

* V1.0 版本完全串行的标记和清除，会暂停整个程序`STW`
* V1.5 引入三色标记清扫的并发处理
* V1.8 使用混合屏障(插入和删除屏障)
* V1.9 彻底移除暂停整个程序的重新扫描栈过程

### 反射

反射是通过`reflect`(ruifulekede)包来实现。它可以让程序操作不同类型的对象
`reflect`包提供了两个重要的接口`reflect.TypeOf` 和 `reflect.ValueOf`。TypeOf用来获取类型信息。ValueOf用来获取数据

实现了运行时的反射能力，能够让程序操作不同类型的对象1。反射包中有两对非常重要的函数和类型，reflect.TypeOf 能获取类型信息，reflect.ValueOf 能获取数据的运行时表示

```
type User struct {
Name string
Age  int
}

func main() {
u := User{Name: "ning", Age: 12}
tp := reflect.TypeOf(u)
va := reflect.ValueOf(u)
for i := 0; i < tp.NumField(); i++ {
fmt.Println(tp.Field(i).Name, "：", va.FieldByName(tp.Field(i).Name))
}
}

```


### 高并发场景

高并发是一个系统工程主要是通过四个方面来着优化

1. 业务层面
1. 可以通过流程设计减少热点数据的访问量，比如做预约,增加实名认证等条件才可参加活动
1. 比如可以增加访问验证码等步骤，延迟用户核心数据访问频率，同时可以放置抢购脚本

1. 服务器硬件设施方面
1. 系统分离，可以尝试将高并发场景单独做成一个server服务，单独部署减少对其他系统的影响。并且可以专门做相应的优化，比如请求较高可以做动态的伸缩服务器硬件。也可以进行相应的流量均衡
1. 可以做一些静态资源的缓存CDN。减少对核心服务器的访问。

1. 代码逻辑层面
1. 尽可能的减少数据操作（读写数据库）。
1. 可以采用消息队列做中间层，将并发的访问转换为顺序的请求访问
1. golang的本身支持高并发，可以结合`goroutine`进行逻辑的并发处理
1. 数据库端
1. 读取数据进行Cache缓存。可以使用Memcache、redis进行数据的提前缓存。同时要注意缓存的命中率问题。
在数据更新时，尽可能的及时刷新缓存，保证命中率
1. 可以采用redis结合lua脚本保证数据的一致性
1. 可以采用数据库事务来进行原子性操作
1. 另外可以在应用程序层面, 通过操作后再次查询结果的方式，保证执行的结果
高并发的原则是在保证数据一致性的前提下尽可能的提高响应速度，保证可用性

## redis

### redis有哪些特性/redis为什么快？

1. 纯内存的数据访问
1. 单线程避免上下文切换（io是多路复用，命令还是单线程）
1. `渐进式Rehash`, 缓存系统时间戳

### redis适用哪些场景？

1. 缓存
1. 分布式SESSION
1. 计数器
1. 分布式锁
1. 排行榜
1. 消息队列

### redis有哪些高级功能？

1. lua
1. Pipeline
1. 事务（MULTI）
1. 慢日志

```bash
# 设置多少时间的日志记录到慢日志（1000µs）
127.0.0.1:6380> CONFIG SET slowlog-log-slower-than 1000
```

### redis如何实现分布式锁/redis实现分布式锁需要注意哪些？

1. 死锁(setnx)
1. 锁有效期的续期(witch dog)
1. 锁的释放(根据钥匙进行释放)

### redis如何实现消息队列/redis实现消息队列需要注意哪些？

1. 基于List的`LPUSH`+`BRPOP`实现

优点是足够简单，消费消息延迟几乎为0，但是要处理闲置连接问题。

当没有消息时，线程会阻塞，`BRPOP`客户端闲置，闲置太久，服务器一般会主动断开连接，减少闲置资源占用，这个时候brpop和blpop会抛出异常，所以在编写客户端消费时，要做好异常捕获，然后进行重试。

```
# 生产者发送消息
127.0.0.1:6382> LPUSH lmq1 mess1 mess2 mess3
# 消费者，BRPOP需要提供一个timeout时间,这里是 10s，当没有消息时，会超时断开
127.0.0.1:6382> BRPOP lm1 10
1) "lm1"
2) "mess1"
```

1. sorted-set实现

1. PUB/SUB，订阅发布

优点：

典型的广播模式，一个消息可以发布到多个消费者，多信道订阅，消费者可以同时订阅多个信道，从而接受多类消息，消息即使发送，消息不回等待消费者去读取，消费者会自动接收到信道发布的消息

缺点：

消息一旦发布，如果消费者不在线，这消息丢失，不能寻回。不能保证每个消费者接受到的时间是一致的。同时若消费者客户端消息积压到一定程度，会被强制断开，导致消息丢失。

可见该模式不适合做消息存储，消息积压类的业务。适合处理广播，即时通讯和即时反馈的业务

```bash
127.0.0.1:6382> PUBLISH chan1 mess1
# 客户端订阅两个信道
127.0.0.1:6382> SUBSCRIBE chan1 chan2
```

1. Stream

`Stream`是Redis 5.0版本引入的一个新的数据类型。它提供了一组允许消费者以阻塞的方式等待生产者向`Stream`发送消息。此外它还提供了消费组的概念。注意它并未提供像`kafka`中分区的概念

```bash
# 生产者添加消息
127.0.0.1:6383> XADD mystream * filed1 val1 filed2 val2
# 读取10条消息
127.0.0.1:6383> XREAD count 10 STREAMS mystream 0
# 读取所有消息
127.0.0.1:6383> XREAD count 10 STREAMS mystream 0
# 添加消息组
127.0.0.1:6383> XGROUP CREATE mystream mgroup1 $
# 消息组读取消息
127.0.0.1:6383> XREADGROUP GROUP mgroup1 consumer1 count 2 STREAMS mystream >
```

* Stream 消息太多怎么办？

如果Stream中消息累计太多，Stream链表就变长，内存有可能超出，而`XDEL`只是给消息做了标识，并未删除数据。那如何控制？

`XADD`指令提供了一个定长长度`maxlen`, 可以确保消息超过指定长度时，删除老的消息。因此可能会出现消息丢失

* 消息如果忘记`ACK`会怎样/ 为什么要及时进行`ACK`?

Stream在消息结构中保存了正在处理中的id列表（PEL），如果消费者收到消息处理完而没有回复`ACK`,那么就会导致`PEL`表不断增长，如果消费组很多，那么`PEL`就会占用很多内存。所以消息要及时进行`ACK`


* 死信消息如何处理？

某个消息不能被消费者处理，也就是不能被`XACK`,长时间处于`Pending`列表中，即使被反复的转移至其他消费者也不能被处理，该类消息的`delivery count`（XPENDING 可以查询该次数）就会不断累加，当累加我们预设的累加值时，我们就认为是坏消息。

`XDEL` + `XACK`进行删除

### 如何提高缓存的命中率？

1. 数据预加载（数据提前加载到redis）
1. 增加缓存的存储空间，提高缓存数据量
1. 调整缓存类型，提高业务缓存利用率
1. 提升缓存的刷新频次，（可以通过bin log/mq 及时通知到缓存服务，进行数据更新）

### 如何保证缓存与数据库双写时的一致性？

要保证数据与缓存双写时的数据一致性，可以采取以下几种常见的策略：

1. 写数据时先更新缓存：在写入数据之前，先更新缓存中的对应数据。这样可以确保缓存中的数据与数据库中的数据保持一致。然后再写入数据库，确保数据库中的数据也是最新的。这种策略需要保证缓存的更新操作成功，否则可能会导致数据不一致。

2. 更新缓存时先删除缓存：在更新数据库数据时，先删除缓存中的对应数据，然后再更新数据库。当下次查询该数据时，发现缓存不存在，就会从数据库中重新加载最新的数据，并将其写入缓存。这样可以确保缓存中的数据与数据库中的数据保持一致。

3. 异步更新缓存：在写入数据库后，异步地更新缓存。即先将数据写入数据库，然后再更新缓存。这种方式可以提高写入数据库的性能，并且可以容忍一定程度的缓存更新延迟。但需要注意的是，异步更新缓存可能会导致一段时间内缓存中的数据与数据库中的数据不一致。

4. 使用事务保证一致性：在支持事务的数据库中，可以使用事务来保证数据与缓存的一致性。将数据库更新和缓存更新操作放在同一个事务中，要么同时成功，要么同时失败回滚。这样可以确保数据与缓存的一致性。但需要注意的是，事务的使用会增加数据库的负载和延迟。

综合考虑具体业务场景和需求，选择适合的策略来保证数据与缓存双写时的一致性。同时，还需要注意异常情况下的处理，例如缓存更新失败、数据库更新失败等，需要有相应的处理机制来保证数据的一致性。


只要是使用了缓存与数据库两种数据源，就一定会存在数据同步的问题

数据更新时缓存更新的四种策略：

1. 先更新缓存后更新数据：这种策略在更新缓存之后，如果数据操作失败，可能导致缓存中的数据与数据库中的数据不一致。这种情况下，可以选择回滚缓存的更新操作，或者通过其他方式进行补偿操作，以保证数据一致性。
1. 先更新数据后更新缓存：如果在更新数据库之后，更新缓存操作失败，可能导致缓存中的数据与数据库中的数据不一致。这种情况下，可以选择回滚数据库的更新操作，或者通过其他方式进行补偿操作，以保证数据一致性
1. 先删除缓存后更新数据：如果在删除缓存之后，更新数据库操作失败，可能导致缓存中的数据被删除，但数据库中的数据没有被更新。
1. 先更新数据后删除缓存：


1. 先更新缓存后更新DB(一般不考虑，如果DB出错导致数据不一致)
2. 先更新DB后更新缓存(一般不考虑，如果缓存更新失败导致数据不一致)
3. 删除缓存后更新DB

这种策略也会出现一下问题：

A请求先删除了缓存，还未及时更新数据库。而此时B进行了查询, 将老数据更新到了缓存, 就造成数据不一致。
该问题一般可采用`延迟双删`解决，延迟双删策略是：

1. 先删除缓存
2. 更新数据库
3. 休眠1s后，再次删除缓存

这样可以将1s内的缓存脏数据进行删除，休眠时间可以根据自己的业务平均耗时进行评估一般在几百毫秒

4. 先更新DB后删除缓存（一般情况选择该种类）

这种策略保证只有在查询速度比删除缓存慢的情况下才会出现数据不一致的情况，而这种情况来说相对少很多。
而且可以结合采用延时删除进一步保证数据的一致性。

### redis的持久化有哪些方式和区别？

1. rdb
1. aof
1. reb+aof

aof记录命令，rdb备份数据快照

### 缓存穿透？

通过bitmap设置布隆过滤器，在缓存前进行拦截过滤（类似于二级缓存）

### 缓存雪崩？

缓存雪崩是指在缓存系统中，大量的缓存同时失效或过期，导致请求直接打到数据库或后端系统，从而引起数据库或后端系统压力瞬间增大，甚至导致系统崩溃的现象。

为了避免缓存缓存雪崩，可以采用以下机制策略

1. 合理设置缓存过期时间：可以通过设置随机的过期时间，或过期时间段来分散缓存失效的时间点，避免大量的缓存在同一时间失效
1. 缓存数据预热：在系统启动或低峰期，提前加载热点数据，避免在高峰时期大量请求到来，缓存数据还未加载导致的缓存雪崩
1. 使用多级缓存：引入多级缓存架构，将缓存分为不同层级，可以使用本地缓存为一级缓存，将分布式缓存（如Redis）作为二级缓存。这样即使一级缓存失效，二级缓存应然可以提供缓存数据

redis缓存的过期时间随机设置

### redis如何解决key冲突？

1. 业务前缀（Prefix）, 可以在Key的设计中为每个业务或模块添加特定的前缀。通过为Key添加业务前缀，可以避免不同业务之间的Key冲突。例如，可以使用"business:key"的形式来命名Key。
1. 业务隔离
1. 采用不同的集群
1. 单机系统，可以采用不同的数据库(select)

## kafka

### kafka名词解释

1. Group: 若干个消费者组成一个分组，一个topic可以有多个分组。topic分组内，每一个分区只能有一个消费。如果需要实现广播，只要每个消费者有一个独立的分组即可实现
1. Topic: 主题，队列
1. Partition: 一个topic可以分为多个partition， 每个`partition`是顺序的，但是kafka只保证同一个partition的消息顺序，不保证一个topic的整体顺序。因此如果想要保证顺序只能设置一个partition
1. OFFSET: 偏移量
1. Zookeeper

### kafka消息丢失场景及解决方案？

#### 消息发送

1. 设置ack=0，生产者发送消息，不确认消息是否发送成功，消费发生失败则消息丢失
2. 设置ack=1，生产者发送消息，只确认leader节点发生成功。leader未同步到follwer节点时，leader故障则消失丢失
3. unclean.leader.election.enable设置为true(默认false),允许选举ISR列表以外的副本为leader,

解决方案：ack=all/-1/2/3等多个从节点 ,unclean.leader.election.enable=false

#### 消费者

先commit后处理消息。如果消息处理异常，但是offset已经提交，这条消息对于消费者来说已丢失了

解决方案：先进行消息处理，后进行commit。同时做好业务处理的幂等性


### kafka使用哪些场景/为什么要使用kafka？

1. 应用解耦
1. 日志收集
1. 流量削峰
1. 用户行为记录

### kafka存储是如何实现的？

### kafka的分区如何使用？

### kafka如何保证消息顺序

1. 只有一个生产者(保证生产者线程安全)
1. kafka的topic必须设置一个分区（保证kafka内部消息发送接收顺序）
1. 只有一个消费者（保证消费者接收消息顺序）

### kafka为什么那么快？

1. 零拷贝

    kafka通过零拷贝技术，避免了数据在内核空间和用户空间的拷贝(采用发送文件描述符, 相对于传统4次拷贝，减少了两次CPU拷贝)，减少了CPU和内存开销。

1. 批量处理

    kafka支持批量发送和消费消息，可以将多个消息一起发送或消费，减少了网络传输和IO操作次数，提高了性能

1. 文件顺序读写

    kafka的持久化方案，使用顺序读写磁盘的方式，减少了随机读写磁盘的开销

1. 分布和并行处理

    kafka可将消息分为多个分区，并行处理不同分区的消息，从而实现了水平扩展和并行处理，提高了系统的吞吐量和并发性能

### kafka有哪些参数?（答出5-10个）

回答该问题主要分四个方面：`系统配置`，`主题参数`，`生产者参数`，`消费者参数`。

1. 系统配置参数

* broker.id

在集群下的唯一id,要求是整数。如果服务器ip发生变化，而broker.id没有变化，则不影响consumers消费情况

* listeners

监听列表，逗号分割，如果hostname为`0.0.0.0`则绑定所有的网卡地址，如果为空，则绑定默认网卡

* zookeeper.connect

zookeeper集群地址，多个采用逗号分隔

* auto.create.topics.enable=true

是否运行自动创建主题，如果设置为true, 那么product,consumer,fetach一个不存在的主题时，会自动创建。一般处于安全考虑会设置为false

1. 主题配置

* num.partitions

新建主题的分区个数。该参数根据消费者处理能力和数量进行确定，分区数应大于消费者数量（1，便于后面扩充消费者数量，2，每一个分区至少一个消费者，分区数大于消费者数量时，消费者会平分分区[分区策略]([https://iscod.github.io/#/amqp/kafka?id=partition]))

* message.max.bytes

消息最大字节数，生产者消费者应该设置一致。该值一般与`fetch.message.max.bytes`配合设置。

1. 生产者配置

* request.timeout.ms

生产者请求的确认超时时间

1. 消费者配置

* enable.auto.commit

是否默认提交，即消费者受到消息后是否会后台自动提交`offsets`，默认是true

* auto.commit.interval.ms

使用者偏移量的提交频率

## RocketMQ

### RocketMQ 的持久化机制？

### RocketMQ 怎么实现消息顺序？

### RocketMQ 事务消息原理？

## MySQL

### 隔离级别

1. 读未提交(Read Uncommitted)

    最低级别的隔离级别，事务可以读取其他事务未提交的数据。这种隔离级别可能导致脏读（Dirty Read），即读取到未提交的数据。

1. 提已交读（Read Committed）

    在事务提交后，其他事务才能读取其修改的数据。这种隔离级别解决了脏读问题，但可能导致不可重复读（Non-Repeatable Read），即同一个事务内多次读取同一数据时，得到的结果不一致（别的事务完成提交修改了数据）。

1. 可重复读（Repeatable Read）

    确保同一事物内多次读取同一数据时，得到的结果是一致的。其他事务在该事务提交前不能修改相关数据。这种隔离级别避免了不可重复读的，但可能导致幻读（Phantom Read），即同一个事物内多次查询时，得到的结果集不同（别的事务新增了查询结果集的数据）。
    MySql默认是该级别的隔离，但是InnoDB通过多版本并发控制（MVCC）解决可幻读问题

1. 串行化（Serializable）

    最高级别的隔离级别，要求事务串行执行，确保了数据的完全隔离。但由于串行化执行的特点，导致并发性能下降

### MySQL有哪些存储引擎

`InnoDB`, `MyISAM`, `CSV`等

关键核心：MyISAM不支持事务，InnoDB支持事务。MyISAM支持表锁，InnoDB支持表和行锁。 InnoDB是`MySQL5.1`版本后的默认存储引擎

### MySQL的索引有哪些？

该问题的核心点是：MySQL的索引是存储引擎实现的，不同的存储引擎索引的工作方式不同。也不是所有的存储引擎都支持所有类型的索引

1. unique(唯一索引)
1. index（b+tree）索引
1. 多列/复合索引
1. 全文索引（FULLTEXT）

### explain用过吗？有哪些主要字段

`type`, `key`, `extra`, `rows`

`type` 常见有 const (查询主键索引，属于精确查找), eq_ref (使用了唯一索引，属于精确查找), ref（使用到了非唯一索引）, range（查找某个索引的部分索引，属于范围查询）, index (查找索引树), all (不使用任何索引，全表扫描)

`extra` 常见有 "Using filesort", "Using index", Using where"等提示我们是否使用索引，是否使用了临时表等

### 哪些情况下索引会失效？

1. 不使用索引列：如果查询条件中没有使用到索引列，MySQL将无法使用索引进行查询优化，而是需要进行全表扫描。
1. 使用LIKE操作符时，通配符（%）在开头时，MySQL无法使用索引进行匹配
1. 存在OR条件时，如果其中一个条件可以使用索引，而另一条件无法使用索引，MySQL将无法使用索引优化整个查询
1. 对索引列进行了函数操作、类型转换或者使用了表达式进行计算时，MySQL无法直接使用索引
1. MySQL优化器认为全表扫描比使用索引快时，也会失效，例如：当查询条件中使用了IN操作符，并且IN列表中的值数量过多时

> 使用联合索引，查询条件没有使用联合索引的第一列，索引仍然可能起作用，但是只能部分利用索引，而不是完全利用。

### 什么是写失效？

InnoDB的page和操作系统的页大小不一致，InnoDB的页大小是16kb,操作系统是4kb, InnoDB的页写入磁盘时，一个页需要分四次。

如果在存储引擎写入磁盘时，发生了宕机，那么就有可能未全部写入16kb，这种情况叫部分写入失效（partial page write）,可能导致数据丢失。

* InnoDB如何解决写失效？

InnoDB通过写缓冲区和重做日志（redo log）保证数据的安全性。

当发生写操作时，InnoDB引擎会首先将修改的数据写入重做日志中，然后再将数据写入写缓冲区。
重做日志的写入是顺序的，而且是磁盘IO密集型的操作。

正常情况下，InnoDB引擎会将写缓冲区中的数据定期刷新到磁盘，这个操作称为刷新（flush）。
刷新通常由一个后台线程或者刷新日志（flush log）操作来执行。
在刷新之前，InnoDB会将写缓冲区的数据写入到重做日志中，以便在系统恢复后能够重新将数据写入磁盘。
这样即使在异常发生时，写缓冲区中的数据也能够通过重做日志进行恢复。

在系统发生异常时，通过重做日志可以回放这些操作，将写缓冲区中的数据重新写入磁盘，从而恢复数据的一致性。

```bash
mysql> show variables like "%doublewrite%"; # 查询doublewrite是否开启，默认启用双写缓冲区
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| innodb_doublewrite            | ON    |
| innodb_doublewrite_files      | 2     |
+-------------------------------+-------+
```

### 什么是行溢出？

在MySQL中，行溢出（Row Overflow）指的是当一个表的行的数据大小超出了行大小限制时，部分或全部数据将被存储在行溢出页中。

MySQL的InnoDB存储引擎使用了固定大小的页（通常为16KB）进行与磁盘和内存的数据交互，每一行的数据都应该能够适应页的大小。
通常情况下，一行的数据应该被存储在主数据页中，但是如果一行的数据大小超过了页的限制，MySQL就会将部分或全部数据存储在行溢出页中。

行溢出发生的情况通常是由于以下原因之一：

1. 列的数据类型为可变长度类型，例如VARCHAR、TEXT、BLOB等，并且存储的数据超过了定义的长度。
2. 表定义了很多列，导致行的总大小超出了页的限制。

当发生行溢出时，MySQL会将溢出的数据存储在`行溢出页`中，并在主数据页中存储一个指向行溢出页的引用。
这样，在查询时，MySQL会根据需要从主数据页和行溢出页中读取数据来获取完整的行数据。

需要注意的是，行溢出可能会导致查询性能的下降，因为在读取行的数据时需要额外的IO操作来获取行溢出页中的数据。
为了避免行溢出的情况，可以考虑调整表的设计，例如使用合适的数据类型和优化列的数量，以确保行的数据大小适应页的限制。

### MySQL有哪些日志？

1. 错误日志
2. bin log日志
3. 慢日志
4. 查询日志

```bash
mysql> set global slow_query_log = 1; # 开启慢日志
mysql> set long_query_time = 2; # 时间临界值2s,可以设置为0记录所有查询
mysql> show variables like "slow_query%";
+---------------------+--------------------------------------+
| Variable_name       | Value                                |
+---------------------+--------------------------------------+
| slow_query_log      | ON                                   |
| slow_query_log_file | /var/lib/mysql/d3e3db712483-slow.log |
+---------------------+--------------------------------------+
2 rows in set (0.02 sec)
```

### Mysql内存相关参数

### Mysql内部支持缓存查询吗？

Mysql5.7支持内存缓存、Mysql8.0之后废弃

Mysql缓存的限制

1. Mysql基本没有灵活的管理缓存失效和生效，尤其对于频繁更新的数据
1. SQL语句必须完全一致才能命中缓存
1. SQL语句有触发器, 自定义函数时，Mysql缓存不生效
1. Mysql缓存在分库分表环境下是不起作用的
1. 在表结构或数据发生变化时，基于该表的缓存立即全部失效

替代方案：应用层使用redis, memcached等

#### InnoDB缓存池（Buffer pool)

`Buffer pool`主要是用来缓存表数据、索引数据、自适应哈希索引、插入缓冲、锁、以及其他内部数据。
InnoDB还使用缓冲池帮助延迟写入（能合并多个写入操作，然后顺序写会）减少磁盘IO操作，提升效率。
`Buffer pool`默认是 128M, 以page页为单位, 每个page页大小是16k。

一个大的日志缓存区允许大量的事务在提交之前不写入日志到磁盘。通过设置Buffer pool参数大小，可以大量减少磁盘的I/O次数。（一般建议在数据库服务器上，可以将缓冲池大小设置为服务器物理内存的60%~80%）

```bash
mysql> show variables like "%buffer_pool%"; #查看buffer pool大小
mysql> set global innodb_buffer_pool_size = 3221225472; # 调整buffer pool大小
```

* 如何评估`innodb_buffer_pool_size`设置是否合理？

通过分析InnoDB缓冲池的命中率来验证是否合理，一般命中率低于90%时，可考虑适当增加。

`命中率=Innodb_buffer_pool_read_requests/(Innodb_buffer_pool_read_requests+Innodb_buffer_pool_reads)`

```bash
mysql> show status  like "%buffer_pool_read%";# 查询缓存命中与非命中次数
+---------------------------------------+-------------+
| Variable_name                         | Value       |
+---------------------------------------+-------------+
| Innodb_buffer_pool_read_requests      | 32786546234 |
| Innodb_buffer_pool_reads              | 1096914     |
+---------------------------------------+-------------+
```

#### Page参数

```bash
mysql> show status like "%page_size%";# 查询page_size大小，该值只能通过配置更改，且一般不会设置，使用默认16kb
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| Innodb_page_size | 16384 |
+------------------+-------+
mysql> show status like "%pool_page%";# 查询pool_page相关指标
+----------------------------------+---------+
| Variable_name                    | Value   |
+----------------------------------+---------+
| Innodb_buffer_pool_pages_data    | 7143    |
| Innodb_buffer_pool_pages_dirty   | 0       |
| Innodb_buffer_pool_pages_flushed | 3234771 |
| Innodb_buffer_pool_pages_free    | 1024    |
| Innodb_buffer_pool_pages_misc    | 25      |
| Innodb_buffer_pool_pages_total   | 8192    |
+----------------------------------+---------+
```

### MySQL的主键与唯一索引有哪些区别

1. 一个表只能有一个主键，而唯一索引可以有多个
1. 主键可以作为外键使用，而唯一索引不行
1. 主键必定是唯一索引，而唯一索引不一定是主键

主键是用来唯一标识表中每一行数据的，必须是唯一的且不能为空值，一个表只能有一个主键。
唯一索引用于确保表中某列或多列的数据的唯一性，可以有多个唯一索引，可以包含空值。
主键在数据库中自动创建索引，而唯一索引需要手动创建。

### MySQL慢查询优化？

1. 优先选择优化高并发，因为高并发的SQL带来的问题后果更严重，且仅仅一点的优化整体的性能就会明显提升
1. 明确优化目标（根据业务和当前数据库状态，以优化的结果给用户一个好的体验，而不一定是消耗的资源最少）
1. explain分析sql语句
1. 永远用小的结果集驱动大的结果集（小的数据集驱动大的结果集，减少内层表读取次数）
1. 尽可能的在索引中完成排序
1. 只使用最有效的过滤条件
1. 只获取自己需要的列
1. 尽量避免复杂的join和子查询
1. 合理设计使用索引（唯一性太差的字段不适合创建索引，更新频繁的字段不适合创建索引，不会出现咋where字句中的不适合创建索引）

### 如何进行JOIN优化？

1. 在不影响查询结果的情况下，驱动表尽量选择结果集最小的那张表（减少外层循环次数）
1. 为匹配的条件增加索引（减少内层表循环的次数）
1. 增加 join buffer size的大小（一次缓存的数据越多，内层循环的扫表次数减少）
1. 减少不必要的字段查询（字段越少，join buffer所缓存的数据越多）

### Mysql为什么使用B+Tree?

* B-Tree的特征

1. 调整B-Tree 的阶数（m），可以降低树的高度，降低查询节点次数
1. 数据记录在所以节点，当数据较大时，相同存储空间，节点保存的key值就会变少，导致B树层数变高

B+Tree是在B-Tree的基础上的一种优化，使其更适合索引结构

* B+Tree的特征

1. 非叶子节点只存储键值信息（根节点和支节点只保存索引，这样节点相同大小的情况下可以保存更多的关键字）
1. 所有叶子节点都有一个链指针用于查找上下节点（这样B+Tree的扫表能力更新，只需要扫描叶子节点即可，而B-Trees需要扫描整个树）
1. 数据记录都存放到叶子节点中


## k8s

k8s是google开发的

## 其它

### 幂等有哪些技术方案?

1. 查询请求

查询`select`是天然幂等，在数控不发生变化时，查询总是相同

1. 删除请求

删除操作也是幂等，区别是删除不存在的数据时，返回可能不同，但对业务没有太多影响

1. 唯一索引（数据唯一）

防止新增脏数据，对需要限制的数据设置唯一索引，保证数据中只存在一份，这样即使有多于请求，也能保证数据仅生成一份。

1. token机制（防止页面重复提交）

数据提交前, 先向服务器申请token, token放到Redis，或内存中，并设置有效期。
客户端拿到token, 提交请求，后台校验token, 同时删除token, 生产新的token返回客户端, 以供下次访问提交

token的特点：需要申请，一次有效性，可以限流

> Redis根据删除操作命令是否返回成功，验证token是否存在

1. traceId(操作标识)

根据用户的动作标识+设备ID+时间等 生产一个`traceId`, 用户提交后，后台判断`traceId`是否已处理，进而判断否是是重复提交

### 常见的设计模式有哪些？使用过哪些设计模式

设计模式总共有23个，按照他们的用途分为三类：创建型、结构型、行为型

#### 创建型

创建型提供创建对象的机制，提高代码的灵活性和复用性。常使用的有：单例模式、工厂模式、建造者模式。不常用的有：原型模式

#### 结构型

结构型主要是将对象和类封装成较大的结构，并同时保持结构的灵活性。常使用的有：代理模式、桥接模式、装饰模式、适配器模式。不常使用的有：门面模式、组合模式、享元模式

#### 行为型

行为型主要是：负责对象的间的高效沟通和负责传递委派。常用有：观察者模式、模版模式、策略模式、职责链模式、迭代器模式、状态模式。不常见的有：访问者模式、备忘录模式、命令模式、解释器模式、中介模式

*观察者模式*

观察者模式有两个角色：观察者和被观察者，观察者可以有多个。被观察者一般实现三个接口：添加观察者，删除观察者、通知

观察者常见的场景有：

1. 当一个对象发生改变时，需要改变其它对象时。例如，商品库存发生变化时，需要通知商品详情页，购物车等系统改变数量
1. 当一个对象发生改变时，只需要发送通知，而不关心接受者是谁。例如，订阅微信公众号文章，发送者通过公众号发送，发送者并不知道哪些用户订阅了公众号。
1. 需要创建一个链式触发机制时。比如一个系统中创建一个触发链，A对象的行为将影响B，B对象的行为将影响C...这样通过观察者模式比较好实现
1. 微博或微信朋友圈发送场景。这是观察者模型的典型应用场景，一个人发微博或朋友圈，只是关联的朋友都会接受到通知，一旦取消关注，此人以后不会受到通知


* 参考
* [mysql索引](https://cloud.tencent.com/developer/article/1785124)






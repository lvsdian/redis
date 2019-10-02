### 基础

- redis使用**单线程架构**和**IO多路复用模型**实现高性能的内存数据库服务。一条命令从客户端到达服务端不会立即被执行，所有命令都会进入一个队列中，然后逐个被执行。单线程模型还会达到每秒万级别的处理能力原因：

  1.  纯内存访问。
  2. 非阻塞IO。（使用epoll作为IO多路复用技术的实现，在加上Redis自身的事件处理模型将epoll中的连接、读写、关闭都转为事件，不在IO上浪费过多事件）
  3. 单线程避免的线程切换和竞态产生的消耗。

  但单线程模型中，如果某个命令执行时间过长，会阻塞其他命令，所以Redsi是面向快速执行场景的数据库。

- Redis5种数据结构：
  - string
    - 内部编码
      1. int（8字节长整型）
      2. embstr（<=39字节的字符串）
      3. raw（>39字节的字符串）
  - hash
    - 内部编码
      1. ziplist（ziplist更紧凑，比hashtable节省内存，当元素个数小于`hash-max-ziplist-entries` (默认512个)&& 所有的值小于`hash-max-ziplist-value`(默认64字节)时用ziplist）
      2. hashtable（hashtable读写时间复杂度为O(1)，比ziplist效率高）
  - list
    - 内部编码
      1. ziplist （当元素个数小于`list-max-ziplist-entries` (默认512个)&& 所有的值小于`list-max-ziplist-value`(默认64字节)时用ziplist）
      2. linkedlist 
      3. Redis3.2提供了quicklist内部编码，它是以ziplist为节点的linkedlist。
  - set
    - 内部编码
      1. intset （集合中元素都是整数且个数小于`set-max-intset-entries`(默认512)个时用intset，减少内存的使用）
      2. hashtable
  - zset(sort set)
    - 内部编码
      1. ziplist  （有序集合元素个数小于`zset-max-ziplist-entries`(默认128个)&&所有的元素值都小于`zet-max-ziplist-value`时用ziplist  ，减少内存的使用）
      2. skiplist

[操作命令](http://redisdoc.com/)

### Pipeline

- **Pipeline能组装一组Redis命令，通过一次RTT传输给Redis,再将这组Redis命令的执行结果按顺序返回给客户端。**（即批量执行客户端命令）

- 原生批量命令与Pipeline对比
  - 原生批量命令是原子的，Pipeline不是原子的
  - 原生批量命令是一个命令对应多个key，Pipeline支持多个命令
  - 原生命令是Redis服务端支持实现的，而Pipeline需要客户端、服务端共同实现。

### 事务

- Redis提供了简单的事务。不支持事务的回滚特性。可能一个命令错了，所有命令都不能执行(比如某个命令写错)，也可以一个命令错了，其他命令可以正常执行(运行时错误)。

- 常用命令

  `multi`：标记一个事务的开始。

  `exec`：执行事务块内的命令。

  `discard`：取消事务，放弃事务块内的所有命令。

  `watch`：监视一个或多个key，如果在事务执行前这些key被其他命令修改了，那么事务将会被打断。

 ### 发布订阅

- 进程建的一种消息通信模式：发送者发送消息，订阅者接收消息，发布者与订阅者不进行直接通信。

- 常用命令

  `publish channel message`：将message发送到channel频道

  `subscribe channle [channle...]`：订阅一个或多个频道

  `unsubscribe [channle [channle ...]]`：取消订阅

  <div align="center"><img src="img/发布订阅二.bmp" style="center"></div><br>
  
  `psubscribe pattern [pattern ...]`：按照模式订阅
  
  <div align="center"><img src="img/发布订阅一.bmp" style="center"></div><br>
  
  `punsubscribe pattern [pattern ...]`：安装模式取消订阅
  
  `pubsub numpat`：查看模式订阅数
  
  `pubsub numsub [channle ...]`：查看频道订阅数

### 客户端通信协议

- 几乎所有的主流编程语言都有Redis的客户端，原因
  1. 客户端与服务端之间的通信协议基于TCP协议构建。
  2. Redis制定了RESP(Redis Serialization Protocol ，Redis序列化协议)实现客户端与服务端的正常交互。

### 持久化

#### RDB

- RDB持久化即把当前数据一次性生成快照保存到硬盘，产生文件压缩比更高，因此读取RDB速度更快。由于生成RDB开销大，无法做到实时持久化，所以一般用于数据备份和复制传输。
- 触发方式有手动触发和自动触发。
  - 手动触发：两个命令`save`和`bgsave`。
    - `save`：阻塞Redis服务器，直到RDB完成，这个过程中客户端不能连接
    - `bgsave`：Redis进程执行fork操作创建子进程，RDB持久化过程由子进程负责，完成后自动结束，阻塞只发生在fork阶段，一般时间很短。
  - 自动触发：在配置文件的SNAPSHOTTING栏，如下为默认配置。 `save m n`表示m秒内数据集存在n次修改就自动触发`bgsave`。可通过`save ""`关闭自动触发。

<div align="center"><img src="img/持久化一.bmp" style="center"></div><br>
- `bgsave`执行流程：  
1. 执行`bgsave`命令，父进程判断当前是否存在正在执行的子进程，如RDB/AOF。如果存在`bgsave`命令就直接返回。  
2. 父进程执行fork操作，自身阻塞。  
3. fork完成后`bgsave`命令返回`background saving started`信息并不再阻塞父进程。  
4. 子进程创建RDB文件并对原有文件进行原子替换。  
5. 进程发送信号给父进程表示完成，父进程更新统计信息。
- RDB优缺点
  - 优点
    1. RDB是一个紧凑压缩的二进制文件(LZF压缩算法)，适合备份和容灾恢复。
    2. Redis加载RDB恢复数据远快于AOF方式。
    3. RDB对redis对外提供读写服务的时候，影响非常小，因为redis 主进程只需要fork一个子进程出来，主进程不需进行任何磁盘IO操作。
  - 缺点
    1. 在一定间隔时间做一次备份，所以如果redis意外down掉的话，就会丢失最后一次快照后的所有修改。
    2. Fork的时候，内存中的数据被克隆了一份，大致2倍的膨胀性需要考虑。

#### AOF

- AOF持久化以日志的形式来记录每个写操作(读操作不记录)，只许追加文件但不可以改写文件，redis启动之初会读取该文件重新构建数据。

  使用AOF需要先开启，默认关闭，在配置文件的APPEND ONLY MODE下将` appendonly no`改为 `appendonly yes`即可开启。
- 运行流程

  1. 命令写入。所有写入的命令会追加到aof_buf(aof缓冲区)中。
  2. 文件同步。aof缓冲区根据对应的策略向硬盘做同步操作。
  3. 文件重写。随着aof文件越来越大，定期对aof文件重写以达到压缩目的。
  4. 重启加载。Redis服务器重启时，加载aof文件进行数据恢复。
  
  - 命令写入
    - 命令写入的内容是文本协议的格式，文本协议有很好的兼容性，而且可读性强，方便直接修改。
    - aof把命令追加到aof_buf中而不直接写入硬盘，这样redis可以提供多种缓冲区同步硬盘的策略，在性能和安全性方面做出平衡。
  - 文件同步策略
    - always：同步持久化 每次发生数据变更会被立即记录到磁盘  性能较差但数据完整性比较好。
    - everysec：出厂默认推荐，异步操作，每秒记录   如果一秒内宕机，有数据丢失。
    - no：从不同步。 
  - 重写机制
    - 当AOF文件的大小超过所设定的阈值时，Redis就会启动AOF文件的内容压缩，这样就降低了文件占用空间，而且更小的文件可以更快的被Redis加载。
    - 触发方式
      1. 手动触发：`bgrewriteaof`命令。
      2. 自动触发：配置文件中有`auto-aof-rewrite-percentage 100` 和`auto-aof-rewrite-min-size 64mb`两项配置，表示当AOF文件大小是上次rewrite后大小的一倍且文件大于64M时触发。
    - 重写流程
      1. 执行aof重写请求，如果当前进程正在执行aof重写，请求就不会执行并返回`err background append only file rewriting already in progress`；如果当前进程正在执行`bgsave`，重写命令延迟到`bgsave`完成后再执行，返回`background append only file rewriting scheduled`。
      2. 父进程执行fork创建子进程，开销等同于`bgsave`过程。
      3. 父进程fork操作完成后，继续响应其他命令。
      4. 由于fork操作运用写时复制技术，子进程只能共享fork操作时的内存数据。由于父进程依然会响应命令，Redis使用“aof重写缓冲区”保存这部分数据，防止新aof文件生成期间丢失这部分数据。
      5. 子进程根据内存快照，按照命令合并规则写入到新的aof文件。如果`aof-rewrite-incremental-fsync`配置为yes，则每次写32M。
      6. 新aof文件写入完成后，子进程发送信号给父进程，父进程更新统计信息。
      7. 父进程把“aof重写缓冲区”的数据写入到新的aof文件。
      8. 使用新aof文件替换老文件，完成aof重写。
  - 重启加载
  
    - aof开启并存在aof文件时，会优先加载aof。
    - 如果aof文件错误，可通过 `redis-check-aof --fix  <filename>`命令修复,修复后使用`diff -u`对比数据差异，找出丢失数据。
    - `aof-load-truncated`如果为yes(默认yes)，会兼容aof文件结尾不完整的情况。即在aof写入时，可能存在指令写错的问题(突然断电，写了一半)。

#### 总结

- aof文件是一个只进行追加的日志文件。
- redis可以在aof文件体积变得过大时自动在后台对aof进行重写。
- aof有序保存了对数据库所有的写入操作，这些写入操作以文本协议格式保存，因此aof文件容易被读懂。
- 对于相同数据集来说，aof文件通常大于rdb文件体积。
- 根据所使用的fsync策略，aof的速度可能会慢于rdb。

#### 持久化阻塞主线程场景

1. fork阻塞

   fork创建的子进程会复制父进程的空间内存页表，所以fork阻塞时间与进程总内存量息息相关。
   
   -  改善fork操作耗时
   
     - 优先使用物理机或支持fork操作的虚拟化技术。
   
     - 控制Redis实例最大可用内存。线上建议每个Redis实例内存控制在10GB内。
     - 合理配置Linux内存分配策略。
     - 降低fork操作频率。
   
2. aof追加阻塞

  - 阻塞流程

    1. 主线程负责写入aof缓冲区。
    2. aof线程负责每秒执行一次同步磁盘操作，并记录最近一次同步时间。
    3. 主线程负责对比上一次aof同步时间：如果距上次同步成功时间在2秒内，主线程直接返回；如果距上次同步成功时间超过2秒内，主线程会阻塞，直到同步操作完成。

    所以，everysec配置最多丢失2秒数据。

    如果系统fsync缓慢，会导致Redis主线程阻塞影响效率。

    解决阻塞要优化**系统硬盘负载**。


#### 多实例部署

- Redis单线程架构导致无法充分利用CPU多核特性，通常做法是在一台机器上部署多个Redis实例。
- 多个Redis实例会产生对CPU和IO的竞争，需要进行**隔离控制**。通过`info persistence`监控子进程运行状况。基于`info persistence`提供的指标，通过外部程序轮询控制aof重写操作的执行。

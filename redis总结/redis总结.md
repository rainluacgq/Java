## Redis总结

### 一、概念

Redis 是速度非常快的非关系型（NoSQL）内存键值数据库，可以存储键和五种不同类型的值之间的映射。键的类型只能为字符串，值支持五种数据类型：字符串、列表、集合、散列表、有序集合。Redis 支持很多特性，例如将内存中的数据持久化到硬盘中，使用复制来扩展读性能，使用分片来扩展写性能。  

![image-20200505103557919](https://github.com/rainluacgq/java/blob/master/redis%E6%80%BB%E7%BB%93/pic/image-20200505103557919.png)

Redis如此快的原因：

- Redis 完全基于内存，绝大部分请求是纯粹的内存操作，执行效率高。

- Redis 使用单进程单线程模型的（K，V）数据库，将数据存储在内存中，存取均不会受到硬盘 IO 的限制，因此其执行速度极快。

  另外单线程也能处理高并发请求，还可以避免频繁上下文切换和锁的竞争，如果想要多核运行也可以启动多个实例。

- 数据结构简单，对数据操作也简单，Redis 不使用表，不会强制用户对各个关系进行关联，不会有复杂的关系限制，其存储结构就是键值对，类似于 HashMap，HashMap 最大的优点就是存取的时间复杂度为 O(1)。

- Redis 使用多路 I/O 复用模型，为非阻塞 IO。

  Redis 采用的 I/O 多路复用函数：epoll/kqueue/evport/select。

### Redis实战技术

#### Redis持久化

Redis提供两种持久化机制 RDB 和 AOF 机制:

1、RDB（Redis DataBase)持久化方式：

是指用数据集快照的方式半持久化模式记录 redis 数据库的所有键值对,在某个时间点将数据写入一个临时文件，持久化结束后，用这个临时文件替换上次持久化的文件，达到数据恢复。

优点：

（1）只有一个文件 dump.rdb，方便持久化。

（2）容灾性好，一个文件可以保存到安全的磁盘。

（3）性能最大化，fork 子进程来完成写操作，让主进程继续处理命令，所以是 IO最大化。使用单独子进程来进行持久化，主进程不会进行任何 IO 操作，保证了 redis的高性能

（4）相对于数据集大时，比 AOF 的启动效率更高。

缺点：

数据安全性低。RDB 是间隔一段时间进行持久化，如果持久化之间 redis 发生故障，会发生数据丢失。所以这种方式更适合数据要求不严谨的时候

RDB  redis.conf配置如下：

```ini
 save 900 1 #在900s内如果有1条数据被写入，则产生一次快照。
 save 300 10 #在300s内如果有10条数据被写入，则产生一次快照
 save 60 10000 #在60s内如果有10000条数据被写入，则产生一次快照
 stop-writes-on-bgsave-error yes 
 #stop-writes-on-bgsave-error ：
 如果为yes则表示，当备份进程出错的时候，
 主进程就停止进行接受新的写入操作，这样是为了保护持久化的数据一致性的问题。
```

2、AOF（Append-only file)持久化方式：

是指所有的命令行记录以 redis 命令请求协议的格式完全持久化存储)保存为 aof 文件。

 优点：

（1）数据安全，aof 持久化可以配置 appendfsync 属性，有 always，每进行一次命令操作就记录到 aof 文件中一次。

（2）通过 append 模式写文件，即使中途服务器宕机，可以通过 redis-check-aof工具解决数据一致性问题。

（3）AOF 机制的 rewrite 模式。AOF 文件没被 rewrite 之前（文件过大时会对命令进行合并重写），可以删除其中的某些命令（比如误操作的 flushall）)

缺点：

（1）AOF 文件比 RDB 文件大，且恢复速度慢。

（2）数据集大的时候，比 rdb 启动效率低。

AOF 配置：

```ini
 appendonly yes

 #appendsync always
 appendfsync everysec
 # appendfsync no
```





### Redis 的淘汰策略有哪些？

#### Redis 有六种淘汰策略

| 策略            | 描述                                                         |
| :-------------- | :----------------------------------------------------------- |
| volatile-lru    | 从已设置过期时间的 KV 集中优先对最近最少使用(less recently used)的数据淘汰 |
| volitile-ttl    | 从已设置过期时间的 KV 集中优先对剩余时间短(time to live)的数据淘汰 |
| volitile-random | 从已设置过期时间的 KV 集中随机选择数据淘汰                   |
| allkeys-lru     | 从所有 KV 集中优先对最近最少使用(less recently used)的数据淘汰 |
| allKeys-random  | 从所有 KV 集中随机选择数据淘汰                               |
| noeviction      | 不淘汰策略，若超过最大内存，返回错误信息                     |



#### 4.0 版本后增加以下两种

- volatile-lfu：从已设置过期时间的数据集(server.db[i].expires)中挑选最不经常使用的数据淘汰
- allkeys-lfu：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的 key



#### Redis 集群：

（1）Redis Sentinal 着眼于高可用，在 master 宕机时会自动将 slave 提升为master，继续提供服务。

 主从模式弊端：当 Master 宕机后，Redis 集群将不能对外提供写入操作。Redis Sentinel 可解决这一问题。

解决主从同步 Master 宕机后的主从切换问题：

- **监控：**检查主从服务器是否运行正常。
- **提醒：**通过 API 向管理员或者其它应用程序发送故障通知。
- **自动故障迁移：**主从切换（在 Master 宕机后，将其中一个 Slave 转为 Master，其他的 Slave 从该节点同步数据）。
- ![image-20200505112548094](https://github.com/rainluacgq/java/blob/master/redis%E6%80%BB%E7%BB%93/pic/image-20200505112548094.png)

（2）Redis Cluster 着眼于扩展性，在单个 redis 内存不足时，使用 Cluster 进行分片存储。

Redis Cluster是一种服务端Sharding技术，3.0版本开始正式提供。Redis Cluster并没有使用一致性hash，而是采用slot(槽)的概念，一共分成16384个槽。将请求发送到任意节点，接收到请求的节点会将查询请求发送到正确的节点上执行

![image-20200505112606144](https://github.com/rainluacgq/java/blob/master/redis%E6%80%BB%E7%BB%93/pic/image-20200505112606144.png)

参考文章：https://mp.weixin.qq.com/s/COZ8Uohv9gDYiAPJ_J4oww

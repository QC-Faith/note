# 基本知识

它支持的数据结，如 [字符串（strings）](http://redis.cn/topics/data-types-intro.html#strings)， [散列（hashes）](http://redis.cn/topics/data-types-intro.html#hashes)， [列表（lists）](http://redis.cn/topics/data-types-intro.html#lists)， [集合（sets）](http://redis.cn/topics/data-types-intro.html#sets)， [有序集合（sorted sets）](http://redis.cn/topics/data-types-intro.html#sorted-sets) 与范围查询， [bitmaps](http://redis.cn/topics/data-types-intro.html#bitmaps)， [hyperloglogs](http://redis.cn/topics/data-types-intro.html#hyperloglogs) 和 [地理空间（geospatial）](http://redis.cn/commands/geoadd.html) 索引半径查询。 Redis 内置了 [复制（replication）](http://redis.cn/topics/replication.html)，[LUA脚本（Lua scripting）](http://redis.cn/commands/eval.html)， [LRU驱动事件（LRU eviction）](http://redis.cn/topics/lru-cache.html)，[事务（transactions）](http://redis.cn/topics/transactions.html) 和不同级别的 [磁盘持久化（persistence）](http://redis.cn/topics/persistence.html)， 并通过 [Redis哨兵（Sentinel）](http://redis.cn/topics/sentinel.html)和自动 [分区（Cluster）](http://redis.cn/topics/cluster-tutorial.html)提供高可用性（high availability）。

> Redis的工作线程是单线程的，但是整个Redis是多线程的（`持久化、异步删除、集群数据同步等`这些都是由额外的线程执行的）

因为 Redis 是基于内存的操作，CPU不是Redis的性能瓶颈。Redis 的瓶颈最有可能是机器内存的大小或者网络带宽。既然单线程容易实现，而且 CPU 不会成为瓶颈，那就顺理成章地采用单线程的方案了。Redis是使用C语言开发的，官方数据表明单个节点的Redis  QPS为 10w+，完全不比同样是使用key-value的memecache差

> Redis为什么要设计为单线程？单线程为何还这么快？
>
> 1. 基于内存。Redis命令操作主要都是基于内存，这已经足够快，不需要借助多线程。对于内存系统来说，如果没有上下文切换效率就是最高的，多次读写都是在一个CPU上的
> 2. 高效的数据结构。Redis底层提供了动态简单动态字符串([SDS](https://github.com/jmilktea/jmilktea/blob/master/redis/string类型底层SDS.md))、跳表(skiplist)、压缩列表(ziplist)等数据结构来高效访问数据。
> 3. 保持简单。引入多线程会使Redis变得复杂，例如需要考虑多线程并发访问资源竞争问题，数据结构也会变得复杂，hash就不能是单纯的hash，需要像java一样设计一个ConcurrentHashMap。还需要考虑线程切换带来的性能损耗

误区 ：

1. 高性能的服务器不一定是多线程的
2. 多线程（CPU上下文会切换）不一定比单线程效率高

速度：CPU > 内存 > 硬盘

**优势：**

1. 提高性能：缓存查询速度比数据库查询速度快（内存 `VS` 硬盘）
2. 提高并发能力：缓存分担了部分请求，支持更高的并发

# 分布式锁

## 特征

- **「互斥性」**: 任意时刻，只有一个客户端能持有锁。
- **「锁超时释放」**：持有锁超时，可以释放，防止不必要的资源浪费，也可以防止死锁。
- **「可重入性」**:一个线程如果获取了锁之后,可以再次对其请求加锁。
- **「高性能和高可用」**：加锁和解锁需要开销尽可能低，同时也要保证高可用，避免分布式锁失效。
- **「安全性」**：锁只能被持有的客户端删除，不能被其他客户端删除

## 实现方案：

<font color='Chestnut Red'>**单机部署**</font>：

1. <font color='Apricot'>**基于缓存（如Redis）实现**</font>：利用分布式缓存系统（如Redis）的原子性操作，通过SETNX（SET if Not eXists）或者类似的原子操作来实现锁。Redis 还提供了更高级的 SETEX 命令，可以设置锁的过期时间，避免死锁问题。

   1. SETNX + EXPIRE

      - SETNX key value，如果 key不存在，则 SETNX 成功返回1，如果 key 已经存在，则返回0。
      - 但是这个方案中，`setnx`和`expire`两个命令分开了，**「不是原子操作」**。如果执行完`setnx`加锁，正要执行`expire`设置过期时间时，进程**崩溃**或者要重启维护了，那么这个锁就“长生不老”了，**「别的线程永远获取不到」**

   2. SETNX + value值是（系统时间+过期时间）

      可以把过期时间放到`setnx`的value值里面。如果加锁失败，再拿出value值校验一下即可。加锁代码如下：

      ~~~java
      long expires = System.currentTimeMillis() + expireTime; //系统时间+设置的过期时间
      String expiresStr = String.valueOf(expires);
      
      // 如果当前锁不存在，返回加锁成功
      if (jedis.setnx(key_resource_id, expiresStr) == 1) {
              return true;
      } 
      // 如果锁已经存在，获取锁的过期时间
      String currentValueStr = jedis.get(key_resource_id);
      
      // 如果获取到的过期时间，小于系统当前时间，表示已经过期
      if (currentValueStr != null && Long.parseLong(currentValueStr) < System.currentTimeMillis()) {
      
           // 锁已过期，获取上一个锁的过期时间，并设置现在锁的过期时间（
          String oldValueStr = jedis.getSet(key_resource_id, expiresStr);
          if (oldValueStr != null && oldValueStr.equals(currentValueStr)) {
               // 考虑多线程并发的情况，只有一个线程的设置值和当前值相同，它才可以加锁
               return true;
          }
      }        
      //其他情况，均返回加锁失败
      return false;
      }
      ~~~

      **缺点**：

      - 过期时间是客户端自己生成的（System.currentTimeMillis()是当前系统的时间），必须要求分布式环境下，每个客户端的时间必须同步。
      - 如果锁过期的时候，并发多个客户端同时请求过来，都执行`jedis.getSet()`，最终只能有一个客户端加锁成功，但是该客户端锁的过期时间，可能被别的客户端覆盖
      - 该锁没有保存持有者的唯一标识，可能被别的客户端释放/解锁。

   3. 使用Lua脚本(包含SETNX + EXPIRE两条指令)

   4. SET的扩展命令（SET EX PX NX）

      > SET key value [EX seconds] [PX milliseconds] [NX|XX]，它也是原子性的
      >
      > - NX :表示key不存在的时候，才能set成功，也即保证只有第一个客户端请求才能获得锁，而其他客户端请求只能等其释放锁，才能获取。
      > - EX seconds :设定key的过期时间，时间单位是秒。
      > - PX milliseconds: 设定key的过期时间，单位为毫秒
      > - XX: 仅当key存在时设置值

      **缺点**：

      - **「锁过期释放了，业务还没执行完」**。假设线程a获取锁成功，一直在执行临界区的代码。但是100s过去后，它还没执行完。但是，这时候锁已经过期了，此时线程b又请求过来。显然线程b就可以获得锁成功，也开始执行临界区的代码。那么问题就来了，临界区的业务代码都不是严格串行执行的。
      - **「锁被别的线程误删」**。假设线程a执行完后，去释放锁。但是它不知道当前的锁可能是线程b持有的（线程a去释放锁时，有可能过期时间已经到了，此时线程b进来占有了锁）。那线程a就把线程b的锁释放掉了，但是线程b临界区业务代码可能都还没执行完。

   5. SET EX PX NX  + 校验唯一随机值，再释放锁

2. <font color='Apricot'>**基于Zookeeper实现**</font>：使用 ZooKeeper 提供的临时节点（ephemeral node）特性，通过创建一个临时节点来表示锁的持有。客户端创建临时节点成功的那个将获得锁，其他客户端则监听该节点的删除事件。Zookeeper作为一种高可用的分布式协调服务，相比于Redis分布式锁，Zookeeper分布式锁具有以下优缺点：		

   > 优点：			
   >
   > 1. 可以避免锁的失效问题：Zookeeper采用基于ZAB协议的分布式一致性算法，可以保证分布式锁的强一致性，避免因主从同步延迟导致锁失效问题。
   > 2. 支持更复杂的锁机制：Zookeeper提供了两种锁实现方式：共享锁和排他锁，可以根据具体的业务场景选择最适合的方式。
   > 3. 可以与其他Zookeeper服务集成：Zookeeper还提供了诸如分布式队列、命名服务等功能，可以与分布式锁一起使用，构建更完整的分布式应用系统。
   >
   > 缺点：
   >
   > 1. 性能相对较低：Zookeeper采用基于ZAB协议的分布式一致性算法，需要进行多次网络通信和数据同步，相比于Redis分布式锁，性能相对较低。
   > 2. 部署和维护成本较高：Zookeeper需要部署专门的服务器集群，需要进行一定的配置和维护工作，相比于Redis分布式锁，部署和维护成本较高。

3. <font color='Apricot'>**基于数据库实现**</font>，有两种方式：		

   - 使用数据库的事务性和唯一性约束来实现分布式锁。通过在数据库中插入一条记录或者更新一条记录，利用数据库的唯一性约束来确保只有一个客户端能够成功插入或更新。
   - 利用数据库的乐观锁机制，通过在记录中添加版本号等字段，实现分布式锁。客户端在获取锁时，检查版本号，成功则获取锁，失败则重试或者放弃。

   > 相比于Redis和Zookeeper分布式锁，MySQL分布式锁具有以下优缺点：
   >
   > 优点：
   >
   > 1. 易于部署和维护：MySQL已经广泛应用于各种应用场景中，部署和维护相对较为简单。
   > 2. 支持更复杂的锁机制：MySQL提供了多种锁实现方式，如行锁、表锁、读锁、写锁等，可以根据具体的业务场景选择最适合的方式。
   >
   > 缺点：
   >
   > 1. 性能较低：MySQL采用基于磁盘的存储方式，相比于Redis和Zookeeper，性能较低。
   > 2. 可扩展性较差：由于MySQL采用基于磁盘的存储方式，其可扩展性较差，难以应对高并发场景下的锁控制需求。
   > 3. 存在单点故障问题：MySQL采用主从复制的方式进行数据同步，存在单点故障问题，可能导致锁失效问题。

4. <font color='Apricot'>**开源框架~Redisson**</font>

   > Redisson中有比较成熟的分布式锁的方案，其提供了一个专门用来监控和续期锁的 Watch Dog（ 看门狗），如果操作共享资源的线程还未执行完成的话，Watch Dog 会不断地延长锁的过期时间，进而保证锁不会因为超时而被释放。（默认情况下，每过 10 秒，看门狗就会执行续期操作，将锁的超时时间设置为 30 秒。看门狗续期前也会先判断是否需要执行续期操作，需要才会执行续期，否则取消续期操作，具体是异步续期通过lua脚本操作）
   >
   > 可重入锁：一个线程中可以多次获取同一把锁，为每个锁关联一个可重入计数器和一个占有它的线程。Redisson 内置了多种类型的锁比如可重入锁（Reentrant Lock）、自旋锁（Spin Lock）、公平锁（Fair Lock）、多重锁（MultiLock）、 红锁（RedLock）、 读写锁（ReadWriteLock）

<font color='Chestnut Red'>**集群部署**</font>：

- 多机实现的分布式锁Redlock

  - Redlock核心思想是这样的：

    搞多个Redis master部署，以保证它们不会同时宕掉。并且这些master节点是完全相互独立的，相互之间不存在数据同步。同时，需要确保在这多个master实例上，是与在Redis单实例，使用相同方法来获取和释放锁。

    > 假设当前有5个Redis master节点，在5台服务器上面运行这些Redis实例。RedLock的实现步骤如下：
    >
    > 1. 获取当前时间，以毫秒为单位。
    > 2. 按顺序向5个master节点请求加锁。客户端设置网络连接和响应超时时间，并且超时时间要小于锁的失效时间。（假设锁自动失效时间为10秒，则超时时间一般在5-50毫秒之间,我们就假设超时时间是50ms吧）。如果超时，跳过该master节点，尽快去尝试下一个master节点。
    > 3. 客户端使用当前时间减去开始获取锁时间（即步骤1记录的时间），得到获取锁使用的时间。当且仅当超过一半（N/2+1，这里是5/2+1=3个节点）的Redis master节点都获得锁，并且使用的时间小于锁失效时间时，锁才算获取成功。（如上图，10s> 30ms+40ms+50ms+4m0s+50ms）
    > 4. 如果取到了锁，key的真正有效时间就变啦，需要减去获取锁所使用的时间。
    > 5. 如果获取锁失败（没有在至少N/2+1个master实例取到锁，有或者获取锁时间已经超过了有效时间），客户端要在所有的master节点上解锁（即便有些master节点根本就没有加锁成功，也需要解锁，以防止有些漏网之鱼）。

# Redis的高并发和快速原因

1. Redis是纯内存数据库，一般都是简单的存取操作，线程占用的时间很多，时间的花费主要集中在IO上，所以读取速度快

2. Redis采用了单线程的模型，保证了每个操作的原子性，也减少了线程的上下文切换和竞争。

  > Redis 单线程主要指的是网络IO和键值对读写是由一个线程来完成的，Redis在处理客户端的请求的时候包括获取(Socket)，解析，执行，内容返回(Socket写)等由一个顺序串行的主线程处理，这即为单线程 但Redis的其它功能，比如持久化，异步删除，集群数据同步等等，都是由额外的线程执行的，是多线程 
  >
  > 即 <font color='Chestnut Red'>Redis工作线程是单线程的，但整个Redis来说是多线程的</font>

3. Redis使用IO多路复用技术（解决对多个I/O监听时,一个I/O阻塞影响其他I/O的问题），可以处理并发的连接（非阻塞IO）。非阻塞 IO 内部实现采用epoll，采用了epoll+自己实现的简单的事件框架。epoll中的读、写、关闭、连接都转化成了事件，然后利用epoll的多路复用特性，绝不在io上浪费一点时间。

4. Redis使用的是非阻塞IO，IO多路复用，使用了单线程来轮询描述符，将数据库的开、关、读、写都转换成了事件，减少了线程切换时上下文的切换和竞争。

5. Redis全程使用hash的键值对存储结构，读取速度快，还有一些特殊的数据结构，对数据存储进行了优化，如压缩表，对短数据进行压缩存储，再如，跳表，使用有序的数据结构加快读取的速度。

6. Redis采用自己实现的事件分离器，效率比较高，内部采用非阻塞的执行方式，吞吐能力比较大。

# 热 key

我们把<font color='Magenta'>访问频率高的Key，称为热Key</font>。比如突然又几十万的请求去访问redis中某个特定的Key，那么这样会造成redis服务器短时间流量过于集中，很可能导致redis的服务器宕机。那么接下来对这个Key的请求，都会直接请求到我们的后端数据库中，数据库性能本来就不高，这样就可能直接压垮数据库，进而导致后端服务不可用。

## 产生原因

1. 用户消费的数据远大于生产的数据，如商品秒杀、热点新闻、热点评论等读多写少的场景。

   > 双十一秒杀商品，短时间内某个爆款商品可能被点击/购买上百万次，或者某条爆炸性新闻等被大量浏览，此时会造成一个较大的请求Redis量，这种情况下就会造成热点Key问题。

2. 请求分片集中，超过单台Redis服务器的性能极限。

   > 在服务端读数据进行访问时，往往会对数据进行分片切分，例如采用固定 Hash 分片，Hash 落入同一台redis服务器，如果瞬间访问量过大，超过机器瓶颈时，就会导致热点 Key 问题的产生。

## 危害

> 缓存击穿，压垮redis服务器，导致大量请求直接发往后端服务，并且DB本身性能较弱，很可能进一步导致后端服务雪崩。

## 识别热 key

> 1. 凭借个人经验，结合业务场景，判断哪些是热Key
>
>    比如，双十一大促的时候，苹果手机正在秒杀，那么我们可以判断苹果手机这个 sku 就是热Key。
>
> 2. 使用redis之前，在客户端写程序统计key的使用次数。
>
>    修改业务代码，在操作redis之前，加入Key使用次数的统计逻辑，定时把收集到的数据上报到统一的服务进行聚合计算，这样我们就可以找到那些热点Key。缺点就是对我们的业务代码有一定的侵入性。
>
> 3. redis节点抓包分析
>
>    自己写程序监听端口，解析数据，进行分析。

## 解决方案

1. redis集群扩容，增加分片数量，分摊客户端发过来的读请求
1. 随机的Redis Key，通过得知redis集群的分片数量，去设置这个key，将redis集群的key分散到不同的机器上。
1. 使用二级缓存，即 JVM 本地缓存，减少redis的读请求
1. 限流：通过控制请求的速率来防止系统过载。在应用层实现限流，可以有效减轻热点Key对Redis的压力。常见的限流算法有漏桶算法和令牌桶算法。

# 大key问题

<font color='Magenta'>`Redis`的`key`和`String`类型`value`限制均为`512MB`。</font>虽然`Key`的大小上限为`512M`,但是**一般建议`key`的大小不要超过`1KB`**，这样既可以节约存储空间，又有利于`Redis`进行检索

<span style="background:#f9eda6;">String类型</span>：一个`String`类型的`value`最大可以存储`512M`

<span style="background:#f9eda6;">List、Set、Hash、Sorted set类型</span>：单个元素的`value`上限为`512M`

## 影响

1. 内存占用过高。大`Key`占用过多的内存空间，可能导致可用内存不足，从而触发内存淘汰策略。在极端情况下，可能导致内存耗尽，`Redis`实例崩溃，影响系统的稳定性。
2. 性能下降。大`Key`会占用大量内存空间，导致内存碎片增加，进而影响`Redis`的性能。对于大`Key`的操作，如读取、写入、删除等，都会消耗更多的`CPU`时间和内存资源，进一步降低系统性能。
3. 阻塞其他操作。某些对大`Key`的操作可能会导致`Redis`实例阻塞。例如，使用`DEL`命令删除一个大`Key`时，可能会导致`Redis`实例在一段时间内无法响应其他客户端请求，从而影响系统的响应时间和吞吐量。
4. 网络拥塞。每次获取大`key`产生的网络流量较大，可能造成机器或局域网的带宽被打满，同时波及其他服务。例如：一个大key占用空间是`1MB`，每秒访问`1000`次，就有`1000MB`的流量。
5. 主从同步延迟。当`Redis`实例配置了主从同步时，大`Key`可能导致主从同步延迟。由于大`Key`占用较多内存，同步过程中需要传输大量数据，这会导致主从之间的网络传输延迟增加，进而影响数据一致性。
6. 数据倾斜。在`Redis`集群模式中，某个数据分片的内存使用率远超其他数据分片，无法使数据分片的内存资源达到均衡。另外也可能造成`Redis`内存达到`maxmemory`参数定义的上限导致重要的`key`被逐出，甚至引发内存溢出。

## 原因

1. 业务设计不合理。这是最常见的原因，不应该把大量数据存储在一个`key`中，而应该分散到多个`key`。例如：把全国数据按照省行政区拆分成`34`个`key`，或者按照城市拆分成`300`个`key`，可以进一步降低产生大`key`的概率。
2. 没有预见`value`的动态增长问题。如果一直添加`value`数据，没有删除机制、过期机制或者限制数量，迟早出现大`key`。例如：微博明星的粉丝列表、热门评论等。
3. 过期时间设置不当。如果没有给某个`key`设置过期时间，或者过期时间设置较长。随着时间推移，`value`数量快速累积，最终形成大`key`。
4. 程序`bug`。某些异常情况导致某些`key`的生命周期超出预期，或者`value`数量异常增长 ，也会产生大`key`。

## 排查大key

<font color='Magenta'>SCAN命令</font>

通过使用`Redis`的`SCAN`命令，我们可以逐步遍历数据库中的所有`Key`。结合其他命令（如`STRLEN`、`LLEN`、`SCARD`、`HLEN`等），我们可以识别出大`Key`。`SCAN`命令的优势在于它可以在不阻塞`Redis`实例的情况下进行遍历。

<font color='Magenta'>bigkeys参数</font>

使用`redis-cli`命令客户端，连接`Redis`服务的时候，加上 `—bigkeys `参数，可以扫描每种数据类型数量最大的`key`。

> redis-cli -h 127.0.0.1 -p 6379 —bigkeys

<font color='Magenta'>Redis RDB Tools工具</font>

使用开源工具`Redis RDB Tools`，分析`RDB`文件，扫描出`Redis`大`key`。

> 例如：输出占用内存大于1kb，排名前3的keys。
>
> rdb —commond memory —bytes 1024 —largest 3 dump.rbd

## 解决方式

1. 拆分成多个小`key`。这是最容易想到的办法，降低单`key`的大小，读取可以用`mget`批量读取。
2. 数据压缩。使用`String`类型的时候，使用压缩算法减少`value`大小。或者是使用`Hash`类型存储，因为`Hash`类型底层使用了压缩列表数据结构。
3. 设置合理的过期时间。为每个`key`设置过期时间，并设置合理的过期时间，以便在数据失效后自动清理，避免长时间累积的大`Key`问题。
4. 启用内存淘汰策略。启用`Redis`的内存淘汰策略，例如`LRU`（`Least Recently Used`，最近最少使用），以便在内存不足时自动淘汰最近最少使用的数据，防止大`Key`长时间占用内存。
5. 数据分片。例如使用`Redis Cluster`将数据分散到多个`Redis`实例，以减轻单个实例的负担，降低大`Key`问题的风险。
6. 删除大`key`。使用`UNLINK`命令删除大`key`，`UNLINK`命令是`DEL`命令的异步版本，它可以在后台删除`Key`，避免阻塞`Redis`实例。

# 数据库&缓存一致性保证

建议采用“<span style="background:#f9eda6;">**先写数据库再删除缓存**</span>”的方式

1. 先确定是更新缓存还是删除缓存：

   <font color='Apricot'>应该优先选择删除缓存而不是更新缓存。</font>原因如下：

   1. 我们放到缓存中的数据，很多时候可能不只是简单的一个字符串类型的值，他还可能是一个大的`JSON`串，一个`map`类型等等。如果选择更新，那么业务中先需要从缓存中查出整个模型数据，把他进行反序列化之后，再解析出其中的库存字段，把他修改掉，然后再序列化，最后再更新到缓存中。相比于直接删除缓存，操作过程比较的复杂，而且也容易出错。

   2. "写写并发"的场景中，如果同时更新缓存和数据库，那么很容易会出现因为并发的问题导致数据不一致的情况。如：

      <img src="https://gitee.com/qc_faith/picture/raw/master/image/202312251719055.png" alt="image-20231225171927029" style="zoom: 25%;" />

      <font color='Magenta'>如果是做缓存的删除的话，在写写并发的情况下，缓存中的数据都是要被清除的，所以就不会出现数据不一致的问题。</font>但是，删除缓存相比更新缓存有一个小缺点，那就是带来的一次额外的`cache miss`，也就是说在删除缓存后的下一次查询会无法命中缓存，要查询一下数据库。<font color='Magenta'>这种cache miss在某种程度上可能会导致缓存击穿</font>，也就是刚好缓存被删除之后，同一个`Key`有大量的请求过来，导致缓存被击穿，大量请求访问到数据库。但是，通过加锁的方式是可以比较方便的解决缓存击穿的问题的。

      总之，删除缓存相比较更新缓存，方案更加简单，而且带来的一致性问题也更少。所以，在删除和更新缓存之间，建议优先选择删除缓存。

2. 确定好是删除缓存后再确定是"先写数据库后删除缓存"还是"先删除缓存后写数据库"。

   <font color='Apricot'>先写数据库后删除缓存</font>，因为数据库和缓存的操作是两步的，没办法做到保证原子性，所以就有可能第一步成功而第二步失败。一般情况下，如果把缓存的删除动作放到第二步，由如下好处：

   1. <font color='Magenta'>缓存删除失败的概率还是比较低的</font>，除非是网络问题或者缓存服务器宕机的问题，否则大部分情况都是可以成功的。

   2. 先写数据库后删除缓存虽然不存在"写写并发"导致的数据一致性问题，但是会存在"读写并发"情况下的数据一致性问题。我们知道，当我们使用了缓存之后，一个读的线程在查询数据的过程是这样的：1、查询缓存，如果缓存中有值，则直接返回 2、否则查询数据库 3、把数据库的查询结果更新到缓存中。所以，<font color='Magenta'>对于一个读线程来说，虽然不会写数据库，但是是会更新缓存的</font>，所以，在一些特殊的并发场景中，就会导致数据不一致的情况。读写并发的时序如下：

      <img src="https://gitee.com/qc_faith/picture/raw/master/image/202402281752845.png" alt="image-20231225171056260" style="zoom: 33%;" />

   也就是说，假如一个读线程，在读缓存的时候没查到值，他就会去数据库中查询，但是如果自查询到结果之后，更新缓存之前，数据库被更新了，但是这个读线程是完全不知道的，那么就导致最终缓存会被重新用一个"旧值"覆盖掉。这也就导致了<font color='Magenta'>缓存和数据库的不一致的现象</font>。但是这种现象其实发生的概率比较低，因为一般一个读操作是很快的，数据库+缓存的读操作基本在十几毫秒左右就可以完成了。而在这期间，刚好另一个线程执行完了一个比较耗时的写操作的概率确实比较低。

3. 如果先删缓存后写数据库会有什么问题

   首先，<font color='Magenta'>如果是选择先删除缓存后写数据库的这种方案，那么第二步的失败是可以接受的</font>，因为这样不会有脏数据，也没什么影响，只需要重试就好了。

   但是，<font color='Magenta'>先删除缓存后写数据库的这种方式，会无形中放大前面我们提到的"读写并发"导致的数据不一致的问题。</font>

   因为这种"读写并发"问题发生的前提是读线程读缓存没读到值，而先删缓存的动作一旦发生，刚好可以让读线程就从缓存中读不到值。

   所以，本来一个小概率会发生的"读写并发"问题，在先删缓存的过程中，问题发生的概率会被放大。

   而且这种问题的后果也比较严重，那就是缓存中的值一直是错的，就会导致后续的所以命中缓存的查询结果都是错的！

## 延迟双删

```tsx
先删除缓存；再写数据库；休眠1-2s；再次删除缓存。不休眠的话，读请求还有可能未结束，造成脏数据
```

虽然先写数据后删除缓存的这种情况，可以大大的降低并发问题的概率，但是依旧是有出问题的可能。可以采用延迟双删来应对。

因为"读写并发"的问题会导致并发发生后，缓存中的数据被读线程写进去脏数据，那么就只需要在写线程在写数据库、删缓存之后，延迟一段时间，在执行一把删除缓存动作就行了。

这样就能保证缓存中的脏数据被清理掉，避免后续的读操作都读到脏数据。这个延迟的时长一般建议设置1-2s就可以了。

<font color='RedOrange'>弊端</font>：可能会导致缓存中准确的数据被删除掉。当然这也问题不大，只是增加一次cache miss罢了。

## 总结

主要还是根据实际的业务情况来分析。

如果业务量不大，并发不高的情况，可以选择**先写数据库，后删除缓存**的方式，因为这种方案更加简单。

但是，如果是业务量比较大，并发度很高的话，建议选择**先删除缓存**，因为这种方式在引入延迟双删、分布式锁等机制后，会使得整个方案会更加趋近于完美，带来的并发问题更少。当然，也会更复杂。

#  I/O 多路复用

Redis 底层采用的是 I/O 多路复用模型来保证在多连接的时候系统的高吞吐量，**IO多路复用机制是指一个线程处理多个IO流，多路是指网络连接，复用指的是同一个线程。**关于 I/O 多路复用(又被称为“事件驱动”)，首先要理解的是，操作系统为你提供了一个功能，当你的某个 socket 可读或者可写的时候，它可以给你一个通知。这样当配合非阻塞的 socket 使用时，只有当系统通知我哪个描述符可读了，我才去执行 read 操作，可以保证每次 read 都能读到有效数据而不做纯返回 -1 和 EAGAIN 的无用功，写操作类似。

操作系统的这个功能是通过 select/poll/epoll/kqueue 之类的系统调用函数来实现，这些函数都可以同时监视多个描述符的读写就绪状况，这样，多个描述符的 I/O 操作都能在一个线程内并发交替地顺序完成，这就叫 I/O 多路复用。多路---指的是多个 socket 连接，复用---指的是复用同一个 Redis 处理线程。多路复用主要有三种技术：select，poll，epoll。epoll 是最新的也是目前最好的多路复用技术。

采用多路 I/O 复用技术可以让单个线程高效的处理多个连接请求(尽量减少网络 I/O 的时间消耗)，且 Redis 在内存中操作数据的速度非常快，也就是说内存内的操作不会成为影响 Redis 性能的瓶颈，基于这两点 Redis 具有很高的吞吐量。

#  数据类型

`string 	set 	zset 	hash	list`

| 数据类型 | 可以存储的值           | 操作                                                         | 应用场景                                                     | 内部实现                                                     |
| -------- | ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| STRING   | 字符串、整数或者浮点数 | 对整个字符串或者字符串的其中一部分执行操作，对整数和浮点数执行自增或者自减操作 | 做简单的键值对缓存                                           | 主要是 int 和 SDS（简单动态字符串）                          |
| LIST     | 列表                   | 按照插入顺序排序，从两端压入或者弹出元素，对单个或者多个元素进行修剪， 只保留一个范围内的元素。最大长度为 2^32 - 1，也即每个列表支持超过 40 亿个元素。 | 存储一些列表型的数据结构，类似粉丝列表、文章的评论列表之类的数据；可当做栈、队列、阻塞队列 | **双向链表或压缩列表**实现（如果列表的元素个数小于 512 个（默认值，可由 list-max-ziplist-entries 配置），列表每个元素的值都小于 64 字节（默认值，可由 list-max-ziplist-value 配置），Redis 会使用**压缩列表**作为 List 类型的底层数据结构；否则使用**双向链表**） |
| SET      | 无序并唯一的键值集合   | 添加、获取、移除单个元素、检查一个元素是否存在于集合中、计算交集、并集、差集 从集合里面随机获取元素 | 交集、并集、差集的操作，比如交集，可以把两个人的粉丝列表整一个交集 | 由**哈希表或整数集合**实现（如果集合中的元素都是整数且元素个数小于 512 （默认值，set-maxintset-entries配置）个使用**整数集合**作为底层数据结构；否则使用**哈希表**） |
| HASH     | 包含键值对的无序散列表 | 添加、获取、移除单个键值对、获取所有键值对、检查某个键是否存在 | 存储用户信息，商品信息等                                     | Redis 7.0 后由 listpack 数据结构来实现；7.0之前是**压缩列表或哈希表**（如果哈希类型元素个数小于 512 个（默认值，可由 hash-max-ziplist-entries 配置），所有值小于 64 字节（默认值，可由 hash-max-ziplist-value 配置）的话使用**压缩列表**作为底层数据结构；否则使用**哈希表**） |
| ZSET     | 有序集合               | 添加、获取、删除元素、根据分值范围或者成员来获取元素、计算一个键的排名 | 去重但可以排序，如获取排名前几名的用户                       | 由**压缩列表或跳表**实现（如果有序集合的元素个数小于 128 个，且每个元素的值小于 64 字节使用**压缩列表**作为 Zset 类型的底层数据结构；否则使用**跳表**） |

**三种特殊数据类型**：

> `BitMap`（位图--2.2 版新增）：位存储，`vip`和非`vip`用户，网站活跃、不活跃用户，登录、未登录用户，打开、未打卡，任何两种状态都可以用`bitmap`实现、操作二进制位，只有`0、1`两个状态
>
> `HyperLogLog`（2.8 版新增）：用来做基数（集合中不重复的数）统计的算法，优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。
>
> `GEO`（地理位置--3.2 版新增）：`Geo`底层是`Zset`实现，可以用`zset`命令来操作`geo`，用于存储和操作地理位置信息，例如滴滴打车叫车、搜索附近餐厅
>
> `Stream`（5.0 版新增）：专为消息队列设计的数据类型，支持消息的持久化、支持自动生成全局唯一 ID、支持 ack 确认消息的模式、支持消费组模式等，让消息队列更加的稳定和可靠

### 一、String（字符串）

在`Redis`中`String`是可以修改的，称为`动态字符串`(`Simple Dynamic String` 简称 `SDS`)，内部实现主要是 int 和 SDS（简单动态字符串）。可存储字符串、整数或者浮点数，说是字符串但它的内部结构更像是一个 `ArrayList`，内部维护着一个字节数组，并且在其内部预分配了一定的空间，以减少内存的频繁分配。

`Redis`的内存分配机制是这样：

- 当字符串的长度小于 1MB时，每次扩容都是加倍现有的空间。
- 如果字符串长度超过 1MB时，每次扩容时只会扩展 1MB 的空间。

这样既保证了内存空间够用，还不至于造成内存的浪费，**字符串最大长度为** **`512MB`.**。

`String`的数据结构

```java
struct SDS{
  T capacity;       //数组容量
  T len;            //实际长度
  byte flages;  //标志位,低三位表示类型
  byte[] content;   //数组内容
}
```

`capacity` 和 `len`两个属性都是泛型，为更合理的使用内存，不同长度的字符串采用不同的数据类型表示，且在创建字符串的时候 `len` 会和 `capacity` 一样大，不产生冗余的空间，所以`String`值可以是字符串、数字（整数、浮点数) 或者 二进制。

**应用场景**：存储key-value键值对

### 二、List(列表)

`Redis`中的`list`和`Java`中的`LinkedList`很像，但底层是**双向链表或压缩列表**实现， `list`的插入和删除操作非常快，时间复杂度为 0(1)，不像数组结构插入、删除操作需要移动数据。

当列表的元素个数小于 512 个（`默认值，可由 list-max-ziplist-entries 配置`），列表每个元素的值都小于 64 字节（`默认值，可由 list-max-ziplist-value 配置`）的时候它的底层存储结构为一块连续内存，称之为`ziplist(压缩列表)`，它将所有的元素紧挨着一起存储，分配的是一块连续的内存；否则使用**双向链表**。

可纯的链表也是有缺陷的，链表的前后指针 `prev` 和 `next` 会占用较多的内存，会比较浪费空间，而且会加重内存的碎片化。在redis 3.2之后就都改用`ziplist+链表`的混合结构，称之为 `quicklist(快速链表)`。

#### ziplist(压缩列表)

`ziplist`的数据结构

```c
struct ziplist<T>{
    int32 zlbytes;            //压缩列表占用字节数
    int32 zltail_offset;    //最后一个元素距离起始位置的偏移量,用于快速定位到最后一个节点
    int16 zllength;            //元素个数
    T[] entries;            //元素内容
    int8 zlend;                //结束位 0xFF
}
```

压缩列表为了支持双向遍历，所以才会有 `ztail_offset` 这个字段，用来快速定位到最后一个元素，然后倒着遍历

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220502162733.jpg" alt="img" style="zoom:67%;" />

`entry`的数据结构：

```c
struct entry{
    int<var> prevlen;            //前一个 entry 的长度
    int<var> encoding;            //元素类型编码
    optional byte[] content;    //元素内容
}
```

`entry`它的 `prevlen` 字段表示前一个 `entry` 的字节长度，当压缩列表倒着遍历时，需要通过这个字段来快速定位到下一个元素的位置。

**应用场景：**

由于`list`它是一个按照插入顺序排序的列表，所以应用场景相对还较多的，例如：

- 消息队列：`lpop`和`rpush`（或者反过来，`lpush`和`rpop`）能实现队列的功能；以及栈、阻塞队列等
- 朋友圈的点赞列表、评论列表、排行榜：`lpush`命令和`lrange`命令能实现最新列表的功能，每次通过`lpush`命令往列表里插入新的元素，然后通过`lrange`命令读取最新的元素列表。

### 三、Hash （字典）

`Redis` 中的 `Hash`和 `Java`的 `HashMap` 更加相似，都是 `数组+链表` 的结构，当发生 hash 碰撞时将会把元素追加到链表上，值得注意的是在 `Redis` 的 `Hash` 中 `value` 只能是字符串.

Hash 类型的底层数据结构是由**压缩列表或哈希表**实现的：

- 如果哈希类型元素个数小于 512 个（默认值，可由 hash-max-ziplist-entries 配置），所有值小于 64 字节（默认值，可由 hash-max-ziplist-value 配置）的话，Redis 会使用**压缩列表**作为 Hash 类型的底层数据结构；
- 如果哈希类型元素不满足上面条件，Redis 会使用**哈希表**作为 Hash 类型的 底层数据结构。

**在 Redis 7.0 中，压缩列表数据结构已经废弃了，交由 listpack 数据结构来实现了**。

```sql
hset books java "Effective java" (integer) 1
hset books golang "concurrency in go" (integer) 1
hget books java "Effective java"
hset user age 17 (integer) 1
hincrby user age 1    #单个 key 可以进行计数 和 incr 命令基本一致 (integer) 18
```

`Hash` 和`String`都可以用来存储用户信息 ，但不同的是`Hash`可以对用户信息的每个字段单独存储；`String`存的是用户全部信息经过序列化后的字符串，如果想要修改某个用户字段必须将用户信息字符串全部查询出来，解析成相应的用户信息对象，修改完后在序列化成字符串存入。而 hash可以只对某个字段修改，从而节约网络流量，不过hash内存占用要大于 `String`，这是 `hash` 的缺点。

**应用场景：**

- 购物车：`hset [key] [field] [value]` 命令， 可以实现以`用户Id`，`商品Id`为`field`，商品数量为`value`，恰好构成了购物车的3个要素。
- 存储对象：`hash`类型的`(key, field, value)`的结构与对象的`(对象id, 属性, 值)`的结构相似，也可以用来存储对象。

### 四、Set(集合)

`Redis` 中的 `set`和`Java`中的`HashSet` 有些类似，它内部的键值对是无序的、唯一 的。它的内部实现相当于一个特殊的字典，字典中所有的`value`都是一个值 `NULL`。当集合中最后一个元素被移除之后，数据结构被自动删除，内存被回收。一个集合最多可以存储 2^32-1 个元素。

Set 类型的底层数据结构是由**哈希表或整数集合**实现的：

- 如果集合中的元素都是整数且元素个数小于 512 （默认值，set-maxintset-entries配置）个，Redis 会使用**整数集合**作为 Set 类型的底层数据结构；
- 如果集合中的元素不满足上面条件，则 Redis 使用**哈希表**作为 Set 类型的底层数据结构。

**应用场景：**

- 好友、关注、粉丝、感兴趣的人集合：
  1. `sinter`命令可以获得A和B两个用户的共同好友；
  2. `sismember`命令可以判断A是否是B的好友；
  3. `scard`命令可以获取好友数量；
  4. 关注时，`smove`命令可以将B从A的粉丝集合转移到A的好友集合
- 首页展示随机：美团首页有很多推荐商家，但是并不能全部展示，set类型适合存放所有需要展示的内容，而`srandmember`命令则可以从中随机获取几个。
- 存储某活动中中奖的用户ID ，因为有去重功能，可以保证同一个用户不会中奖两次。

### 五、Zset(有序集合)

Zset 相比于 Set 类型多了一个排序属性 score（分值），每个存储元素相当于有两个值组成的，一个是有序结合的元素值，一个是排序值。ZSet 保留了Set 不能有重复成员的特性（分值可以重复），不同的是，有序集合中的元素可以排序。内部实现为`zipList`和`dict+skipList`

当`zset`满足以下两个条件的时候，使用`ziplist`：（**在 Redis 7.0 中，压缩列表数据结构已经废弃了，由 listpack 数据结构来实现。**）

1. 保存的元素少于128个
2. 保存的所有元素大小都小于64字节

**不满足这两个条件则使用skiplist。**（这两个数值可以通过`redis.conf`的`zset-max-ziplist-entries` 和 `zset-max-ziplist-value`选项 进行修改。）

**应用场景**：排行榜

#### ziplist

- 当`ziplist`作为`zset`的底层存储结构时候，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的成员，第二个元素保存元素的分值。其实质是一个双向链表；虽然元素是按 `score` 有序排序的， 但对 `ziplist` 的节点指针只能线性地移动，所以查找给定元素的复杂度是 `O(N)`

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202312212347092.png" alt="img" style="zoom:67%;" />

- `ziplist`内存结构

<img src="https://gitee.com/qc_faith/picture/raw/master/image/202312212349611.png" alt="img" style="zoom: 50%;" />

各个部分在内存上是前后相邻的并连续的，每一部分作用如下：

- `zlbytes`： 存储一个无符号整数，固定四个字节（32bit），用于存储压缩列表所占用的字节（也包括`zlbytes`本身占用的4个字节），当重新分配内存的时候使用，不需要遍历整个列表来计算内存大小。

- `zltail`： 存储一个无符号整数，固定四个字节（32bit），表示`ziplist`表中最后一项（`entry`）在ziplist中的偏移字节数。`zltail`的存在，使得我们可以很方便地找到最后一项（不用遍历整个`ziplist`），从而可以在`ziplist`尾端快速地执行`push`或`pop`操作。

- `zllen`： 压缩列表包含的节点个数，固定两个字节（16bit），表示`ziplist`中数据项（`entry`）的个数。由于`zllen`字段只有`16bit`，所以可以表达的最大值为2^16-1^。

  <font color=VioletRed>**注意**</font>：如果`ziplist`中数据项个数超过了`16bit`能表达的最大值，`ziplist`仍然可以表示。`ziplist`是如何做到的？

  如果`zllen`小于等于2^16-2^（也就是不等于2^16-1^），那么`zllen`就表示`ziplist`中数据项的个数；否则，也就是`zllen`等于`16bit`全为`1`的情况，那么`zllen`就不表示数据项个数了，这时候要想知道`ziplist`中数据项总数，那么必须对`ziplist`从头到尾遍历各个数据项，才能计数出来。

- `entry`，表示真正存放数据的数据项，长度不定。一个数据项（`entry`）也有它自己的内部结构。

- `zlend`， `ziplist`最后1个字节，值固定等于255，其是一个结束标记。

​	<font color=Rhodamine>**优缺点**</font>：当数据小的时候，由一段连续的内存组成,最大的优点就是节省内存，但这种结构不善于修改

#### skiplist

包含`一个dict + 一个skiplist`。字典的键保存元素的值，字典的值则保存元素的分值；跳跃表节点的 `object` 属性保存元素的成员，跳跃表节点的 `score` 属性保存元素的分值。查找单个`key`的时间复杂度为`O(log n)`

这两种数据结构会<font color='each'>通过指针来共享相同元素的成员和分值</font>，所以不会产生重复成员和分值，造成内存的浪费。

>其实有序集合单独使用字典或跳跃表其中一种数据结构都可以实现，但是这里使用两种数据结构组合起来，原因是假如我们单独使用 字典，虽然能以 O(1) 的时间复杂度查找成员的分值，但是因为字典是以无序的方式来保存集合元素，所以<font color='orange'>每次进行范围操作的时候都要进行排序</font>；假如我们单独使用跳跃表来实现，虽然能执行范围操作，<font color='orange'>但是查找操作由 O(1)的复杂度变为了O(logN)。因此Redis使用了两种数据结构来共同实现有序集合。</font>

字典中的键是唯一的，可以通过`key`来查找值

- 字典底层实现是哈希表，字典有两个哈希表，一个在扩容时使用，哈希表扩容使用渐进式扩容，发送扩容时需要在两个哈希表中进行搜索。
- 发生哈希冲突时使用链地址法解决

<font color='VioletRed'>skiplist与平衡树、哈希表的比较</font>

- <font color='Peach'>有序性</font>：`skiplist`和各种平衡树（如AVL、红黑树等）的元素是有序排列的，而哈希表不是有序的。因此，在哈希表上只能做单个`key`的查找，不适宜做范围查找。所谓范围查找，指的是查找那些大小在指定的两个值之间的所有节点。
- <font color='Peach'>范围查找</font>：平衡树比`skiplist`操作要复杂。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。如果不对平衡树进行一定的改造，这里的中序遍历并不容易实现。而在`skiplist`上进行范围查找就非常简单，只需要在找到小值之后，对第1层链表进行若干步的遍历就可以实现。
- <font color='Peach'>插入和删除</font>：平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而`skiplist`的插入和删除只需要修改相邻节点的指针，操作简单又快速。
- <font color='Peach'>内存占用</font>：`skiplist`比平衡树更灵活一些。一般来说，平衡树每个节点包含2个指针（分别指向左右子树），而`skiplist`每个节点包含的指针数目平均为`1/(1-p)`，具体取决于参数`p`的大小。如果像`Redis`里的实现一样，取`p=1/4`，那么平均每个节点包含`1.33`个指针，比平衡树更有优势。
- <font color='Peach'>查找单个`key`</font>：`skiplist`和平衡树的时间复杂度都为`O(log n)`，大体相当；而哈希表在保持较低的哈希值冲突概率的前提下，查找时间复杂度接近`O(1)`，性能更高一些。所以我们平常使用的各种`Map`或`dictionary`结构，大都是基于哈希表实现的。
- <font color='Peach'>算法实现难度上</font>：`skiplist`比平衡树要简单得多。

**zset应用场景：**

`zset` 可以用做排行榜，但是和`list`不同的是`zset`它能够实现动态的排序，例如： 可以用来存储粉丝列表，value 值是粉丝的用户 ID，score 是关注时间，我们可以对粉丝列表按关注时间进行排序。

`zset` 还可以用来存储学生的成绩， `value` 值是学生的 ID, `score` 是他的考试成绩。 我们对成绩按分数进行排序就可以得到他的名次。

### 跳跃链表

**有序列表 Zset** 它的内部实现就依赖了一种叫做 **「跳跃列表」** 的数据结构。

<font color='Apricot'>为什么用跳跃列表？</font>

首先，因为 `zset `要支持随机的插入和删除，所以它 **不宜使用数组来实现**，关于排序问题，出于 **性能 **和 **实现** 考虑使用的是跳跃列表而不是 **红黑树/ 平衡树** 这样的树形结构

1. **性能考虑：** 在高并发的情况下，树形结构需要执行一些类似于 rebalance 这样的可能涉及整棵树的操作，相对来说跳跃表的变化只涉及局部 *(下面详细说)*；
2. **实现考虑：** 在复杂度与红黑树相同的情况下，跳跃表实现起来更简单，看起来也更加直观；

<font color='Apricot'>本质是解决查找问题</font>

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220502105147.png" alt="image-20220502105145786" style="zoom:50%;" />

我们需要这个链表按照 score 值进行排序，这也就意味着，当我们需要添加新的元素时，我们需要定位到插入点，这样才可以继续保证链表是有序的，通常我们会使用 **二分查找法**，但二分查找是有序数组的，链表没办法进行位置定位，我们除了遍历整个找到第一个比给定数据大的节点为止 *（时间复杂度 O(n))* 似乎没有更好的办法。

但假如我们每相邻两个节点之间就增加一个指针，让指针指向下一个节点，这样所有新增的指针连成了一个新的链表，但它包含的数据却只有原来的一半 。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220502105257.png" alt="image-20220502105255807" style="zoom:50%;" />

现在假设我们想要查找数据时，可以根据这条新的链表查找，如果碰到比待查找数据大的节点时，再回到原来的链表中进行查找，比如，我们想要查找 7，查找的路径则是沿着下图中标注出的红色指针所指向的方向进行的：

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220502105401.png" alt="image-20220502105400335" style="zoom:50%;" />

这是一个略微极端的例子，但我们仍然可以看到，通过新增加的指针查找，我们不再需要与链表上的每一个节点逐一进行比较，这样改进之后需要比较的节点数大概只有原来的一半。

利用同样的方式，我们可以在新产生的链表上，继续为每两个相邻的节点增加一个指针，从而产生第三层链表：

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220502105427.png" alt="image-20220502105426146" style="zoom:50%;" />

在这个新的三层链表结构中，我们试着 **查找 13**，那么沿着最上层链表首先比较的是 11，发现 11 比 13 小，于是我们就知道只需要到 11 后面继续查找，**从而一下子跳过了 11 前面的所有节点。**

可以想象，当链表足够长，这样的多层链表结构可以帮助我们跳过很多下层节点，从而加快查找的效率

#### 更进一步的跳跃表

**跳跃表 skiplist** 就是受到这种多层链表结构的启发而设计出来的。按照上面生成链表的方式，上面每一层链表的节点个数，是下面一层的节点个数的一半，这样查找过程就非常类似于一个二分查找，使得查找的时间复杂度可以降低到 *O(logn)*。

但是，这种方法在插入数据的时候有很大的问题。新插入一个节点之后，就会打乱上下相邻两层链表上节点个数严格的 2:1 的对应关系。如果要维持这种对应关系，就必须把新插入的节点后面的所有节点 *（也包括新插入的节点）* 重新进行调整，这会让时间复杂度重新蜕化成 *O(n)*。删除数据也有同样的问题。

**skiplist** 为了避免这一问题，它不要求上下相邻两层链表之间的节点个数有严格的对应关系，而是 **为每个节点随机出一个层数(level)**。比如，一个节点随机出的层数是 3，那么就把它链入到第 1 层到第 3 层这三层链表中。为了表达清楚，下图展示了如何通过一步步的插入操作从而形成一个 skiplist 的过程：

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220502105630.png" alt="image-20220502105628712" style="zoom:50%;" />

从上面的创建和插入的过程中可以看出，每一个节点的层数（level）是随机出来的，而且新插入一个节点并不会影响到其他节点的层数，因此，**插入操作只需要修改节点前后的指针，而不需要对多个节点都进行调整**，这就降低了插入操作的复杂度。

现在我们假设从我们刚才创建的这个结构中查找 23 这个不存在的数，那么查找路径会如下图：

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220502105703.png" alt="image-20220502105701759" style="zoom:50%;" />

#### 跳跃表的实现

Redis 中的跳跃表由 `server.h/zskiplistNode` 和 `server.h/zskiplist` 两个结构定义，前者为跳跃表节点，后者则保存了跳跃节点的相关信息，同之前的 `集合 list` 结构类似，其实只有 `zskiplistNode` 就可以实现了，但是引入后者是为了更加方便的操作：

```java
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    // value
    sds ele;
    // 分值
    double score;
    // 后退指针
    struct zskiplistNode *backward;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    // 跳跃表头指针
    struct zskiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中层数最大的节点的层数
    int level;
} zskiplist;
```

<font color='Apricot'>随机层数</font>

对于每一个新插入的节点，都需要调用一个随机算法给它分配一个合理的层数，源码在 `t_zset.c/zslRandomLevel(void)` 中被定义：

```java
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

直观上期望的目标是 50% 的概率被分配到 `Level 1`，25% 的概率被分配到 `Level 2`，12.5% 的概率被分配到 `Level 3`，以此类推...有 2-63 的概率被分配到最顶层，因为这里每一层的晋升率都是 50%。

**Redis 跳跃表默认允许最大的层数是 32**，被源码中 `ZSKIPLIST_MAXLEVEL` 定义，当 `Level[0]` 有 264 个元素时，才能达到 32 层，所以定义 32 完全够用了。

<font color='Apricot'>创建跳跃表</font>

这个过程比较简单，在源码中的 `t_zset.c/zslCreate` 中被定义：

```java
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    // 申请内存空间
    zsl = zmalloc(sizeof(*zsl));
    // 初始化层数为 1
    zsl->level = 1;
    // 初始化长度为 0
    zsl->length = 0;
    // 创建一个层数为 32，分数为 0，没有 value 值的跳跃表头节点
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    
    // 跳跃表头节点初始化
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        // 将跳跃表头节点的所有前进指针 forward 设置为 NULL
        zsl->header->level[j].forward = NULL;
        // 将跳跃表头节点的所有跨度 span 设置为 0
        zsl->header->level[j].span = 0;
    }
    // 跳跃表头节点的后退指针 backward 置为 NULL
    zsl->header->backward = NULL;
    // 表头指向跳跃表尾节点的指针置为 NULL
    zsl->tail = NULL;
    return zsl;
}
```

即执行完之后创建了如下结构的初始化跳跃表：

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220502105923.png" alt="image-20220502105922140" style="zoom:50%;" />

<font color='Apricot'>插入节点实现</font>

1. 找到当前我需要插入的位置 *（其中包括相同 score 时的处理）*；
2. 创建新节点，调整前后的指针指向，完成插入；

源码 `t_zset.c/zslInsert` 定义的插入函数可分为几个部分

**第一部分：声明需要存储的变量**

```c
// 存储搜索路径
zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
// 存储经过的节点跨度
unsigned int rank[ZSKIPLIST_MAXLEVEL];
int i, level;
```

**第二部分：搜索当前节点插入位置**

```c
serverAssert(!isnan(score));
x = zsl->header;
// 逐步降级寻找目标节点，得到 "搜索路径"
for (i = zsl->level-1; i >= 0; i--) {
    /* store rank that is crossed to reach the insert position */
    rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
    // 如果 score 相等，还需要比较 value 值
    while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) < 0)))
    {
        rank[i] += x->level[i].span;
        x = x->level[i].forward;
    }
    // 记录 "搜索路径"
    update[i] = x;
}
```

**讨论：** 有一种极端的情况，就是跳跃表中的所有 score 值都是一样，zset 的查找性能会不会退化为 O(n) 呢？

从上面的源码中我们可以发现 zset 的排序元素不只是看 score 值，也会比较 value 值 *（字符串比较）*

**第三部分：生成插入节点**

```c
/* we assume the element is not already inside, since we allow duplicated
 * scores, reinserting the same element should never happen since the
 * caller of zslInsert() should test in the hash table if the element is
 * already inside or not. */
level = zslRandomLevel();
// 如果随机生成的 level 超过了当前最大 level 需要更新跳跃表的信息
if (level > zsl->level) {
    for (i = zsl->level; i < level; i++) {
        rank[i] = 0;
        update[i] = zsl->header;
        update[i]->level[i].span = zsl->length;
    }
    zsl->level = level;
}
// 创建新节点
x = zslCreateNode(level,score,ele);
```

**第四部分：重排前向指针**

```c
for (i = 0; i < level; i++) {
    x->level[i].forward = update[i]->level[i].forward;
    update[i]->level[i].forward = x;

    /* update span covered by update[i] as x is inserted here */
    x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
    update[i]->level[i].span = (rank[0] - rank[i]) + 1;
}

/* increment span for untouched levels */
for (i = level; i < zsl->level; i++) {
    update[i]->level[i].span++;
}
```

**第五部分：重排后向指针并返回**

```c
x->backward = (update[0] == zsl->header) ? NULL : update[0];
if (x->level[0].forward)
    x->level[0].forward->backward = x;
else
    zsl->tail = x;
zsl->length++;
return x;
```

<font color='Apricot'>节点删除实现</font>

删除过程由源码中的 `t_zset.c/zslDeleteNode` 定义，和插入过程类似，都需要先把这个 **"搜索路径"** 找出来，然后对于每个层的相关节点重排一下前向后向指针，同时还要注意更新一下最高层数 `maxLevel`，直接放源码 *(如果理解了插入这里还是很容易理解的)*：

```c
/* Internal function used by zslDelete, zslDeleteByScore and zslDeleteByRank */
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            update[i]->level[i].span -= 1;
        }
    }
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    zsl->length--;
}

/* Delete an element with matching score/element from the skiplist.
 * The function returns 1 if the node was found and deleted, otherwise
 * 0 is returned.
 *
 * If 'node' is NULL the deleted node is freed by zslFreeNode(), otherwise
 * it is not freed (but just unlinked) and *node is set to the node pointer,
 * so that it is possible for the caller to reuse the node (including the
 * referenced SDS string at node->ele). */
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* We may have multiple elements with the same score, what we need
     * is to find the element with both the right score and object. */
    x = x->level[0].forward;
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        zslDeleteNode(zsl, x, update);
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}
```

<font color='Apricot'>节点更新实现</font>

当我们调用 `ZADD` 方法时，如果对应的 value 不存在，那就是插入过程，如果这个 value 已经存在，只是调整一下 score 的值，那就需要走一个更新流程。

假设这个新的 score 值并不会带来排序上的变化，那么就不需要调整位置，直接修改元素的 score 值就可以了，但是如果排序位置改变了，那就需要调整位置，该如何调整呢？

从源码 `t_zset.c/zsetAdd` 函数 `1350` 行左右可以看到，Redis 采用了一个非常简单的策略：

```c
/* Remove and re-insert when score changed. */
if (score != curscore) {
    zobj->ptr = zzlDelete(zobj->ptr,eptr);
    zobj->ptr = zzlInsert(zobj->ptr,ele,score);
    *flags |= ZADD_UPDATED;
}
```

**把这个元素删除再插入这个**，需要经过两次路径搜索，从这一点上来看，Redis 的 `ZADD` 代码似乎还有进一步优化的空间。

<font color='Apricot'>元素排名的实现</font>

跳跃表本身是有序的，Redis 在 skiplist 的 forward 指针上进行了优化，给每一个 forward 指针都增加了 `span` 属性，用来 **表示从前一个节点沿着当前层的 forward 指针跳到当前这个节点中间会跳过多少个节点**。在上面的源码中我们也可以看到 Redis 在插入、删除操作时都会小心翼翼地更新 `span` 值的大小。

所以，沿着 **"搜索路径"**，把所有经过节点的跨度 `span` 值进行累加就可以算出当前元素的最终 rank 值了：

```c
/* Find the rank for an element by both score and key.
 * Returns 0 when the element cannot be found, rank otherwise.
 * Note that the rank is 1-based due to the span of zsl->header to the
 * first element. */
unsigned long zslGetRank(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *x;
    unsigned long rank = 0;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) <= 0))) {
            // span 累加
            rank += x->level[i].span;
            x = x->level[i].forward;
        }

        /* x might be equal to zsl->header, so test if obj is non-NULL */
        if (x->ele && sdscmp(x->ele,ele) == 0) {
            return rank;
        }
    }
    return 0;
}
```

### Bitmap

`Bitmap`，即位图，是一串连续的二进制数组（0和1），可以通过偏移量（`offset`）定位元素。`BitMap`通过最小的单位`bit`来进行`0|1`的设置，表示某个元素的值或者状态，时间复杂度为`O(1)`。由于`bit`是计算机中最小的单位，使用它进行储存将非常节省空间，特别适合一些数据量大且使用二值统计的场景。

> 这里的二值状态就是指集合元素的取值就只有 0 和 1 两种。例如在签到打卡的场景中，我们只用记录签到（1）或未签到（0），所以它就是非常典型的二值状态。在签到统计时，每个用户一天的签到用 1 个 bit 位就能表示，一个月（假设是 31 天）的签到情况用 31 个 bit 位就可以，而一年的签到也只需要用 365 个 bit 位，根本不用太复杂的集合类型。这个时候，我们就可以选择 Bitmap。

**内部实现：**

Bitmap不属于Redis的基本数据类型，而是基于String类型进行的位操作。String 类型是会保存为二进制的字节数组，所以，Redis 就把字节数组的每个 bit 位利用起来，用来表示一个元素的二值状态，可以把 Bitmap 看作是一个 bit 数组。而Redis中字符串的最大长度是 512M，所以 BitMap 的 offset 值也是有上限的，其最大值是：`8 * 1024 * 1024 * 512  =  2^32`

> 优点
>
> - 基于最小的单位bit进行存储，所以非常省空间。
> - 设置时候时间复杂度`O(1)`、读取时候时间复杂度`O(n)`，操作是非常快的。
> - 二进制数据的存储，进行相关计算的时候非常快。
> - 方便扩容
>
> 缺点
>
> `Redis`中`bit`映射被限制在`512MB`之内，所以最大是`2^32`位。建议每个`key`的位数都控制下，因为读取时候时间复杂度`O(n)`，越大的串读的时间花销越多。

### HyperLogLog

Redis 2.8.9 版本新增的数据类型，是一种用于「统计基数」的数据集合类型，基数统计就是指统计一个集合中不重复的元素个数。注意：HyperLogLog 是统计规则是基于概率完成的，不是非常准确，标准误算率是 0.81%。简单来说 HyperLogLog **提供不精确的去重计数**。

优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的内存空间总是固定的、并且是很小的。**每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64个不同元素的基数**，和元素越多就越耗费内存的 Set 和 Hash 类型相比，HyperLogLog 非常节省空间。

**应用场景**：百万级网页 UV 计数

### GEO

Redis 3.2 版本新增的数据类型，主要用于存储地理位置信息，并对存储的信息进行操作。

在日常生活中，我们越来越依赖搜索“附近的餐馆”、在打车软件上叫车，这些都离不开基于位置信息服务（Location-Based Service，LBS）的应用。LBS 应用访问的数据是和人或物关联的一组经纬度信息，而且要能查询相邻的经纬度范围，GEO 就非常适合应用在 LBS 服务的场景中。

**内部实现**：直接使用了 Sorted Set 集合类型。

> GEO 类型使用 GeoHash 编码方法实现了经纬度到 Sorted Set 中元素权重分数的转换，这其中的两个关键机制就是「对二维地图做区间划分」和「对区间进行编码」。一组经纬度落在某个区间后，就用区间的编码值来表示，并把编码值作为 Sorted Set 元素的权重分数。这样一来，就可以把经纬度保存到 Sorted Set 中，利用 Sorted Set 提供的“按权重进行有序范围查找”的特性，实现 LBS 服务中频繁使用的“搜索附近”的需求。

### Stream

Redis 5.0 版本新增加的数据类型，专门为消息队列设计的数据类型。

没有 Stream 之前，消息队列的实现方式都有着各自的缺陷，例如：

- 发布订阅模式，不能持久化也就无法可靠的保存消息，并且对于离线重连的客户端不能读取历史消息的缺陷；
- List 实现消息队列的方式不能重复消费，一个消息消费完就会被删除，而且生产者需要自行实现全局唯一 ID。

基于以上问题，Redis 5.0 便推出了 Stream 类型也是此版本最重要的功能，用于完美地实现消息队列，它支持消息的持久化、支持自动生成全局唯一 ID、支持 ack 确认消息的模式、支持消费组模式等，让消息队列更加的稳定和可靠。

# 管道（pipeline）

Redis客户端与服务器之间使用TCP协议进行通信。在某些高并发的场景下，网络开销成了Redis速度的瓶颈，所以需要使用管道技术来实现突破。在Redis中，管道（pipeline）是一种将多个命令一次性发送给服务器并获取所有回复的机制。通过使用管道，可以在一次通信中发送多个命令，而不是通过多次独立的通信来执行这些命令。这种方式可以减少通信的开销，提高性能。

## 优势与作用

1. **减少通信开销：** 在一次通信中发送多个命令，减少了每个命令的通信开销，提高了效率。
2. **原子性：** 一次性发送的多个命令会在服务器端原子性地执行，不会被其他命令插入。
3. **减少网络延迟：** 通过减少通信次数，降低了因网络延迟引起的总体响应时间。

<font color='Chestnut Red'>注意：</font>

如果客户端使用管道发送了多条命令，那么服务器就会将多条命令放入一个队列中，这一操作会消耗一定的内存，所以**管道中命令的数量并不是越大越好**（太大容易撑爆内存），而是应该有一个合理的值。

## 使用流程

1. 客户端向服务器发送多个命令，而不等待服务器回复。
2. 服务器按照接收到的顺序执行这些命令。
3. 服务器将执行结果按照命令发送的顺序返回给客户端。

使用管道可以在一定程度上提高Redis的性能，特别是在需要执行多个命令的场景下，比如批量读写操作。

**单条命令的执行步骤**：

- 客户端把命令发送到服务器，然后阻塞客户端，等待着从socket读取服务器的返回结果

- 服务器处理命令并将结果返回给客户端

  > 按照这样的描述，每个**命令的执行时间 = 客户端发送时间+服务器处理和返回时间+一个网络来回的时间**
  >
  > 其中一个网络来回的时间是不固定的，它的决定因素有很多，比如客户端到服务器要经过多少跳，网络是否拥堵等等。但是这个时间的量级也是最大的，也就是说一个命令的完成时间的长度很大程度上取决于网络开销。如果我们的服务器每秒可以处理10万条请求，而网络开销是250毫秒，那么实际上每秒钟只能处理4个请求。最暴力的优化方法就是使客户端和服务器在一台物理机上，这样就可以将网络开销降低到1ms以下。但是实际的生产环境并不会这样做。而且即使使用这种方法，当请求非常频繁时，这个时间和服务器处理时间比较仍然是很长的。

# 哈希槽

一个 Redis Cluster包含16384（0~16383）个哈希槽，存储在Redis Cluster中的所有键都会被映射到这些slot中，集群中的每个键都属于这16384个哈希槽中的一个。按照槽来进行分片，通过为每个节点指派不同数量的槽，可以控制不同节点负责的数据量和请求数。哈希槽（Hash Slot）是用于实现分布式存储和数据划分的一种机制。Redis的分布式特性主要通过哈希槽来实现。

<font color='Apricot'>**为什么是16384（2^14）个？**</font>

> 在redis节点发送心跳包时需要把所有的槽放到这个心跳包里，以便让节点知道当前集群信息，16384=16k，在发送心跳包时使用`char`进行bitmap压缩后是2k（`2 * 8 (8 bit) * 1024(1k) = 16K`），也就是说使用2k的空间创建了16k的槽数。
>
> 虽然使用CRC16算法最多可以分配65535（2^16-1）个槽位，65535=65k，压缩后就是8k（`8 * 8 (8 bit) * 1024(1k) =65K`），也就是说需要需要8k的心跳包，作者认为这样做不太值得；并且一般情况下一个redis集群不会有超过1000个master节点，所以16k的槽位是个比较合适的选择。

<font color='Apricot'>**哈希槽计算公式**</font>

> 在Redis中，哈希槽的计算通常是通过CRC16算法计算键名的哈希值，然后取模运算将其映射到一个哈希槽上。这样可以确保键名的分布性，避免了热点集中的问题。
>
> 集群使用公式slot=CRC16（key）/16384来计算key属于哪个槽，其中CRC16(key)语句用于计算key的CRC16 校验和

<font color='Apricot'>**哈希槽怎么工作**</font>

master节点在 Redis Cluster中的实现时，都存有所有的路由信息。当客户端的key 经过hash运算，发送slot 槽位不在本节点的时候：

- （1）如果是非集群方式连接，则直接报告错误给client，告诉它应该访问集群中哪个IP的master主机。
- （2）如果是集群方式连接，则将客户端重定向到正确的节点上。

<font color='Apricot'>**哈希槽的优点**</font>

1. **数据分片：** Redis将所有的数据划分成固定数量的哈希槽（通常是16384个）。每个键值对被映射到其中一个哈希槽上。这样就可以将整个数据集分布到多个节点上，实现分布式存储。

2. **节点分布：** Redis Cluster（集群模式）中的每个节点负责处理其中一部分哈希槽的数据。这样，集群中的每个节点都负责处理一部分数据，实现了负载均衡。

3. **数据迁移：** 当Redis集群中添加或删除节点时，数据迁移可以通过移动哈希槽来实现。这样可以确保数据在节点之间的均匀分布。

   > - 比如如果我想新添加个节点D, 我需要从节点 A, B, C中转移部分槽到D上即可.
   > - 如果我想移除节点A,需要将A中得槽移到B和C节点上,然后将没有任何槽的A节点从集群中移除即可.
   >
   > 由于从一个节点将哈希槽移动到另一个节点并不会停止服务,所以无论添加删除或者改变某个节点的哈希槽的数量都不会造成集群不可用的状态。

4. **数据定位：** 客户端在访问数据时，可以通过键名的哈希值计算出它属于哪个哈希槽，然后找到负责该哈希槽的节点进行数据操作。这种方式可以避免全局扫描，提高数据的查找效率。

<font color='Apricot'>**查看哈希槽分区情况**</font>

通过cluster nodes命令，可以清晰看到Redis Cluster中的每一个master节点管理的哈希槽。比如 127.0.0.1:7001 拥有哈希槽 0-5460， 127.0.0.1:7002 拥有哈希槽 5461-10922， 127.0.0.1:7003 拥有哈希槽 10923-16383。

```shell
//查看集群中，各个master节点的哈希槽分区情况
127.0.0.1:7001> cluster nodes
903322e4431 127.0.0.1:7001@17001 myself,master - 0 1590126182000 1 connected 0-5460
e51711eb03d 127.0.0.1:7002@17002 master - 0 1590126183862 2 connected 5461-10922
68c5fc14287 127.0.0.1:7003@17003 master - 0 1590126181856 3 connected 10923-16383
```

# 穿透、击穿、雪崩

//补充实际业务场景

##缓存穿透

缓存穿透是指频繁请求查询<font color='Apricot'>**缓存和数据库中都没有**</font>的数据

**解决方案：**

1. null结果缓存，并加入短暂过期时间
2. 采用布隆过滤器，将所有可能存在的数据哈希到一个足够大的 `bitmap `中，一个一定不存在的数据会被这个 `bitmap `拦截掉，从而避免了对底层存储系统的查询压力

##缓存击穿

缓存击穿是指<font color='Apricot'>**缓存中没有但数据库中有**</font>的数据（一般是缓存时间到期），一个热点的Key并发访问特别多，在它失效的瞬间，持续的大并发就穿破缓存，直接请求到了数据库。

 **解决方案：**

1. 如果业务允许的话，设置热点数据永远不过期。
2. 使用互斥锁。如果缓存失效的情况，只有拿到锁才可以查询数据库，降低了在同一时刻打在数据库上的请求，防止数据库打死。当然这样会导致系统的性能变差。

##缓存雪崩

缓存雪崩是指某一个时刻出现大规模的缓存失效的情况，导致大量请求直接访问数据库，导致数据库压力巨大，如果在高并发的情况下，可能瞬间就会导致数据库宕机。这时候如果运维马上又重启数据库，马上又会有新的流量把数据库打死。这就是缓存雪崩。<font color='Apricot'>**和缓存击穿不同的是，缓存击穿指并发查同一条数据，缓存雪崩是不同数据都过期了，很多数据都查不到从而查数据库**</font>。

<font color='Apricot'>原因：</font>

造成缓存雪崩的关键在于在**同一时间大规模的key失效**。出现这个问题有两种可能，第一种可能是`Redis`宕机，第二种可能是采用了相同的过期时间。

<font color='Apricot'>解决方案：</font>

1. 在原有的失效时间上加上一个随机值，比如1-5分钟随机。这样就避免了因为采用相同的过期时间导致的缓存雪崩。
2. 如果缓存数据库是分布式部署，将热点数据均匀分布在不同的缓存数据库中。同时，分布式集群可以防止Redis宕机导致缓存雪崩的问题。
3. 设置热点数据永远不过期。

<font color='Apricot'>**缓存雪崩的兜底措施**：</font>

1. 使用熔断机制。当流量到达一定的阈值时，就直接返回“系统拥挤”之类的提示，防止过多的请求打在数据库上。至少能保证一部分用户是可以正常使用，其他用户多刷新几次也能得到结果。

2. 提高数据库的容灾能力，可以使用分库分表，读写分离的策略。

# 事务

**`Redis`事务保证原子性吗，支持回滚吗？**

`Redis`中，单条命令是原子性执行的，但**事务不保证原子性，且没有回滚**。事务中任意命令执行失败，其余的命令仍会被执行。

# 内存淘汰

## <font color='VioletRed'>过期删除策略</font>

<font color='Peach'>定期删除+惰性删除</font>

> <font color='Apricot'>定期删除</font>：`Redis`默认是每隔` 100ms` 就<font color='Apricot'>随机抽取</font>一些设置了过期时间的key，检查其是否过期，如果过期就删除。注意这里随机抽取的原因是假如 `Rredis` 存了几十万个 `key` ，每隔`100ms`就遍历所有设置了过期时间的 `key` 的话，会给 `CPU` 带来很大的负载！
>
> <font color='Apricot'>惰性删除 </font>：定期删除可能会导致很多过期 `key` 到了时间并没有被删除掉。惰性策略就是在客户端访问这个`key`的时候，`redis`对`key`的过期时间进行检查，如果过期了就立即删除，不会给你返回任何东西。

## <font color='VioletRed'>数据淘汰策略</font>

1. no-envicition：该策略对于写请求不再提供服务，会直接返回错误，当然排除`del`等特殊操作，**`Redis`的默认策略**。

2. allkeys-lru：使用`LRU`（`Least Recently Used`，最近最少使用）算法，从`Redis`中选取最近最少使用的`key`进行淘汰

3. volatile-lru：使用`LRU`（`Least Recently Used`，最近最少使用）算法，从设置了过期时间的键集合中驱逐最近最少使用的键

4. allkeys-random：从所有key随机删除

5. volatile-random：从配置了过期时间的键的集合中进行随机淘汰

6. volatile-ttl：从配置了过期时间的键中淘汰马上就要过期的键

   <font color='Tasma'>在Redis 4.0以后，又增加了以下两种</font>

7. volatile-lfu：使用`LFU`（`Least Frequently Used`，最不经常使用），从所有配置了过期时间的键中驱逐使用频率最少的键

8. allkeys-lfu：使用`LFU`（`Least Frequently Used`，最不经常使用），从所有键中驱逐使用频率最少的键

<font color='VioletRed'>内存淘汰算法</font>

- 随机

- TTL：

  > 从设置了过期时间的 Keys 中获取最早过期的一批 Keys，然后淘汰这些 Keys

- LRU（Least Recently Used，最近最少使用）:

  > 所有的 Keys 都根据最后被访问的时间来进行排序的，所以在淘汰时只需要按照所有 Keys 的最后被访问时间，由小到大来进行即可；
  >
  > <font color='orange'>LRU算法的特点：</font>
  >
  > 1. 新增key value的时候首先在链表结尾添加Node节点，如果超过LRU设置的阈值就淘汰队头的节点并删除掉HashMap中对应的节点。
  > 2. 修改key对应的值的时候先修改对应的Node中的值，然后把Node节点移动队尾。
  > 3. 访问key对应的值的时候把访问的Node节点移动到队尾即可。

  <font color='blue'>Redis的LRU实现</font>

  > Redis维护了一个24位时钟，可以简单理解为当前系统的时间戳，每隔一定时间会更新这个时钟。每个key对象内部同样维护了一个24位的时钟，当新增key对象的时候会把系统的时钟赋值到这个内部对象时钟。比如我现在要进行LRU，那么首先拿到当前的全局时钟，然后再找到内部时钟与全局时钟距离时间最久的（差最大）进行淘汰，这里值得注意的是全局时钟只有24位，按秒为单位来表示才能存储194天，所以可能会出现key的时钟大于全局时钟的情况，如果这种情况出现那么就两个相加而不是相减来求最久的key。
  >
  > 
  >
  > Redis中的LRU与常规的LRU实现并不相同，常规LRU会准确的淘汰掉队头的元素，但是Redis的LRU并不维护队列，只是根据配置的策略要么从所有的key中随机选择N个（N可以配置）要么从所有的设置了过期时间的key中选出N个键，然后再从这N个键中选出最久没有使用的一个key进行淘汰。
  >
  > <font color='orange'>为什么要使用近似LRU？</font>
  >
  > 1、性能问题，由于近似LRU算法只是最多随机采样N个key并对其进行排序，如果精准需要对所有key进行排序，这样近似LRU性能更高
  >
  > 2、内存占用问题，redis对内存要求很高，会尽量降低内存使用率，如果是抽样排序可以有效降低内存的占用
  >
  > 3、实际效果基本相等，如果请求符合长尾法则，那么真实LRU与Redis LRU之间表现基本无差异
  >
  > 4、在近似情况下提供可自配置的取样率来提升精准度，例如通过 CONFIG SET maxmemory-samples 指令可以设置取样数，取样数越高越精准，如果你的CPU和内存有足够，可以提高取样数看命中率来探测最佳的采样比例。

- LFU（Least Frequently Used，最不经常使用）：

  > 它是根据数据的历史访问频率来淘汰数据，核心思想是“如果数据过去被访问多次，那么将来被访问的频率也更高”。LFU算法反映了一个key的热度情况，不会因LRU算法的偶尔一次被访问被误认为是热点数据。

  <font color='blue'>LFU算法的常见实现方式为链表：</font>

  新数据放在链表尾部 ，链表中的数据按照被访问次数降序排列，访问次数相同的按最近访问时间降序排列，链表满的时候从链表尾部移出数据。

#持久化

## RDB

RDB快照是某个时间点的一次全量数据备份，是二进制文件，在存储上非常紧凑。

> <font color='blue'>优点</font>
>
> - RDB文件小，非常适合定时备份，用于灾难恢复
> - Redis加载RDB文件的速度比AOF快很多，因为RDB文件中直接存储的时内存数据，而AOF文件中存储的是一条条命令，需要重演命令。
>
> <font color='blue'>缺点</font>
>
> - RDB无法做到实时持久化，若在两次bgsave间宕机，则会丢失区间（分钟级）的增量数据，不适用于实时性要求较高的场景
> - RDB的cow机制中，fork子进程属于重量级操作，并且会阻塞redis主进程  (Redis 使用操作系统的多进程 cow(Copy On Write) 机制来实现RDB快照持久化)
> - 存在老版本的Redis不兼容新版本RDB格式文件的问题

## AOF

AOF日志是持续增量的备份，是基于写命令存储的可读的文本文件。AOF日志会在持续运行中持续增大，由于Redis重启过程需要优先加载AOF日志进行指令重放以恢复数据，恢复时间会无比漫长。所以需要定期进行AOF重写，对AOF日志进行瘦身。

> 当两种方式同时开启时，数据恢复`Redis`会优先选择`AOF`恢复，因为`AOF`数据更新的频率更高，会保存更新的数据。

<font color='RedOrange'>重写（rewrite）机制</font>

> AOF日志会在持续运行中持续增大，需要定期进行AOF重写，对AOF日志进行瘦身。
>
> **AOF Rewrite** 虽然是“压缩”AOF文件的过程，但并非采用“基于原AOF文件”来重写或压缩，而是采取了类似RDB快照的方式：基于Copy On Write，全量遍历内存中数据，然后逐个序列到AOF文件中。因此AOF rewrite能够正确反应当前内存数据的状态。
>
> AOF重写（bgrewriteaof）和RDB快照写入（bgsave）过程类似，二者都消耗磁盘IO。Redis采取了“schedule”策略：无论是“人工干预”还是系统触发，快照和重写需要逐个被执行。
>
> 重写过程中，对于新的变更操作将仍然被写入到原AOF文件中，同时这些新的变更操作也会被Redis收集起来。当内存中的数据被全部写入到新的AOF文件之后，收集的新的变更操作也将被一并追加到新的AOF文件中。然后将新AOF文件重命名为appendonly.aof，使用新AOF文件替换老文件，此后所有的操作都将被写入新的AOF文件。

<font color='RedOrange'>优缺点</font>

> <font color='blue'>优点</font>： AOF只是追加写日志文件，对服务器性能影响较小，速度比RDB要快，消耗的内存较少
>
> <font color='blue'>缺点</font>：
>
> - AOF方式生成的日志文件太大，需要不断AOF重写，进行瘦身。
> - 即使经过AOF重写瘦身，由于文件是文本文件，文件体积较大（相比于RDB的二进制文件）。
> - AOF重演命令式的恢复数据，速度显然比RDB要慢。

## 混合持久化

- 仅使用RDB快照方式恢复数据，由于快照时间粒度较大，会丢失大量数据。
- 仅使用AOF重放方式恢复数据，日志性能相对 rdb 来说要慢。在 Redis 实例很大的情况下，启动需要花费很长的时间。

Redis 4.0 为了解决这个问题，带来了一个新的持久化选项——**混合持久化**。将 rdb 文件的内容和增量的 AOF 日志文件存在一起。这里的 AOF 日志不再是全量的日志，而是自持久化开始到持久化结束的这段时间发生的增量 AOF 日志，通常这部分 AOF 日志很小。相当于：

- 大量数据使用粗粒度（时间上）的rdb快照方式，性能高，恢复时间快。
- 增量数据使用细粒度（时间上）的AOF日志方式，尽量保证数据的不丢失。

在 Redis 重启的时候，可以先加载 rdb 的内容，然后再重放增量 AOF 日志就可以完全替代之前的 AOF 全量文件重放，重启效率因此大幅得到提升。

缺点：兼容性差，一旦开启了混合持久化，在4.0之前版本都不识别该aof文件，同时由于前部分是RDB格式，阅读性较差

#Redis高可用

在 `Web` 服务器中，**高可用** 是指服务器可以 **正常访问** 的时间，衡量的标准是在 **多长时间** 内可以提供正常服务（`99.9%`、`99.99%`、`99.999%` 等等）。在 `Redis` 层面，**高可用** 的含义要宽泛一些，除了保证提供 **正常服务**（如 **主从分离**、**快速容灾技术** 等），还需要考虑 **数据容量扩展**、**数据安全** 等等。

在 `Redis` 中，实现 **高可用** 的技术主要包括 **持久化**、**复制**、**哨兵** 和 **集群**

- **持久化**：持久化是 **最简单的** 高可用方法。它的主要作用是 **数据备份**，即将数据存储在 **硬盘**，保证数据不会因进程退出而丢失。
- **复制**：复制是高可用 `Redis` 的基础，**哨兵** 和 **集群** 都是在 **复制基础** 上实现高可用的。复制主要实现了数据的多机备份以及对于读操作的负载均衡和简单的故障恢复。缺陷是故障恢复无法自动化、写操作无法负载均衡、存储能力受到单机的限制。
- **哨兵**：在复制的基础上，哨兵实现了 **自动化** 的 **故障恢复**。缺陷是 **写操作** 无法 **负载均衡**，**存储能力** 受到 **单机** 的限制。
- **集群**：通过集群，`Redis` 解决了 **写操作** 无法 **负载均衡** 以及 **存储能力** 受到 **单机限制** 的问题，实现了较为 **完善** 的 **高可用方案**。

## 持久化

~~~markdown
持久化功能是 Redis 和 Memcached 的主要区别之一，因为只有 Redis 提供了此功能。

在 Redis 4.0 之前数据持久化方式有两种：AOF 方式和 RDB 方式。

	RDB（Redis DataBase，快照方式）是将某一个时刻的内存数据，以二进制的方式写入磁盘。AOF（Append Only File，文件追加方式）是指将所有的操作命令，以文本的形式追加到文件中。
	RDB 默认的保存文件为 dump.rdb，优点是以二进制存储的，因此占用的空间更小、数据存储更紧凑，并且与 AOF 相比，RDB 具备更快的重启恢复能力。

	AOF 默认的保存文件为 appendonly.aof，它的优点是存储频率更高，因此丢失数据的风险就越低，并且 AOF 并不是以二进制存储的，所以它的存储信息更易懂。缺点是占用空间大，重启之后的数据恢复速度比较慢。

可以看出 RDB 和 AOF 各有利弊，RDB 具备更快速的数据重启恢复能力，并且占用更小的磁盘空间，但有数据丢失的风险；而 AOF 文件的可读性更高，但却占用了更大的空间，且重启之后的恢复速度更慢，于是在 Redis 4.0 就推出了混合持久化的功能。

- 混合持久化的功能指的是 Redis 可以使用 RDB + AOF 两种格式来进行数据持久化，这样就可以做到扬长避短物尽其用了。 可以使用config get aof-use-rdb-preamble的命令来查询 Redis 混合持久化的功能是否开启
	Redis 混合持久化的存储模式是，开始的数据以 RDB 的格式进行存储，因此只会占用少量的空间，并且之后的命令会以 AOF 的方式进行数据追加，这样就可以减低数据丢失的风险，同时可以提高数据恢复的速度。
~~~

## 主从同步

~~~markdown
主从同步是 Redis 多机运行中最基础的功能，它是把多个 Redis 节点组成一个 Redis 集群，在这个集群当中有一个主节点用来进行数据的操作，其他从节点用于同步主节点的内容，并且提供给客户端进行数据查询。

Redis 主从同步分为：主从模式和从从模式。
	主从模式就是一个主节点和多个一级从节点,而从从模式是指一级从节点下面还可以拥有更多的从节点。 主从模式可以提高 Redis 的整体运行速度，因为使用主从模式就可以实现数据的读写分离，把写操作的请求分发到主节点上，把其他的读操作请	  求分发到从节点上，这样就减轻了 Redis 主节点的运行压力，并且提高了 Redis 的整体运行速度。
~~~

- `Redis` **主从复制** 可将 **主节点** 数据同步给 **从节点**，从节点此时有两个作用：
  1. 一旦 **主节点宕机**，**从节点** 作为 **主节点** 的 **备份** 可以随时顶上来。
  2. 扩展 **主节点** 的 **读能力**，分担主节点读压力。

### 实现方式

~~~Markdown
1. Redis 2.8 以前使用 SYNC 命令同步复制。
Redis的同步功能分为同步(sync)和命令传播(command propagate)。
1）同步操作：
	1. 通过从服务器发送到SYNC命令给主服务器；
	2. 主服务器生成RDB文件并发送给从服务器，同时发送保存所有写命令给从服务器；
	3. 从服务器清空之前数据并执行解释RDB文件；
	4. 保持数据一致（还需要命令传播过程才能保持一致）。
2）命令传播操作：
同步操作完成后，主服务器执行写命令，该命令发送给从服务器并执行，使主从保存一致。

缺陷：没有全量同步和增量同步的概念，从服务器在同步时，会清空所有数据。主从服务器断线后重复制，主服务器会重新生成RDB文件和重新记录缓冲区的所有命令，并全量同步到从服务器上。

2. 在Redis 2.8之后使用PSYNC命令，具备完整重同步和部分重同步模式。
	1. Redis 的主从同步，分为全量同步和增量同步；
	2. 只有从机第一次连接上主机是全量同步；
	3. 断线重连有可能触发全量同步也有可能是增量同步（ master 判断runid 是否一致）；
	4. 除此之外的情况都是增量同步；

3. 全量同步
Redis 的全量同步过程主要分三个阶段：
	1. 同步快照阶段： Master 创建并发送快照RDB给Slave ， Slave 载入并解析快照。Master 同时将此阶段所产生的新的写命令存储到缓冲区。
	2. 同步写缓冲阶段： Master 向Slave 同步存储在缓冲区的写操作命令；
	3. 同步增量阶段： Master 向Slave 同步写操作命令。

4. 增量同步
	1、Redis增量同步主要指Slave完成初始化后开始正常工作时， Master 发生的写操作同步到Slave 的过程；
	2、通常情况下， Master 每执行一个写命令就会向Slave 发送相同的写命令，然后Slave 接收并执行；

5. 命令传播
当同步数据完成后，主从服务器就会进入命令传播阶段，主服务器只要将自己执行的写命令发送给从服务器，而从服务器只要一直执行并接收主服务器发来的写命令。

6. 心跳检测
在命令传播阶段，从服务器默认会以每秒一次的频率向主服务器发送命令。
	作用：
	1. 检测主从的连接状态。检测主从服务器的网络连接状态通过向主服务器发送INFO replication命令，可以列出从服务器列表，可以看出从最后一次向主发送命令距离现在过了多少秒。lag的值应该在0或1之间跳动，如果超过1则说明主从之间的连接有故障。
	2. 辅助实现min-slaves：
		Redis可以通过配置防止主服务器在不安全的情况下执行写命令；
		min-slaves-to-write 3 （min-replicas-to-write 3 ）；
		min-slaves-max-lag 10 （min-replicas-max-lag 10）；
	上面的配置表示：从服务器的数量少于3个，或者三个从服务器的延迟（lag）值都大于或等于10秒时，主服务器将拒绝执行写命令。这里的延迟值就是上面INFOreplication命令的lag值。
	3. 检测命令丢失
		如果因为网络故障，主服务器传播给从服务器的写命令在半路丢失，那么当从服务器向主服务器发送REPLCONF ACK命令时，主服务器将发觉从服务器当前的复制偏移量少于自己的复制偏移量，然后主服务器就会根据从服务器提交的复制偏移	量，在复制积压缓冲区里面找到从服务器缺少的数据，并将这些数据重新发送给从服务器。
~~~

- **主从复制** 同时存在以下几个问题：
  1. 一旦 **主节点宕机**，**从节点** 晋升成 **主节点**，同时需要修改 **应用方** 的 **主节点地址**，还需要命令所有 **从节点** 去 **复制** 新的主节点，整个过程需要 **人工干预**。

  2. **主节点** 的 **写能力** 受到 **单机的限制**。
  3. **主节点** 的 **存储能力** 受到 **单机的限制**。
  4. **原生复制** 的弊端在早期的版本中也会比较突出，比如：`Redis` **复制中断** 后，**从节点** 会发起 `psync`。此时如果 **同步不成功**，则会进行 **全量同步**，**主库** 执行 **全量备份** 的同时，可能会造成毫秒或秒级的 **卡顿**。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220421180241.png" alt="image-20220421180239093" style="zoom:50%;" />

## 哨兵模式

在主从模式下，`redis`同时提供了哨兵命令`redis-sentinel`，哨兵是一个独立的进程，作为进程，它会独立运行。其原理是哨兵进程向所有的`redis`机器发送命令，等待`Redis`服务器响应，从而监控运行的多个`Redis`实例。

哨兵可以有多个，一般为了便于决策选举，使用奇数个哨兵。哨兵可以和`redis`机器部署在一起，也可以部署在其他的机器上。多个哨兵构成一个哨兵集群，哨兵之间也会相互通信，检查哨兵是否正常运行，同时发现`master`宕机哨兵之间会进行决策选举新的`master`

### 哨兵模式的作用:

- 通过发送命令，让`Redis`服务器返回监控其运行状态，包括主服务器和从服务器;
- 当哨兵监测到`master`宕机，会自动将`slave`切换到`master`，然后通过 *发布订阅模式*  通过其他的从服务器，修改配置文件，让它们切换主机;
- 然而一个哨兵进程对`Redis`服务器进行监控，也可能会出现问题，为此，我们可以使用多个哨兵进行监控。各个哨兵之间还会进行监控，这样就形成了多哨兵模式。

哨兵很像`kafka`集群中的`zookeeper`的功能。

### 哨兵模式的工作

每个 `Sentinel` 节点都需要 **定期执行** 以下任务：

1. 每个 `Sentinel` 以 **每秒钟** 一次的频率，向它所知的 **主服务器**、**从服务器** 以及其他 `Sentinel` **实例** 发送一个 `PING` 命令。

   <img src="https://gitee.com/qc_faith/picture/raw/master/image/202411170133044.png" alt="image-20241117013305934" style="zoom:67%;" />

2. 如果一个 **实例**（`instance`）距离 **最后一次** 有效回复 `PING` 命令的时间超过 `down-after-milliseconds` 所指定的值，那么这个实例会被 `Sentinel` 标记为 **主观下线**。

   <img src="https://gitee.com/qc_faith/picture/raw/master/image/202411170133646.png" alt="image-20241117013320534" style="zoom:67%;" />

3. 如果一个 **主服务器** 被标记为 **主观下线**，那么正在 **监视** 这个 **主服务器** 的所有 `Sentinel` 节点，要以 **每秒一次** 的频率确认 **主服务器** 的确进入了 **主观下线** 状态。

   <img src="https://gitee.com/qc_faith/picture/raw/master/image/202411170133498.png" alt="image-20241117013330463" style="zoom:67%;" />



4. 如果一个 **主服务器** 被标记为 **主观下线**，并且有 **足够数量** 的 `Sentinel`（至少要达到 **配置文件** 指定的数量）在指定的 **时间范围** 内同意这一判断，那么这个 **主服务器** 被标记为 **客观下线**。

   <img src="https://gitee.com/qc_faith/picture/raw/master/image/202411170133037.png" alt="image-20241117013347000" style="zoom:67%;" />

5. 在一般情况下， 每个 `Sentinel` 会以每 `10` 秒一次的频率，向它已知的所有 **主服务器** 和 **从服务器** 发送 `INFO` 命令。当一个 **主服务器** 被 `Sentinel` 标记为 **客观下线** 时，`Sentinel` 向 **下线主服务器** 的所有 **从服务器** 发送 `INFO` 命令的频率，会从 `10` 秒一次改为 **每秒一次**。

   <img src="https://gitee.com/qc_faith/picture/raw/master/image/202411170134210.png" alt="image-20241117013406106" style="zoom:67%;" />



6. `Sentinel` 和其他 `Sentinel` 协商 **主节点** 的状态，如果 **主节点** 处于 `SDOWN` 状态，则投票自动选出新的 **主节点**。将剩余的 **从节点** 指向 **新的主节点** 进行 **数据复制**。

   <img src="https://gitee.com/qc_faith/picture/raw/master/image/202411170134212.png" alt="image-20241117013420105" style="zoom:67%;" />

7. 当没有足够数量的 `Sentinel` 同意 **主服务器** 下线时， **主服务器** 的 **客观下线状态** 就会被移除。当 **主服务器** 重新向 `Sentinel` 的 `PING` 命令返回 **有效回复** 时，**主服务器** 的 **主观下线状态** 就会被移除。

   <img src="https://gitee.com/qc_faith/picture/raw/master/image/202411170134002.png" alt="image-20241117013433897" style="zoom:67%;" />

> 注意：一个有效的 `PING` 回复可以是：`+PONG`、`-LOADING` 或者 `-MASTERDOWN`。如果 **服务器** 返回除以上三种回复之外的其他回复，又或者在 **指定时间** 内没有回复 `PING` 命令， 那么 `Sentinel` 认为服务器返回的回复 **无效**（`non-valid`）。

- 每个`Sentinel`（哨兵）进程以每秒钟一次的频率向整个集群中的`Master`主服务器，`Slave`从服务器以及其他`Sentinel`（哨兵）进程发送一个 `PING `命令。
- 如果一个实例（instance）距离最后一次有效回复 `PING `命令的时间超过 `down-after-milliseconds` 选项所指定的值， 则这个实例会被 `Sentinel`（哨兵）进程标记为主观下线（SDOWN）
- 如果一个Master主服务器被标记为主观下线（SDOWN），则正在监视这个`Master`主服务器的所有 `Sentinel`（哨兵）进程要以每秒一次的频率确认`Master`主服务器的确进入了主观下线状态
- 当有足够数量的 `Sentinel`（哨兵）进程（大于等于配置文件指定的值）在指定的时间范围内确认`Master`主服务器进入了主观下线状态（SDOWN）， 则`Master`主服务器会被标记为客观下线（ODOWN）
- 在一般情况下， 每个 `Sentinel`（哨兵）进程会以每 10 秒一次的频率向集群中的所有Master主服务器、`Slave`从服务器发送 `INFO `命令。
- 当Master主服务器被 `Sentinel`（哨兵）进程标记为客观下线（ODOWN）时，`Sentinel`（哨兵）进程向下线的 `Master`主服务器的所有 `Slave`从服务器发送 `INFO `命令的频率会从 10 秒一次改为每秒一次。
- 若没有足够数量的 `Sentinel`（哨兵）进程同意 `Master`主服务器下线， `Master`主服务器的客观下线状态就会被移除。若 `Master`主服务器重新向 `Sentinel`（哨兵）进程发送 `PING `命令返回有效回复，`Master`主服务器的主观下线状态就会被移除。

~~~markdown
假设`master`宕机，`sentinel 1`先检测到这个结果，系统并不会马上进行 `failover`(故障转移)选出新的`master`，仅仅是`sentinel 1`主观的认为`master`不可用，这个现象成为**主观下线**。当后面的哨兵也检测到主服务器不可用，并且数量达到一定值时，那么哨兵之间就会进行一次投票，投票的结果由`sentinel 1`发起，进行 `failover `操作。切换成功后，就会通过发布订阅模式，让各个哨兵把自己监控的从服务器实现切换主机，这个过程称为**客观下线**。这样对于客户端而言，一切都是透明的。
~~~

### 哨兵模式的优缺点

> 优点
>
> - 哨兵模式是基于主从模式的，所有主从的优点，哨兵模式都具有。
> - 主从可以自动切换，系统更健壮，可用性更高。
>
> 缺点
>
> - 具有主从模式的缺点，每台机器上的数据是一样的，内存的可用性较低。
> - `Redis`较难支持在线扩容，在集群容量达到上限时在线扩容会变得很复杂。

## 集群与分区

> **分区的概念**：分区是将数据分布在多个Redis实例（Redis主机）上，以至于每个实例只包含一部分数据。
>
> **分区的意义**：
> 1、**性能的提升**：单机Redis的网络I/O能力和计算资源是有限的，将请求分散到多台机器，充分利用多台机器的计算能力可网络带宽，有助于提高Redis总体的服务能力；
> 2、**存储能力的横向扩展**：随着存储数据的增加，单台机器受限于机器本身的存储容量，将数据分散到多台机器上存储使得Redis服务可以横向扩展。

### client端分区

>  对于一个给定的key，客户端直接选择正确的节点来进行读写。许多Redis客户端都实现了客户端分区(JedisPool)，也可以自行编程实现。
>
>  **分区算法（hash）**
>
>  1. **普通hash**：hash(key)%N，其中hash可以使用hash算法比如CRC32、CRC16等，N是rerdis的节点数
>     **优点**：实现简单，热点数据分布均匀
>     **缺点**：节点数固定，扩展的话需要重新计算查询时必须用分片的key来查，一旦key改变，数据就查不出了，所以要使用不易改变的key进行分片。
>
>  2. **一致性hash**
>
>     我们把2^32^想象成一个圆，hash（服务器的IP地址） % 2^32^，通过上述公式算出的结果一定是一个0到2^32-1^，之间的一个整数，我们就用算出的这个整数，代表服务器A、服务器B、服务器C，既然这个整数肯定处于0到2^32-1^之间，那么，上图中的hash环上必定有一个点与这个整数对应，也就是服务器A、服务器B、服务C就可以映射到这个环上，当我们对一个key做hash运算之后hash（key） % 2^32^得到的值也必然在这个环上，按照顺时针旋转，如果找到的第一个节点就是数据需要存储的节点。
>
>     **优点**：添加或移除节点时，数据只需要做部分的迁移，比如上图中把C服务器移除，则数据4迁移到服务器A中，而其他的数据保持不变。添加效果是一样的。
>     **缺点**：可能会出现hash环偏移（数据倾斜）。解决方案：增加虚拟节点，实现复杂。

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220421204357.png" alt="image-20220421204356596" style="zoom:50%;" />

### 官方cluster分区

> Redis3.0之后，Redis官方提供了完整的集群解决方案，方案采用去中心化的方式，包括：sharding（分区）、replication（复制）、failover（故障转移），称为`RedisCluster`。`Redis5.0`可以直接使用`Redis-cli`进行集群的创建和管理
>
> **集群部署**
>
> **Gossip协议**：Gossip协议是一个通信协议，一种传播消息的方式。通过gossip协议，cluster可以提供集群间状态同步更新、选举自助failover等重要的集群功能。
> **Gossip协议基本思想**：一个节点周期性(每秒)随机选择一些节点，并把信息传递给这些节点。这些收到信息的节点接下来会做同样的事情，即把这些信息传递给其他一些随机选择的节点。信息会周期性的传递给N个目标节点。这个N被称为fanout（扇出）

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220421204536.png" alt="image-20220421204535595" style="zoom:50%;" />

**Gossip的消息种类**：

| 命令    | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| meet    | sender 向 receiver发出，请求 receiver 加入 sender 的集群     |
| ping    | 节点检测其他节点是否在线                                     |
| pong    | receiver 收到 meet 或 ping 后的回复信息，在 failover 后新的 master 也会广播 pong |
| fail    | 节点 A 判断节点 B 下线后，A 节点广播 B节点的 fail 信息，其他受到节点会将 B 节点标记为下线 |
| publish | 节点 A 收到 publish 命令，节点 A 执行改命令，并向集群广播 publish 命令，收到 publish 命令的节点都会执行相同的 publish 命令 |

> **slot**：redis-cluster把所有的物理节点映射到[0-16383]个slot上，集群中的节点基本上采用平均分配和连续分配的方式。cluster 负责维护节点和slot槽的对应关系
> value------>slot-------->节点
> 当需要在 Redis 集群中放置一个 key-value 时，redis 先对 key 使用 crc16 算法算出一个结果，然后把结果对 16384 求余数，这样每个 key 都会对应一个编号在 0-16383 之间的哈希槽，redis 会根据节点数量大致均等的将哈希槽映射到不同的节点。slot槽必须在节点上连续分配，如果出现不连续的情况，则RedisCluster不能工作

### RedisCluster的优势

~~~markdown
1、高性能：Redis Cluster 的性能与单节点部署是同级别的。可实现多主节点、负载均衡、读写分离。
2、高可用：Redis Cluster 支持标准的 主从复制配置来保障高可用和高可靠。可以做到故障转移，并且也实现了一个类似 Raft 的共识方式，来保障整个集群的可用性。
3、易扩展：向 Redis Cluster 中添加新节点，或者移除节点，都是透明的，不需要停机。
4、原生态：部署 Redis Cluster 不需要其他的代理或者工具，而且 Redis Cluster 和单机 Redis 几乎完全兼容。

集群配置：在redis.conf中配置cluster-enable yes即可。
创建集群命令：./redis-cli --cluster create ipList --cluster-replicas 1

分片：不同节点分组服务于相互无交集的分片（sharding），Redis Cluster 不存在单独的proxy或配置服务器，所以需要将客户端路由到目标的分片。
客户端路由：Redis Cluster的客户端相比单机Redis 需要具备路由语义的识别能力，且具备一定的路由缓存能力。
~~~

### moved重定向

~~~markdown
1. 每个节点通过通信都会共享Redis Cluster中槽和集群中对应节点的关系；
2. 客户端向Redis Cluster的任意节点发送命令，接收命令的节点会根据CRC16规则进行hash运算与16384取余，计算自己的槽和对应节点；
3. 如果保存数据的槽被分配给当前节点，则去槽中执行命令，并把命令执行结果返回给客户端；
4. 如果保存数据的槽不在当前节点的管理范围内，则向客户端返回moved重定向异常；
5. 客户端接收到节点返回的结果，如果是moved异常，则从moved异常中获取目标节点的信息；
6. 客户端向目标节点发送命令，获取命令执行结果。
~~~

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220421205828.png" alt="image-20220421205827032" style="zoom: 67%;" />

### ask重定向

~~~markdown
在对集群进行扩容和缩容时，需要对槽及槽中数据进行迁移。当客户端向某个节点发送命令，节点向客户端返回moved异常，告诉客户端数据对应的槽的节点信息，如果此时正在进行集群扩展或者缩空操作，当客户端向正确的节点发送命令时，槽及槽中数据已经被迁移到别的节点了，就会返回ask，这就是ask重定向机制
1.客户端向目标节点发送命令，目标节点中的槽已经迁移支别的节点上了，此时目标节点会返回ask转向给客户端；
2.客户端向新的节点发送Asking命令给新的节点，然后再次向新节点发送命令；
3.新节点执行命令，把命令执行结果返回给客户端。
~~~

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220421205927.png" alt="image-20220421205926172" style="zoom:67%;" />

~~~
moved和ask的区别：

1、moved：槽已确认转移；
2、ask：槽还在转移过程中。
~~~

### 迁移

~~~markdown
在RedisCluster中每个slot 对应的节点在初始化后就是确定的。已下情况，节点和分片需要变更：
1. master加入；
2. 某个节点分组需要下线；
3. 负载不均衡需要调整slot 分布。

此时需要进行分片的迁移，迁移的触发和过程控制由外部系统完成，主要分为两大部位：
1. 节点迁移状态设置：迁移前标记源/目标节点；
2. key迁移的原子化命令：数据迁移。
~~~

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220502162730.png" alt="image-20220421210220096" style="zoom:67%;" />

~~~markdown
数据迁移又分为4个步骤：
1. 向节点B发送状态变更命令，将B的对应slot 状态置为importing；
2. 向节点A发送状态变更命令，将A对应的slot 状态置为migrating；
3. 向A 发送migrate 命令，告知A 将要迁移的slot对应的key 迁移到B；
4. 当所有key 迁移完成后，cluster setslot 重新设置槽位。

**扩容步骤：**
1、添加需要加入的redis的master节点；
2、使用--cluster add-node命令将刚添加的redis节点加入到集群中；
3、使用--cluster reshard命令给新加入集群的节点分配hash槽（指定从哪个节点分配多少个槽给新节点）；

**缩容步骤：**
1、先确定需要删除的节点是否还有槽被使用，如果在使用，应先分配出去。
2、使用--cluster del-node命令删除节点；
~~~

### 容灾

> **故障检测**
> 集群中的每个节点都会定期地（每秒）向集群中的其他节点发送PING消息，如果在一定时间内(cluster-node-timeout)，发送ping的节点A没有收到某节点B的pong回应，则A将B标识为pfail。A在后续发送ping时，会带上B的pfail信息， 通知给其他节点。如果B被标记为pfail的个数大于集群主节点个数的一半（N/2 + 1）时，B会被标记为fail，A向整个集群广播，该节点已经下线，其他节点收到广播，标记B为fail。
>
> **从节点选举（自动切换主从）**
>
> 选举使用raft算法，每个从节点都根据自己对master复制数据的offset，来设置一个选举时间，offset越大（复制数据越多）的从节点，选举时间越靠前，优先进行选举。slave将currentEpoch 自增并 通过向其他master发送FAILVOER_AUTH_REQUEST 消息发起竞选，master 收到后回复FAILOVER_AUTH_ACK 消息告知是否同意，，如果自己未投过票，则回复同意，否则回复拒绝。
> 所有的Master开始slave选举投票，给要进行选举的slave进行投票，如果大部分master node（N/2 +1）都投票给了某个从节点，那么选举通过，那个从节点可以切换成master。
>
> **RedisCluster失效的判定**：
> 1、集群中半数以上的主节点都宕机（无法投票）；
> 2、宕机的主节点的从节点也宕机了（slot槽分配不连续）。
>
> **变更通知**
>
> 当slave 收到过半的master 同意时，会成为新的master。此时会以最新的Epoch 通过PONG 消息广播自己成为master，让Cluster 的其他节点尽快的更新拓扑结构(node.conf)。
>
> **手动主从切换**
>
> 人工故障切换是预期的操作，而非发生了真正的故障，目的是以一种安全的方式(数据无丢失)将当前master节点和其中一个slave节点(执行cluster-failover的节点)交换角色
>
> 手动主从切换分两种情况：**master还在线
> 
> **1、向从节点发送cluster failover 命令（slaveof no one）；
> 2、从节点告知其主节点要进行手动切换（CLUSTERMSG_TYPE_MFSTART）；
> 3、主节点会阻塞所有客户端命令的执行（10s）；
> 4、从节点从主节点的ping包中获得主节点的复制偏移量；
> 5、从节点复制达到偏移量，发起选举、统计选票、赢得选举、升级为主节点并更新配置；
> 6、切换完成后，原主节点向所有客户端发送moved指令重定向到新的主节点。
>
> **master已经离线**：
> 如果主节点下线了，则采用cluster failover force或cluster failover takeover 进行强制切换。

### 副本漂移

> 在集群中，如果某个master没有slave了，可以自动从集群中的slave多的master，选择其中一台slave作为新master的slave
>
> 在集群中如果有的master的slave多，有的master的slave少（比如只有一台slave），这时候如果只有一台slave的master挂掉了，经过主从切换之后，新的master就没有slave了，这时候是比较不安全的，如果新的master挂了，整个集群就不完整了。这时候可以使用副本漂移；

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20220502162731.png" alt="image-20220421210532048" style="zoom:50%;" />

> 如图：
> 如果master1宕机，则slave11会成为新的master，但是这时候他没有slave则可以使用副本漂移策略：
>
> 1. 将Slaver31的从机记录从Master3中删除；
>
> 2. 将Slaver31的的主机改为Master1；
>
> 3. 在Master1中添加Slaver31为从节点；
>
> 4. 将Slaver31的复制源改为Master1；
>
> 5. 通过ping包将信息同步到集群的其他节点。

### 优缺点

> ### **优点**
>
> 采用去中心化思想，数据按照 slot 存储分布在多个节点，节点间数据共享，可动态调整数据分布;
>
> 可扩展性：可线性扩展到 1000 多个节点，节点可动态添加或删除;
>
> 高可用性：部分节点不可用时，集群仍可用。通过增加 Slave 做 standby 数据副本，能够实现故障自动 failover，节点之间通过 gossip 协议交换状态信息，用投票机制完成 Slave 到 Master 的角色提升;
>
> 降低运维成本，提高系统的扩展性和可用性。
>
> ### **缺点**
>
> 1.Redis Cluster是无中心节点的集群架构，依靠Goss协议(谣言传播)协同自动化修复集群的状态
>
> ​	但 GosSIp有消息延时和消息冗余的问题，在集群节点数量过多的时候，节点之间需要不断进行 PING/PANG通讯，不必须要的流量占用了大量的网络资源。虽然Reds4.0对此进行了优化，但这个问题仍然存在
>
> 2.数据迁移问题
>
> ​	Redis Cluster可以进行节点的动态扩容缩容，这一过程，在目前实现中，还处于半自动状态，需要人工介入。在扩缩容的时候，需要进行数据迁移。
>
> ​	而 Redis为了保证迁移的一致性，迁移所有操作都是同步操作，执行迁移时，两端的 Redis均会进入时长不等的阻塞状态，对于小Key，该时间可以忽略不计，但如果一旦Key的内存使用过大，严重的时候会接触发集群内的故障转移，造成不必要的切换。

## 总结

主从模式：master节点挂掉后，需要手动指定新的master，可用性不高，基本不用。

哨兵模式：master节点挂掉后，哨兵进程会主动选举新的master，可用性高，但是每个节点存储的数据是一样的，浪费内存空间。数据量不是很多，集群规模不是很大，需要自动容错容灾的时候使用。

集群模式：数据量比较大，QPS要求较高的时候使用。 **Redis Cluster是Redis 3.0以后才正式推出，时间较晚，目前能证明在大规模生产环境下成功的案例还不是很多，需要时间检验。

#应用场景

**总结一**

1. 计数器

   可以对 `String` 进行自增自减运算，从而实现计数器功能。`Redis` 这种内存型数据库的读写性能非常高，很适合存储频繁读写的计数量。

2. 缓存

   将热点数据放到内存中，设置内存的最大使用量以及淘汰策略来保证缓存的命中率。

3. 会话缓存

   可以使用 `Redis` 来统一存储多台应用服务器的会话信息。当应用服务器不再存储用户的会话信息，也就不再具有状态，一个用户可以请求任意一个应用服务器，从而更容易实现高可用性以及可伸缩性。

4. 全页缓存（FPC）

   除基本的会话`token`之外，`Redis`还提供很简便的`FPC`平台。

5. 查找表

   例如 `DNS` 记录就很适合使用 `Redis` 进行存储。查找表和缓存类似，也是利用了 `Redis` 快速的查找特性。但是查找表的内容不能失效，而缓存的内容可以失效，因为缓存不作为可靠的数据来源。

6. 消息队列(发布/订阅功能)

   `List` 是一个双向链表，可以通过 `lpush` 和 `rpop` 写入和读取消息。不过最好使用 Kafka、RabbitMQ 等消息中间件。

7. 分布式锁实现

   在分布式场景下，无法使用单机环境下的锁来对多个节点上的进程进行同步。可以使用 `Redis` 自带的 `SETNX` 命令实现分布式锁，除此之外，还可以使用官方提供的 `RedLock` 分布式锁实现。

8. 其它

   `Set` 可以实现交集、并集等操作，从而实现共同好友等功能。`ZSet` 可以实现有序性操作，从而实现排行榜等功能。

**总结二**

`Redis`相比其他缓存，有一个非常大的优势，就是支持多种数据类型。

`string`——适合最简单的`k-v`存储，类似于`memcached`的存储结构，短信验证码，配置信息等，就用这种类型来存储。

`hash`——一般`key`为`ID`或者唯一标示，`value`对应的就是详情了。如商品详情，个人信息详情，新闻详情等。

`list`——因为`list`是有序的，比较适合存储一些有序且数据相对固定的数据。如省市区表、字典表等。因为`list`是有序的，适合根据写入的时间来排序，如：消息队列等。

`set`——可以简单的理解为`ID-List`的模式，如微博中一个人有哪些好友，`set`最牛的地方在于，可以对两个`set`提供交集、并集、差集操作。例如：查找两个人共同的好友等。

`Sorted Set`——是`set`的增强版本，增加了一个`score`参数，自动会根据`score`的值进行排序。比较适合类似于`top 10`等不根据插入的时间来排序的数据。


---
layout: posts
title: 我永远学不会系列---Redis(二)
date: 2018-08-15 00:38:36
categories: Redis
tags: [Redis]
top: false
---
这篇就来聊聊Redis的一些相关问题，凡是用过Redis的，难免会遇到各种各样的问题，缓存穿透、雪崩、如何才能保证Redis和数据库的双写一致性，Redis的并发竞争问题等等。其实要说在生产环境，我们目前用的我也不知道是啥缓存，好像是基于Weblogic的一套缓存机制，每次升级重启系统都要去Weblogic下边手动删除文件夹，感觉好难受呦，一般Redis我都是自己做项目时候使用，结合SpringBoot使用简直不要太简单，会用归会用，Redis的一些原理还是要懂滴。凭啥人家单线程就能这么牛逼呢？？

<!--more--> 

#### 一、我为啥要用Redis

其实用缓存，主要是俩用途，保证系统的高性能和高并发。

##### 1：有关高性能

假设这么个场景，有个操作，一个请求过来，你各种乱七八糟操作MySQL，半天查出来一个结果，耗时600ms。但是这个结果可能接下来几个小时都不会变了，或者变了也可以不用立即反馈给用户。那么此时咋办？

缓存啊，折腾600ms查出来的结果，扔缓存里，一个key对应一个value，下次再有人查，别走MySQL折腾600ms了。直接从缓存里，通过一个key查出来一个value，2ms搞定。性能提升300倍。

这就是所谓的高性能。

> 把一些复杂操作耗时查出来的结果，如果确定后面不咋变了，然后但是马上还有很多读请求，那么直接结果放缓存，后面直接读缓存就好了。

##### 2：有关高并发

MySQL这么重的数据库，压根儿设计不是让你玩儿高并发的，虽然也可以玩儿，但是天然支持不好。MySQL单机支撑到2000qps也开始容易报警了。

所以要是有个系统，高峰期一秒钟过来的请求有1万，那一个MySQL单机绝对会死掉。这个时候就只能上缓存，把很多数据放缓存，别放MySQL。缓存功能简单，说白了就是key-value式操作，单机支撑的并发量轻松一秒几万十几万，支撑高并发so easy。单机承载并发量是MySQL单机的几十倍。

##### 3：缓存的缺点了解下？

缓存与数据库双写不一致、缓存雪崩、缓存穿透、缓存并发竞争等等吧。

#### 二、Redis的过期策略及内存淘汰机制

##### 1：Redis的过期策略

在set key的时候，都可以给一个expire time，就是过期时间，指定这个key比如说只能存活1个小时？10分钟？这个很有用，我们自己可以指定缓存到期就失效。

如果假设你设置一个一批key只能存活1个小时，那么接下来1小时后，Redis是怎么对这批key进行删除的？

答案是：**定期删除+惰性删除**

所谓定期删除，指的是Redis默认是每隔100ms就随机抽取一些设置了过期时间的key，检查其是否过期，如果过期就删除。假设Redis里放了10万个key，都设置了过期时间，你每隔几百毫秒，就检查10万个key，那Redis基本上就死了，cpu负载会很高的，消耗在你的检查过期key上了。注意，这里可不是每隔100ms就遍历所有的设置过期时间的key，那样就是一场性能上的灾难。实际上Redis是每隔100ms随机抽取一些key来检查和删除的。

但是问题是，定期删除可能会导致很多过期key到了时间并没有被删除掉，那咋整呢？所以就是惰性删除了。这就是说，在获取某个key的时候，Redis会检查一下 ，这个key如果设置了过期时间那么是否过期了？如果过期了此时就会删除，不会给你返回任何东西。

**并不是key到时间就被删除掉，而是查询这个key的时候，Redis再懒惰的检查一下，是为惰性删除。**

很简单，就是说，你的过期key，靠定期删除没有被删除掉，还停留在内存里，占用着你的内存呢，除非你的系统去查一下那个key，才会被Redis给删除掉。

但是实际上这还是有问题的，如果定期删除漏掉了很多过期key，然后你也没及时去查，也就没走惰性删除，此时会怎么样？如果大量过期key堆积在内存里，导致Redis内存块耗尽了，咋整？

答案是：**走内存淘汰机制**。

##### 2：Redis的内存淘汰机制

Redis内存淘汰指的是用户存储的一些键被可以被Redis主动地从实例中删除，从而产生读miss的情况，Redis最常见的两种应用场景为缓存和持久存储，首先要明确的一个问题是内存淘汰策略更适合于那种场景？是持久存储还是缓存？

**内存的淘汰机制的初衷是为了更好地使用内存，用一定的缓存miss来换取内存的使用效率。**

> 例如：Redis 10个key，现在已经满了，Redis需要删除掉5个key
>
> 1个key，最近1分钟被查询了100次
>
> 1个key，最近10分钟被查询了50次
>
> 1个key，最近1个小时倍查询了1次（一个小时才查一次，肯定删它啊）

有如下一些策略：

我们可以通过配置redis.conf中的`maxmemory`这个值来开启内存淘汰功能

```
# maxmemory <bytes>
```

- 客户端发起了需要申请更多内存的命令（如set）。
- Redis检查内存使用情况，如果已使用的内存大于maxmemory则开始根据用户配置的不同淘汰策略来淘汰内存（key），从而换取一定的内存。
- 如果上面都没问题，则这个命令执行成功。

maxmemory为`0`的时候表示我们对Redis的内存使用没有限制。

Redis提供了下面几种淘汰策略供用户选择，其中默认的策略为noeviction策略

- `noeviction`：当内存使用达到阈值的时候，所有引起申请内存的命令会报错，这个一般没人用。
- `allkeys-lru`：在主键空间中，优先移除最近未使用的key（这个是最常用的）。
- `allkeys-random`：在主键空间中，随机移除某个key，这个一般没人用。
- `volatile-lru`：在设置了过期时间的键空间中，优先移除最近未使用的key（这个一般不太合适）
- `volatile-random`：在设置了过期时间的键空间中，随机移除某个key。
- `volatile-ttl`：在设置了过期时间的键空间中，具有更早过期时间的key优先移除。

> 说一下有关主键空间和设置了过期时间的键空间，举个例子，假设我们有一批键存储在Redis中，则有那么一个哈希表用于存储这批键及其值，如果这批键中有一部分设置了过期时间，那么这批键还会被存储到另外一个哈希表中，这个哈希表中的值对应的是键被设置的过期时间。设置了过期时间的键空间为主键空间的子集。

淘汰策略的选择可以通过下面的配置指定：

```
# maxmemory-policy noeviction
```

各个策略的使用场景：

- `allkeys-lru`：如果应用对缓存的访问符合幂律分布（也就是存在相对热点数据），或者我们不太清楚我们应用的缓存访问分布状况，我们可以选择allkeys-lru策略。
- `allkeys-random`：如果我们的应用对于缓存key的访问概率相等，则可以使用这个策略。
- `volatile-ttl`：这种策略使得我们可以向Redis提示哪些key更适合被eviction。

另外，`volatile-lru`策略和`volatile-random`策略适合我们将一个Redis实例既应用于缓存和又应用于持久化存储的时候，然而我们也可以通过使用两个Redis实例来达到相同的效果，值得一提的是将key设置过期时间实际上会消耗更多的内存，因此我们建议使用`allkeys-lru`策略从而更有效率的使用内存。

#### 三、Redis的集群模式

从redis 3.0之后版本支持**redis-cluster**集群，Redis-Cluster采用无中心结构，每个节点保存数据和整个集群状态,每个节点都和其他所有节点连接。

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx0sN2AUsjdAACjglDFAB8349.jpg)

##### 1：集群模式结构特点

> A.所有的redis节点彼此互联(PING-PONG机制)，内部使用二进制协议优化传输速度和带宽。
> B.节点的fail是通过集群中超过半数的节点检测失效时才生效。
> C.客户端与redis节点直连,不需要中间proxy层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可。
> D.redis-cluster把所有的物理节点映射到[0-16383]slot上（不一定是平均分配）,cluster 负责维护node<->slot<->value。
>
> E.Redis集群预分好16384个桶，当需要在 Redis 集群中放置一个 key-value 时，根据 CRC16(key) mod 16384的值，决定将一个key放到哪个桶中。

##### 2：redis cluster节点分配

现在我们是三个主节点分别是：A, B, C 三个节点，它们可以是一台机器上的三个端口，也可以是三台不同的服务器。那么，采用哈希槽 (hash slot)的方式来分配16384个slot 的话，它们三个节点分别承担的slot 区间是：

> 节点A覆盖0－5460;
> 节点B覆盖5461－10922;
> 节点C覆盖10923－16383.

如果存入一个值，按照redis cluster哈希槽的算法： CRC16(‘key’)384 = 6782。 那么就会把这个key 的存储分配到 B 上了。同样，当我连接(A,B,C)任何一个节点想获取’key’这个key时，也会这样的算法，然后内部跳转到B节点上获取数据。

新增一个节点D，redis cluster的这种做法是从各个节点的前面各拿取一部分slot到D上，大致就会变成这样：

> 节点A覆盖1365-5460
> 节点B覆盖6827-10922
> 节点C覆盖12288-16383
> 节点D覆盖0-1364,5461-6826,10923-12287

##### 3：Redis Cluster主从模式

redis cluster 为了保证数据的高可用性，加入了主从模式，一个主节点对应一个或多个从节点，主节点提供数据存取，从节点则是从主节点拉取数据备份，当这个主节点挂掉后，就会有这个从节点选取一个来充当主节点，从而保证集群不会挂掉。

上面那个例子里, 集群有ABC三个主节点, 如果这3个节点都没有加入从节点，如果B挂掉了，我们就无法访问整个集群了。A和C的slot也无法访问。

所以我们在集群建立的时候，一定要为每个主节点都添加了从节点, 比如像这样, 集群包含主节点A、B、C, 以及从节点A1、B1、C1, 那么即使B挂掉系统也可以继续正确工作。

B1节点替代了B节点，所以Redis集群将会选择B1节点作为新的主节点，集群将会继续正确地提供服务。 当B重新开启后，它就会变成B1的从节点。

不过需要注意，如果节点B和B1同时挂了，Redis集群就无法继续正确地提供服务了。

##### 4：Redis集群模式 VS 哨兵模式

如果你的数据量很少，主要是承载高并发高性能的场景，比如你的缓存一般就几个G，单机足够了。

replication，一个mater，多个slave，要几个slave跟你的要求的读吞吐量有关系，然后自己搭建一个sentinal集群，去保证Redis主从架构的高可用性，就可以了。

Redis cluster，主要是针对海量数据+高并发+高可用的场景，海量数据，如果你的数据量很大，那么建议就用Redis cluster。

##### 3：Redis的数据分布算法

在Redis cluster架构下，每个Redis要放开两个端口号，比如一个是`6379`，另外一个就是`加10000`的端口号，比如16379。

16379端口号是用来进行节点间通信的，也就是cluster bus的东西，集群总线。cluster bus的通信，用来进行故障检测，配置更新，故障转移授权。

cluster bus用了另外一种二进制的协议，主要用于节点间进行高效的数据交换，占用更少的网络带宽和处理时间。

Redis cluster的`hash slot`算法：

> Redis cluster有固定的16384个hash slot，对每个key计算CRC16值，然后对16384取模，可以获取key对应的hash slot。
>
> Redis cluster中每个master都会持有部分slot，比如有3个master，那么可能每个master持有5000多个hash slot。
>
> hash slot让node的增加和移除很简单，增加一个master，就将其他master的hash slot移动部分过去，减少一个master，就将它的hash slot移动到其他master上去，移动hash slot的成本是非常低的。客户端的api，可以对指定的数据，让他们走同一个hash slot，通过hash tag来实现。
>
> 任何一台机器宕机，另外的节点不受影响的，因为key找的是hash slot，找的不是机器。

#### 四、Redis的雪崩及穿透

##### 1：有关缓存雪崩：

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx010OAF6n0AAGqGnFXb_4440.png)

啥是雪崩：

因为缓存层承载了大量的请求，有效的保护了存储 层，但是如果缓存由于某些原因，整体不能够提供服务，于是所有的请求，就会到达存储层，存储层的调用量就会暴增，造成存储层也会挂掉的情况。

缓存雪崩的事前事中事后的解决方案：

事前：Redis高可用，主从+哨兵，Redis cluster，避免全盘崩溃。

事中：本地ehcache缓存 + hystrix限流&降级，避免MySQL被打死。

事后：Redis持久化，快速恢复缓存数据。

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_FxzqAWASBrtAAJADY_cRrc191.png)

##### 2：缓存穿透

一般的缓存系统，都是按照key值去缓存查询，如果不存在对应的value，就应该去DB中查找 。这个时候，如果请求的并发量很大，就会对后端的DB系统造成很大的压力。这就叫做缓存穿透。关键词：`缓存value为空`；`并发量很大去访问DB`。

造成的原因：

- 业务自身代码或数据出现问题；
- 一些恶意攻击、爬虫造成大量空的命中，此时会对数据库造成很大压力

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_FxzqGSAJBfdAADxLYWfm24569.png)

解决方案：

- 设置布隆过滤器，将所有可能存在的数据哈希到一个足够大的bitmap中，一个一定不存在的数据会被这个bitmap拦截掉，从避免了对底层存储系统的查询压力。
- 如果一个查询返回的数据为空，不管是数据不存在还是系统故障，我们仍然把这个结果进行缓存，但是它的过期时间会很短最长不超过5分钟。

#### 五、如何保证缓存和数据库双写一致性

最经典的缓存+数据库读写的模式，cache aside pattern

##### 1：Cache Aside Pattern

（1）读的时候，先读缓存，缓存没有的话，那么就读数据库，然后取出数据后放入缓存，同时返回响应

（2）更新的时候，先删除缓存，然后再更新数据库

##### 2：为什么是删除缓存，而不是更新缓存呢？

原因很简单，很多时候，复杂点的缓存的场景，因为缓存有的时候，不简单是数据库中直接取出来的值。

商品详情页的系统，修改库存，只是修改了某个表的某些字段，但是要真正把这个影响的最终的库存计算出来，可能还需要从其他表查询一些数据，然后进行一些复杂的运算，才能最终计算出。

现在最新的库存是多少，然后才能将库存更新到缓存中去。

比如可能更新了某个表的一个字段，然后其对应的缓存，是需要查询另外两个表的数据，并进行运算，才能计算出缓存最新的值的。

更新缓存的代价是很高的。

是不是说，每次修改数据库的时候，都一定要将其对应的缓存去更新一份？也许有的场景是这样的，但是对于比较复杂的缓存数据计算的场景，就不是这样了

如果你频繁修改一个缓存涉及的多个表，那么这个缓存会被频繁的更新，频繁的更新缓存

但是问题在于，这个缓存到底会不会被频繁访问到？？？

举个例子，一个缓存涉及的表的字段，在1分钟内就修改了20次，或者是100次，那么缓存跟新20次，100次; 但是这个缓存在1分钟内就被读取了1次，有大量的冷数据。

**28法则，黄金法则，20%的数据，占用了80%的访问量**

实际上，如果你只是删除缓存的话，那么1分钟内，这个缓存不过就重新计算一次而已，开销大幅度降低。

每次数据过来，就只是删除缓存，然后修改数据库，如果这个缓存，在1分钟内只是被访问了1次，那么只有那1次，缓存是要被重新计算的，用缓存才去算缓存。

其实删除缓存，而不是更新缓存，就是一个lazy计算的思想，不要每次都重新做复杂的计算，不管它会不会用到，而是让它到需要被使用的时候再重新计算。

mybatis，hibernate，懒加载，思想

查询一个部门，部门带了一个员工的list，没有必要说每次查询部门，都里面的1000个员工的数据也同时查出来啊

80%的情况，查这个部门，就只是要访问这个部门的信息就可以了

先查部门，同时要访问里面的员工，那么这个时候只有在你要访问里面的员工的时候，才会去数据库里面查询1000个员工。

#### 六、Redis的并发竞争问题

多客户端同时并发写一个key，可能本来应该先到的数据后到了，导致数据版本错了。或者是多客户端同时获取一个key，修改值之后再写回去，只要顺序错了，数据就错了。

一个场景：

假如有某个key = “price”， value值为10，现在想把value值进行+10操作。正常逻辑下，就是先把数据key为price的值读回来，加上10，再把值给设置回去。如果只有一个连接的情况下，这种方式没有问题，可以工作得很好，但如果有两个连接时，两个连接同时想对还price进行+10操作，就可能会出现问题了。

例如：两个连接同时对price进行写操作，同时加10，最终结果我们知道，应该为30才是正确。

考虑到一种情况：

T1时刻，连接1将price读出，目标设置的数据为10+10 = 20。

T2时刻，连接2也将数据读出，也是为10，目标设置为20。

T3时刻，连接1将price设置为20。

T4时刻，连接2也将price设置为20，则最终结果是一个错误值20。

Redis自己就有天然解决这个问题的CAS类的`乐观锁`方案。

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fxzq12AHsePAAFTHewNYjQ187.png)

行了，Redis这的内容就这样了。一些常见的问题及解决方案都有了。
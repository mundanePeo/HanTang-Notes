<br>

<div align="center">
    <img src="logo.jpg" width="200px">
</div>

<br>

1. **Redies缓存和map/guava做缓存的区别：**
缓存分为本地缓存和分布式缓存，一般来说，map或者guava缓存实现的是本地缓存，最主要的特点是轻量和快速，声明周期随程序段结束而结束。如果存在多个实例，那么每个实例都需要保存一份缓存，缓存不具备一致性。而redies属于分布式缓存的范畴，在多实例的情况下，各个实例公用缓存，缓存具备一致性和高可用性。
Redies的线程模型：redies是单线程模型，主要原因是执行Redies命令的核心模块文件事件处理器(网络事件处理器)是单线程的。但并不是说整个Redies各个模块都是单线程的，比如redies通过多线程在后台删除对象。Redies设计成单线程是有原因的，其实最主要的还是redies是基于内存的操作，它性能瓶颈并不是cpu，而是内存和网络，而单线程模型设计简单，代码清晰；不必考虑各种锁的问题最重要的是也不需要考虑线程切换导致的CPU消耗。Redies使用IO多路复用机制来同时监听多个socket，并根据socket时间来选择对应的事件处理器来进行处理。文件时间处理器主要包含四个部分：多个socket，IO多路复用程序，文件事件分派器，事件处理器。多个socket事件湖被放到一个Queue中去，文件分派器每次只从其中拿出一个事件来处理。
2. **IO多路复用机制：**我觉得就是一个进程或线程同时检测若干个文件描述符(socket)是否可以执行IO操作的能力。
3. **Redies分布式锁：**主要是利用redies的setnx命令，例如setnx key value。Key是锁唯一的标识，一般按业务来决定命名。解锁命令的del key。设置锁超时，expire key outtime是为了解决锁即使没有被显式释放，也能在一定时间后自动释放的问题。但是这样依旧是存在问题的，因为setnx和expire并不是联合在一起的原子操作，这就可能导致setnx后设置Expire失败导致形成死锁。为了解决这个问题可以使用lua脚本，将这两个命令合在一起。锁误解除操作：即一个A操作没有用完锁，就被释放了，此时B获得锁，之后执行删除锁操作实际释放了B的锁，为了解决这个问题可以用一个id来标识锁，并使用lua脚本来判断这个一下再释放。
4. **Redies和memcached区别：**(1)redies支持更加复杂的数据类型，例如字符串，list,set,hash,sort set。而memcached支持简单的string(2)redies支持数据的持久化，可以将内存中的数据存储到磁盘中，重启的时候可以再次加载使用(3)redies支持cluster模式(4)memcached是多线程的而Redies是单线程的多路IO复用。
5. **Redies常用数据结构和使用场景：**
(1)	string:是简单的key value类型，value不仅可以是字符串也可以是数字。可以用来计数，例如微博数，粉丝数等
(2)	Hash:是string类型的filed和value的映射表，hash特别适合用户存储对象(类)。例如可以用hash来存储用户信息，商品信息等
(3)	List:是一个双向链表，支持反向查找和遍历，更加方便操作。例如可以用来做微博的关注列表，消息列表等
(4)	Set:可以去重。可以很轻易的实现求交集，并集和差集。
(5)	Sorted set(zset):和set 相比Sorted set添加了一个权重参数score，使得集合中的元素能够按照Score进行排序。
(6)	HyperLogLog：主要是用来做基数统计的算法，主要优点是计算基数所需要的空间总是固定的并且很小。Redies默认12kb,它可以被用来统计注册IP数，统计每日访问IP数等。
6. **Redies设置过期时间：**
	Redies中有个设置时间过期的功能。他非常的重要，例如短信验证码时间这种就很合适。可以在set key的时候设置expire time，通过过期时间可以指定这个key的存活时间。到时间以后有 定期删除+惰性删除两种删除方式。定期删除：redies默认每隔100ms就随机抽取一些设置了过期时间的key，并检查其是否过期，如果是过期就删除。注意是随机抽取，因为遍历的话对CPU是很重的负载。惰性删除比较简单。只有当查到当前key并发现过期的时候才会删除。
7. **Redies数据淘汰策略：**
(1)	volatile-lru:从已经设置过期时间的数据集中挑选最近最少使用的数据淘汰
(2)	volatile-ttl:从已经设置过期时间的数据集中挑选将要过期的数据淘汰
(3)	volatile-random:从已经设置过期时间的数据集中随机选择数据淘汰
(4)	allKeys-lru:当内存不够使用的时候，就在键空间中移除最近最少使用的key
(5)	allKeys-random: 从数据集中任意选择数据淘汰
(6)	no-eviction:禁止删除数据
(7)	volatile|allkeys-lfu：移除最不常用的数据
8. **redies持久化机制：**
   Redies不同于memcached很重要的一点就是redies支持持久化，并且支持两种持久化方式：快照持久化(RDB)和只追加文件(AOF)
   快照持久化：Redies可以通过创建快照来获得存储在内存中的数据在某个时间节点上的副本。Redies创建快照后，可以对快照进行备份，可以将快照复制到其他服务器上从而创建具有相同数据的服务器副本，还可以将快照留在原地方便服务器重启使用。
   AOF：当redies服务器收到一条更新指令的时候，操作结束之后会将这条命令添加到aof内存缓冲区中，并根据appendfysnc参数在合适的时候缓冲区中的内容刷新到aof文件中。
   AOF重写：AOF重写可以产生一个新的AOF文件，这个新的AOF文件和原有的AOF文件所保存的数据库状态一样，但体积更小。
AOF重写是一个有歧义的名字，该功能是通过读取数据库中的键值对来实现的，程序无须对现有AOF文件进行任何读入、分析或者写入操作。在执行 BGREWRITEAOF 命令时，Redis 服务器会维护一个 AOF 重写缓冲区，该缓冲区会在子进程创建新AOF文件期间，记录服务器执行的所有写命令。当子进程完成创建新AOF文件的工作之后，服务器会将重写缓冲区中的所有内容追加到新AOF文件的末尾，使得新旧两个AOF文件所保存的数据库状态一致。最后，服务器用新的AOF文件替换旧的AOF文件，以此来完成AOF文件重写操作
RDB的优点是恢复速度更快，数据量更小，是二进制形式，不同版本的redies也可用，只需要fork一个子进程进行持久化。父进程依旧可以提供服务。AOF则是数据可靠性更强，最多丢失一秒数据，数据库容错率更好，误操作可以通过直接改aof进行回退。
9. **Redies同步机制：**redies的主从同步机制可以确保master和slave之间数据的同步，同步的方式主要包括全量复制和增量复制。
	全量复制：(1)slave第一次启动时，连接master并发送psync命令(psync ? -1)(2)当master收到这个命令后，会将自己的run id和offset告知slave,同时执行bgsave命令来生成RDB文件并利用缓冲区记录此后所有的写命令(3)master bgsave完成后向slave发送RDB文件，同时继续缓冲此段时间内的写命令，RDB文件发送完毕后，在向slave发送缓冲区的命令(4)slave收到RDB文件中会丢弃所有旧数据并载入RDB并且执行后续的缓冲区命令(5)此后master的每个命令都会也发送给Slave一份。
10. **Redies事务：**
	Redis 通过 MULTI、EXEC、WATCH 等命令来实现事务(transaction)功能。事务提供了一种将多个命令请求打包，然后一次性、按顺序地执行多个命令的机制，并且在事务执行期间，服务器不会中断事务而改去执行其他客户端的命令请求，它会将事务中的所有命令都执行完毕，然后才去处理其他客户端的命令请求。在传统的关系式数据库中，常常用 ACID 性质来检验事务功能的可靠性和安全性。在 Redis 中，事务总是具有原子性（Atomicity）、一致性（Consistency）和隔离性（Isolation），并且当 Redis 运行在某种特定的持久化模式下时，事务也具有持久性（Durability）。
11. **Redies缓存雪崩**：缓存同一时间大面积失效导致所有的请求都会落在数据库上导致数据库短时间内承受大量请求而崩掉。
解决方案：（1）尽量保证redies集群的可用性，发现机器宕机后尽快重启补上，要选择合适的内存淘汰策略。(2)本地缓存和限流尽可能保证数据库不会崩掉。(3)利用持久化机制尽快回复数据库，其中热点数据的集中失效也会导致缓存雪崩这个问题，为了解决热点数据的集中失效问题，我们可以(1)设置不同的失效时间，例如给失效时间加一个随机数(2)设置互斥锁，在请求数据库查询的时候加一个互斥锁
缓存穿透：如果查询一个数据库中不存在的数据，就会导致缓存和数据库都查不到这个请求，但是每个请求都会落到数据库上，这种查询不存在数据的现象就叫做缓存穿透。解决办法主要两个：(1)将这个不存在的key缓存起来，不过value设置为null(2)设置一个布隆过滤器
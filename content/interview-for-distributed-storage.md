原文在[这里](https://zhuanlan.zhihu.com/p/23284834)。一个实习生小朋友骗我说他不会，问我这些题目怎么做。这明摆着换个法子来面试我呀！要是真啥都不会能来我司实习么？全是套路啊。不过原文出题还是很有水平的，所以我决定写一写。

### 1. 用cas实现spinlock.

spinlock在网上应该一搜一大把，我试着给出一个simple甚至naive的实现。

    type SpinLock struct {
        count uint64
    }
    func (l *SpinLock) Lock() {
        for !atomic.CompareAndSwapUint64(&l.count, 0, 1) {
        }
    }
    func (l *SpinLock) Unlock() {
        atomic.StoreUint64(&l.count, 0)
    }

若是较真的话可以看看[这里](https://www.zhihu.com/question/55764216/answer/147607435)。这个题的考点不在这里，问无锁实现，原子操作，ABA问题这些底层方向思路就走偏了。**其实多线程的并发安全，跟多个事务的并发安全，在某种程度上是共通的。** 这才是考察的核心问题，比如我做了一个kv引擎，只提供了get set 和cas操作，那么能否在这个基础上，实现一个锁操作的API？假设我们能实现出锁操作，那我们又能否利用这个锁，实现出跨多个key的修改操作的安全API？如何保证原子语义，要么全成功，要么全失败。

事实上仅用cas这套搞出跨多key的修改是蛮蛋疼的，再细节到网络失败的时候锁的释放，深入思考下就会想到，因为网络是有3态的：成功失败和不可知。一般会基于快照做，只有cas的保证太弱了。

cas提供的原子性，这是一个很重要的点，假设在分布式的事务里面，跨分片的事务提供能否成功，只取决于一个primary key的提交是否能成功。这里有两个问题思考一下也挺有意思，第一，跨多机的时候，如何提供对某一个key的原子语义？第二，如果只有对一个key的原子保证，如果实现跨许多机器跨多个key-value的原子操作语义？

扯淡扯远了，我们看下一题吧。

### 2. 实现单机kv存储系统, 多节点共享kv存储服务, 怎么解决external consistency的问题?

> kv存储N=0

> 用户A和B操作kv存储系统按照下面时序:

> 1.用户A执行操作: INC N;

> 2.用户A通知用户B执行操作;

> 3.用户B执行操作: if (N % 2 == 0) {N*=2;} else {N +=3;}

> 怎么保证结果符合预期呢? 在网络传输影响操作到达次序的情况下, 怎么保证B后于A完成操作.

> 如果这个过程插入了C, 又如何做呢?

外部一致性我记得不是太清了(假装一脸认真)，产生因果关系的操作之间，执行顺序满足因的操作应该先于果？有点像因果率一致？A操作引发了B，那么B一定应该看得到A执行产生的结果。这个例子里面因为这个因果关系，似乎是希望B看到的值应该是N INC之后的值。

两个操作都访问到了N，如果保证操作的安全？无非是，加锁和MVCC。加锁很好理解，读写锁，写写冲突，读写冲突。那么该如何理解MVCC？**MVCC其实很类似一个特殊的cas，它保证了涉及到跨多key修改的原子操作语义**，这样也可以理解为什么MVCC可以把并发粒度控制得更好。

这里是说的单机存储引擎，如果放到分布式里面会更复杂一点。事务A的开始时间戳先于事务B，但是事务B的提交却先于A，这时会发生什么事情？用多版本带一个逻辑时钟，就可以处理这种情况：假设A做INC N操作的时候逻辑时间是5，给B发消息变成6，B收到消息以后，它操作的N的版本应该是6以后的。只需要逻辑时钟，就可以检测到有相互关联性的事务。如果这个过程插入了C，如果C跟A和B没有共同修改的key，那么C的影响可以忽略。如果有修改到N，但是没有跟A和B交互，那么可以认为C的存在与其它用户并没有因果关系，逻辑时钟也不会检测到这一点，是能满足external consistency的。

### 3. 锁实现和版本控制用那个呢?

两者都是方法和手段，并不冲突和矛盾。锁有很多不同的粒度，比如一把全局的大锁；再比如读写锁，任一时刻如果有写，就不能进行其它操作，而读锁之间相互不影响；我看了好些傻逼的实现都是一把全局大锁，像[boltdb](https://github.com/boltdb/bolt)，还有[leveldb的Go语言封装](https://github.com/syndtr/goleveldb/)里面提供的Transaction接口，都是很没节操的。前阵子我还考虑过写一个RangeLock，调整锁的粒度：只有被同步访问到的key之间，才会有锁冲突，比如我在操作A他在操作B，相互是不影响的。遇到锁冲突了会变得复杂，回滚操作必须记得释放之前的锁，加锁也要有点技巧，如果一个操作锁了A去请求B，另一个操作锁了B去请求A，就成环死锁了。

MVCC也会遇到冲突，冲突时无非两种手段：过一会儿重试或者abort。看！这本质上也是锁，乐观锁悲观锁而已。所以并不是用了MVCC锁的概念就消失了。不过MVCC是个好东西，它比锁可以提供更细粒度的并发。通过读历史版本，让读和写之间的冲突进一步降低。代价当然是问题被搞得更加复杂了。

如何选择？根据实际的场景具体情况具体去分析。挑选适当的隔离级别，RC/RR/SI。

### 4. kv系统数据要持久化, 怎么保证在供电故障的情况下, 依然不丢数据.

先写WAL再做写操作，常识。出故障了从check point重放日志，就可以恢复之前的状态机。

### 5. flush/fsync/WAL/磁盘和ssd的顺序写

说到这个问题，就不得不先从缓存聊起。由于下一级的硬件跟不上上一级的读写速度，缓存这东西应运而生。硬盘有缓存，操作系统有缓存，标准库也有缓存，用户还可能自己设缓存，总之是各种的缓存。命中缓存时，可以大大提高读的速度，只有当缓存穿透才会到下层去请求数据。写操作也由于缓存的存在而变成了批量操作，吞吐得以提高。

然而写的时候遇到突然断电的情况，数据还在缓存层没刷下去，就尴尬了...会丢数据！如果要保证可靠写这里我们需要采取些法子，手动将缓存刷进磁盘里。

flush是刷C标准库的IO缓存。fsync是系统调用，页缓存会被刷到磁盘上。

写IO有好多种方式，最笨的调用C的IO库，然后还有操作系统的read/write，或者mmap又或者使用direct-io，甚至是写祼设备。关于这些写下去相关知识也不少。

WAL是常识性的东西，先出日志，重放日志就可以得到快照，即使快照坏掉了，重放日志也可以恢复出正常的快照。而且做同步一般都是基于日志来做的。

最后是磁盘和ssd，了解硬件的特性对于理解优化非常重要。磁盘是需要寻道的，而寻道的硬件机制决定了这个操作快不了。硬盘顺序读写本身的速率比较快，但寻道却要花掉10ms，所以随机读写性能会比较着。ssd那边没有寻道操作，读的速度非常快。然而顺序写的优势相对磁盘并没有高多少。如果没记错，ssd大概就200MB/s的级别，而磁盘顺序写也有接近100MB的级别。

### 6. 单机kv存储系统, 从掉电到系统重启这段时间, 不可用, 如何保证可用性呢?

要有副本。不然哪来可用性A？而有了副本，一致性C又麻烦来了，呵呵。

### 7. 数据复制, 日志复制, 有哪些实现方法呢?

做数据同步操作时，一般是找到快照点，将快照的数据发过去，之后再从放这个点之后的日志数据。回放日志就可以增量同步了，不过增量同步也有不爽的，中间断了太多就需要重新全量。最蛋疼的问题是，增量同步只能做最终一致性。主挂了切到从，丢一段时间的数据。

### 8. 做主从复制, 采用pull和push操作, 那个好呢?

如果保证一致性，由主push并收到应答处理。如果不保证，由从做pull比较好，从可以挂多个，还可以串起来玩。

### 9. 如何保证多副本的一致性? RSM

副本是一致性的最大敌人，一旦有了副本，就有可能出现副本间不同步的情况。异步写的方式顶多只能做到最终一致性，所以必须同步写。写主之后，同步完其它节点从才返回结果。不过写所有节点太慢了，而且挂掉节点时可用性有问题。

在raft出来之前，号称工业上唯一只有一种一致性协议的实现，就是paxos。然而paxos即难懂，又难实现。无论对于教育还是工程角度都它妈蛋疼的要死，还它妈的统治了业界这么多年。强势安利一波raft。

### 10. 分布式共识算法: zab, paxos, raft.

zab还没研究过。basic paxos还勉强能看一看，multi paxos就蛋疼了，看得云里雾里。[raft](tags?name=raft)我写过几篇博客，话题太大，这里不展开了。不过不管是哪一种，搞分布式是逃不掉的。

### 11. commit语意是什么呢?

私以为是ACID里面的D，持久性。一旦提交了，就不会丢。

### 12. 单机或者单个leader的qps/tps较低, 如何扩大十倍?

如果能加机器就搞分布式。做分布式就走两个方向：可以分片就让leader分片，负载就分摊开了吞吐就上来了；可以副本就考虑走follower read，压力就分到了follower中。只要架构做的scalable了，扩大10倍1000都好说。

如果不能走分布式，就考虑优化单机性能。网络的瓶颈就batch + streaming。CPU，内存什么都不说了。硬盘不行就换SSD。

如果是qps，读嘛，该上缓存上缓存。唯有写是不好优化的，tps就合理选择LSM存储引擎，合并写操作，顺序写。怎么让系统性能更好这个话题，展开就更多了，不过最后我还是想讲个笑话。

某大厂某部门半年间系统性能优化了3倍，怎么优化的？因为他们升级了最新版本的php，php编译器的性能提升了3倍。所以嘛...问我怎么优化？升级硬件吧，升级更牛B的硬件，立杆见影。我们客户把TiDB的硬盘升级到了SSD，性能立马提升了10倍。

### 13. 怎么做partitioning和replicating呢?

分片和副本，分布式系统里面的三板斧。系统规模大了，单机承受不住，肯定就分片。做存储是有状态服务，不能单个分片挂了系统就挂了，于是必须有副本。其实两个都麻烦。

分片的麻烦的关键，在于分片调整。比如按hash分的，按range分的，只要涉及调整，就蛋疼。数据迁移是免不了，如果整个过程是无缝的？如果做到不停机升级？升级处理过程中，无信息的更新以及一致性，都是比较恶心的。

副本麻烦的关键，就是一致性了。有副本就引入了一致性问题，paxos可以解救你，如果大脑不会暴掉。

具体的怎么分片还是看业务的。而副本什么也看一致性级别要求，强同步，半同步，最终一致性乱七八糟的。

### 14. 存储或者访问热点问题, 应该怎么搞?

加缓存。或者业务调整看能否hash将访问打散。

### 15. CAP原理

去问google。先问google。容易google到的都不要问我。谁要问我这种东西，我拒绝回答，并给他一个链接：《提问的智慧》。

### 16. 元数据怎么管理?

etcd呀。开源这么多轮子，不好好用多浪费。

### 17. membership怎么管理?

etcd呀。lease。上线下线都注意走好流程。

### 18. 暂时性故障和永久性故障有哪些呢?

暂时性故障：网络闪断，磁盘空间满了，断电，整机房掉电，光纤被挖断了(中国大的互联网公司服务质量的头号敌人)，被ddos了
永久性故障：硬盘挂了，系统挂了，被墙了。

擦，我分不清勒。

### 19. failover和data replication怎么搞呢?

haproxy

### 20. 磁盘的年故障率预估是多少?

待会去google一下。先瞎写写，假设一块磁盘一年故障的概率是P，假设系统有100块磁盘，那么整个系统的磁盘故障率就变成了(1-(1-P)^100)，我知道这个概率会变得非常大。

这时我们考虑RAID的情况。RAID 0卵用都没有，坏一块就坏了。计算公式跟之前一样的。RAID 1两块盘互为镜像，可以把P就成P/4。还有RAID 01/10，怎么计算来的？还有就是纠错码技术。计算更复杂了。

分布式以后，上层可以控制分片和副本数，跟RAID一个道理，然后挂一个副本不会挂，要挂掉系统的大多数......但是故障率是多少呢？操！数学没学好，怎么办啊？

ssd跟磁盘不一样的地方，它以整个block为单位操作，在写入之前需要先擦除，而擦除的次数是有上限的，所以寿命比磁盘的要短很多。具体是多久，还是得问google。

### 21. kv系统存储小王, 小李, 小张三个人的账户余额信息, 数据分别在不同的节点上, 怎么解决小王向小李, 小李向小张同时转款的问题呢?

两阶段提交。打个广告：我们的[TiKV](https://github.com/pingcap/tikv)提供了分布式事务，这个问题很好解决。

    Prewrite 小王账户减;小张账户加 Commit
    Prewrite 小张账户减；小李账户加 Commit

搞定！

真是越写越是瞎扯淡去了...要是让我们的实习生面试我，估计我要挂...也许当初是混进公司的吧，嘘！别让老板知道了。

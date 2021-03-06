# Redis Hint

## 统计大量数据

1. Bitmap，能够完全准确统计**数字**是否存在，但是需要的空间可能比较大。
2. HyperLogLog，能够较为准确的对一系列数字(字符串)统计去重个数。在一系列的随机数中，根据末尾最大连续0的个数估算整体规模。默认有2^14个桶，桶中存放末尾最大连续0的长度。对每个字符串计算hash值，对2^14取模，即最后14位，更新对应桶。最后取所有桶的调和平均，对2取指数。
3. Bloom Filter，统计一个字符串是否存在，能准确判断不存在，可能误判存在。使用一个bitmap，对一个字符串，使用多种hash函数计算其hash值，将hash值对应的bit位置置为1。判断一个字符串是否存在时，若其hash值对应的bit位置只要有一个0，那就是不存在；但是可能不存在，其hash函数对应的所有位置却被其他字符串置1了，所以可能误判存在。  

## 持久化

1. RDB：执行时刻的快照保存，损失数据风险相对较大，恢复快。save命令阻塞模式，bgsave新启一个子进程利用CopyOnWrite写数据，父进程子进程共享内存，当父进程继续提供服务写内存时，会将对应页复制一份再写。默认900s，1；300s，10；60s，10000写一次。
2. AOF：append only，将存储命令，损失风险小，存储文件可能很大，恢复慢。默认未开启，写文件时写入内存缓存区中，开启后默认1s强制刷入磁盘一次。
3. 混合模式：aof存储rdb间的指令变化，这样aof文件相对较小。

## 事务

redis事务对外部是原子性的，在单线程执行事务时不会执行(响应)其他客户端的请求。基本用法，watch(key)->multi->...->exec/discard。如果事务中有命令执行出错，其余的命令仍旧会执行，watch用在multi前，仅当watch的key没变时才会执行事务(可以在事务中修改key，也可以watch多个key)。  
redis事务不执行rollback，因为和关系型数据库不同，redis指令比较简单，一条指令运行出错只有可能是语法错或者数据类型错误等程序编码问题，因此没有必要做rollback。  

## 集群Cluster

redis的key空间被分割为16384(1<<14)个slot，集群最大支持16384个master，建议最大为1000。根据每个key计算CRC16并对16384取余数来确定key在那个slot中。每个redis服务实例互相连接，各自拥有一份slot-node映射表，不做代理转发工作，如果请求的key所在的slot不归自己处理会返回重定向，由客户端自己重新请求。集群的可用性实际是通过每个master-slave结构来保证的，如果负责一片slot的master和slave都挂了，那redis就不再提供服务。  

### 重定向

* MOVED  
当请求给某个node的key不归自己处理时，会给客户端返回MOVED错误，并将slot已经其归属node的IP:Port返回给客户端，客户端自己重新请求。一般客户端会做缓存和更新。原则上客户端不需要知道所有的slot-node映射表就可以工作，但是为了效率，客户端也可以请求(Cluster Nodes / Cluster Slots)获得整个映射关系，比如首次连接时。  

* ASK  
由于node间的slot允许存在迁移，迁移需要将一个node中slot中的key转移到另一个node中。在迁移过程中，slot会处于中间态，一部分在旧的中，一部分在新的中，每迁移一个key就会在旧的中删去。ASK的情况是，node接受到key发现其slot处于迁移状态，查询发现没有这个key，但是它不能断定迁入的node中是否存在，因此需要发送ASK命令，让客户端取另一个node试一试；之后客户端先给新node发送ASKING命令，强制接下来的一个命令可以查询正在迁入的slot；然后客户端发送正常请求，并且不会更新映射关系。区别在于，MOVED的语义是，客户端需要更新映射，认为此后该slot都对应新的node。而ASK不更新映射，因为此时的slot分布在两个node中，对于每一个请求应当有措施保证能访问到这两个node。  

### 容错

#### Ping/Pong与Gossip  

一个节点每秒会随机ping几个其他节点，在过去的一半 NODE_TIMEOUT 时间里都没有发送 ping 包过去或接收从那节点发来的 pong 包的节点，会保证去 ping每一个其他节点。在 NODE_TIMEOUT 这么长的时间过去之前，若当前的 TCP 连接有问题，节点会尝试去重连接，以确保不会被当作不可达的节点。  
ping和pong中会额外包含一个gossip字段，其中包含一部分发送者已知的node信息，包括ID，IP:Port以及状态。

#### 失效检测  

PFAIL表示一个node可能失效，如果一个节点发现自己无法访问另一个节点的时间已经超过阈值了，会将其设置为PFAIL，并在ping或pong中传递出去。若A节点标记B节点为PFail，同时在Gossip段中接收到集群中大部分master的节点也认为B节点PFAIL了，则A会将B标记为FAIL，并且向所有node发送B节点Fail的消息，所有接收到FAIL消息的node都会强制将B设置为FAIL。  

注意，FAIL 标识基本都是单向的，也就是说，一个节点能从 PFAIL 状态升级到 FAIL 状态，但要清除FAIL 只有以下两种可能方法：

* 节点已经恢复可达的，并且它是一个slave节点。在这种情况下，FAIL可以清除掉，当slave节点并没有被故障转移。
* 节点已经恢复可达的，并且它是不负责任何哈希槽，在这种情况下FAIL可以清除，因为master没有任何哈希槽处在集群中，就等待配置加入到集群中.
* 节点已经恢复可达的，而且它是一个master节点，但经过了很长时间（N *NODE_TIMEOUT）后也没有到任何slave节点被提升了,最好重新加入到集群，并继续这个方法。

PFAIL -> FAIL 的转变使用了一种弱协议：

1. 节点是在一段时间内收集其他节点的信息，所以即使大多数master节点要去”同意”标记某节点为 FAIL，实际上这只是表明说我们在不同时间里从不同节点收集了信息，并且我们不确定或不需要，给定的时刻大多数master同意，然后我们舍弃旧的失效报告，所以失效的通知是大多数master在时间窗口内的。
2. 当每个节点检测到 FAIL 节点的时候会强迫集群里的其他节点把各自对该节点的记录更新为 FAIL，但没有一种方式能保证这个消息能到达所有节点。比如有个节点可能检测到了 FAIL 的节点，但是因为网络分区，这个节点无法到达其他任何一个节点。  

#### slave的选举和升级  

node间的ping/pong会带有epoch字段，这个字段在slave升级为master时会增加1，用来区分不同的映射关系改变。  
slave的选举和升级都由slave处理，master主要负责投票。只有当slave节点的master节点处于FAIL状态并且负责的slot数不为0，slave和master断线不超过一定时间(保证数据可靠)才会发起选举。一个slave会增加epoch然后广播，一旦一个master给这个slave投票，那么在2*Node_TIMEOUT不能再给其他slave投票，如果一个slave收到了大多数master节点的回应，他赢得选举成为实际的master；如果在Node_TMIEOUT时间内无法访问大多数master，则会放弃这次选举。  
slave在发现master进入FAIL状态会等一小段时间进行选举。固定延时+随机延时+排名延时，固定延时保证FAIL状态能在集群内广播完，随机延时尽量避免多个slave同时开始申请选举，排名按slave从master复制的数据量排名，和master更接近的更早进行申请选举。  

## 空间释放

### 过期策略

对每一个设置了过期时间的key而言，有主动删除和惰性删除两种。Redis会将每个设置了过期时间的key放在特别的一个字典中，默认每秒10次过期扫描，扫描不得超过25ms，每次扫描时从字典中随机挑选20个key，删除其中的过期key，如果过期的key超过了1/4则再挑20个，重复。惰性删除就是，每次取出一个key时会判断是不是设置了超时时间且是否超时了。  
从节点不会主动删除key，主节点在key过期被删除时会给从节点的AOF文件增加对应的del指令。  
因此应当给key的过期时间加上随机扰动，避免大量key同时过期。  

### 内存超限

Redis中可以配置maxmemory指定最大内存使用量。当超限时可以使用如下策略：

* noeviction(默认)：不响应写请求(可以del)，可响应读请求。  
* volatile-lru：尝试淘汰设置了过期时间的key，lru的key淘汰。  
* volatile-ttl：类似上，淘汰ttl最小的。  
* volatile-random
* all-keys-lru  
* all-keys-random  

Redis使用近似lru，每个key有额外24bit字段储存时间戳。实现lru淘汰时，随机挑选5(可配置)个key(volatile-lru就从设置了过期时间的key中选，all-keys-lru就从所有的key中选)和淘汰池(固定大小数组)合并，删除其中最老的一个，保留其他较老的key，等待下次循环，直到内存不超限。  

### 懒惰删除

del命令同步的删除key释放空间，unlink允许异步释放。后台有专门的回收线程从队列中拉取任务进行释放。此外，AOF写磁盘任务也专门有线程异步的去做。  

## 数据结构

1. 字符串  
  和Java中StringBuilder类似采用冗余byte[]数组(为了兼容C++结束位置为\0)，1M内capacity加倍，1M以上每次增加1M。如果字符串大于44B(64-19-1)使用raw方式，小于等于44B使用embeded方式。区别在于，每个Redis对象都有一个RedisObject的对象头，里边有个指针指向实际对象。embeded模式下对象头和SDS(字符串struct)是连续存放的。  

2. 字典  
  和Java中的HashMap(JDK7版本)类似。插入时是插在链表头部(不会省时间，无论如何都得遍历一次链表)，插在头部是假定新的数据更有可能被get，因此放在链表头部；JDK8为了缓解多线程下对HashMap错误使用的死循环，将节点插入在链表末尾。  
  rehash过程和HashMap的resize一样，但是改成了渐进式的(同时也会启动定时任务来搬迁，防止很久没有写命令到来)。当存储的KV数量等于table长度时进行rehash。当执行hset/hdel这些写命令时先进行一次搬迁操作(每次搬N个KV数据)，然后如果处在rehash状态，直接将节点插入到ht[1]的对应链表头上。当rehash时get，先遍历旧的table，如果处在rehash状态再遍历新的table。  

3. 压缩列表ziplist  
  一块连续空间，分为对象头和一系列的entry，每个entry相当于是list的一个元素。字段头后每个entry首先是preLen记录前一个entry长度(用于倒序遍历)，然后是encoding字段表示entry的编码类型，最后是byte[]。preLen是可变长整数，长度小于254(0xFE)时为1B，大于时改用5B表示。因此，当修改一个entry的内容时，需要修改后一个entry的preLen字段，如果preLen字段由1B变为了5B，则还需要修改再下一个entry，从而导致级联更新。

4. 快速列表quicklist
  快速列表就是多个压缩列表的链表形式，默认其中一个压缩列表8K。  

5. 升级版压缩列表listpack
  和ziplist相比，listpack将entry的长度length存在了自己的结构中，因此不存在了级联更新。length是1到5字节的可变长整数，通过length的某些bit来标志是多少字节。  

6. 跳表skiplist  
  zset由一个字典和一个跳表组成。
  skiplist是一个有序多层链表，最多64层。其中的每个node有一个指针数组forward[]，以不同的跨度指向后一个node，同时有一个前向指针指向前一个节点。搜索时从高层开始，如果下一个node的score太大，就降低层数。redis默认每层的晋升概率为25%，也即每插入一个新节点，这个新节点有25的概率是1层及以上，25×25的概率是2层及以上。插入时走一遍搜索过程，删除时也是。更新时，先删除再插入，走两边搜索过程。  
  这里的skiplist还不太是sortedSet，也即是和红黑树不一样。红黑树对key做sortedSet，key不能重复；skiplist中每个node有value和score(分别对应key,vlaue)，是以score进行排序的，是允许重复的，没有办法通过value(key)定位到Node，因此需要和Hash结构\<value, node\*\>配合形成zset。如果要将skiplist作为sortedMap/Set使用，应当按照key排序。此外，红黑树和skiplist相比，不适合做范围搜索，因为一旦log(n)找到范围的最小值以及最大值后，skiplist可以通过最底层的链表直接返回，红黑树则需要中序遍历，不是很容易返回之间的所有元素。  

7. 基数树radixtree  
  redis实现的基数树和字典树Trietree相比，每一层可以不是一个字符(传统Trie树)而是一个字符串，从而避免深度过大。  

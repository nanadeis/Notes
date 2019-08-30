# CAP
## C(一致性)
强一致性：针对一个数据项的更新操作执行成功后，所有用户都可以读到最新的值
最终一致性：
【最终一致性有哪些变种? 举个最终一致性的例子】


## A(可用性)
## P(分区容错性)
放弃P意味着放弃了可扩展性，是分布式系统的最基本的要求
三者不可用时满足  

# BASE  


# 两阶段提交协议 2PC

# 三阶段提交协议 3PC  

# Paxos协议  

# Raft协议

# ZAB协议  
Zookeeper通过ZAB协议实现分布式数据一致性  
ZAB包括消息广播和崩溃恢复两个过程，分为发现（Leader选举）、同步（follower服务器和leader服务器保持同步状态）、消息广播三个阶段  
## 消息广播   
leader服务器向follower服务器广播proposal，follower收到proposal后以事务日志的方式写入到本地磁盘中去，并且在写入成功后反馈给leader服务器一个ACK响应。当Leader服务器收到半数的follower的ACK后，就会广播一个commit消息给所有的follower，leader和follower完成事务提交  
## 崩溃恢复  
当leader宕机或者leader和半数的follower失去联系，就会进入崩溃恢复   
重新选举leader+数据同步  
通过epoch编号来区分leader周期变化，每个事务请求都有一个全局唯一的事务ZXID，是一个64位的数字，高32位就是epoch，低32位为单调递增的计数器  

# Zookeeper是什么？  
Zookeeper是google chubby 的开源实现，为分布式应用提供高效且可靠的分布式协调服务，提供诸如统一命名服务、配置管理和分布式锁等分布式基础服务。  

# Zookeeper提供了什么？  
1. **文件系统Znode**   
zookeeper提供一个多层级节点命名空间，节点称为Znode。为了保证高吞吐和低延迟，在内存中维护这个文件系统。因此不能用于存放大量数据，每个节点的存放上限为1M  
Znode分为四种类型  
持久节点  被创建后一直存在，直到有删除操作来主动清除这个节点  
持久顺序节点  每个父节点会为它的第一集子节点维护一份顺序，用于记录子节点创建的先后顺序。基于这个属性在创建子节点的时候，create -s创建顺序节点，zookeeper会自动在路径后面添加上10位的数字（计数器），数字后缀的上限是有符号整数的最大值。  
临时节点  和客户端的session断开就会自动删除
临时顺序节点  

2. **通知机制Watcher**
客户端向zookeeper服务器注册watcher的同时，会将watcher对象存储在客户端watcherManager中，在zookeeper服务器触发watcher事件后，会向客户端发送通知。客户端线程从watchManager中取出对应的watcher对象来执行回调逻辑。  
特点：  
1. 一次性  
一旦一个watcher被触发，zookeeper都会将其从相应的存储中移除。如果客户端想要后续更新的通知，必须在watch被触发后重新注册一个watch  
这样的设计减轻了服务端的压力。避免频繁更新的节点，需要不断地向客户端发送事件通知。  
2. 异步性  
不会block正常的读写请求  
3. 主动推送  
4. 顺序性  
如果多个更新触发了多个watch，那么watch被触发的顺序与更新顺序一致。客户端watcher回调的执行是顺序执行的。  
5. 轻量  
WatcherEvent只包含三部分内容：通知状态、事件类型、节点路径。比如只通知数据节点的内容发生了变更。但是变更后的数据无法直接从事件中获取，需要客户端主动去重新获取数据。    
网络开销、服务器内存开销小  

3. **ACL权限控制**


# Zookeeper的应用场景  
## 数据发布与订阅（配置中心）  
发布者将数据发布到zookeeper节点上，供订阅者动态获取数据，实现配置信息的集中式管理和动态更新。例如全局的配置信息，分布式服务框架的服务地址列表  
【Znode+watcher】  

## 负载均衡  
一个应用或服务的提供方会部署多份，达到对等服务。消费者需要在这些对等服务器里选择一个来执行相关的业务逻辑。比较典型的是消息中间件里的生产者、消费者负载均衡，如Kafka  
1. 生产者负载均衡  
在zookeeper上注册broker和topic节点，并注册watcher监听。生产者就可以动态的获取broker和topic的变化情况  
2. 消费者负载均衡 
注册消费者临时节点，一旦发现消费者新增或者减少或者broker服务器发生变化，就会触发消费者的负载均衡  

【临时节点+watcher】
## 命名服务  
通过使用命名服务，客户端能够根据指定名字来获取资源或服务的地址，提供者等信息。被命名的实体可以是集群中的机器，提供的服务地址或远程对象等。  
如Dubbo使用Zookeeper维护全局的服务地址列表，服务提供者在启动的时候，向Zookeeper上的指定节点目录下写入自己的URL地址，就完成了服务的发布  
也可以用顺序节点生成唯一ID
【Znode】  

# 分布式通知和协调  
**心跳检测**  
通过zookeeper的临时节点可以判断对应的客户端机器是否存活  
**工作进度汇报**  
在zookeeper上选择一个节点，每个工作进程在该节点下创建临时节点，各个工作进程实时地将自己的任务执行进度写到临时节点上，汇总的经常可以监控子节点的变化获得工作进度的实时的全局情况  
**系统调度**  
分布式系统由控制台和客户端系统组成。控制台的职责就是需要将一些指令信息发送给所有的客户端，以控制它们进行相应的业务逻辑。后台管理人员在控制台上做一些操作，实际上就是修改了Zookeeper上某些节点的数据，而Zookeeper进一步把这些数据变更以事件通知的形式发送给了对应的订阅客户端  

# 集群管理  
对集群中每台机器的运行时状态进行数据收集  
对集群中机器进行上下线操作
传统基于Agent的方法（每个机器上运行一个agent向一个监控中心系统汇报自己所在机器的状态)的弊端：
1. 大规模升级困难  
2. 同一的agent无法满足多样的需求，业务不同，需要监控的东西也会不同  
3. 编程语言的多样性。监控中心整合异构系统的数据困难  
Zookeeper可以通过注册watcher监听向订阅的客户端发送通知，并且临时节点在会话失效的时候会被自动清除。利用这两点可以实现另一种集群机器存活性监控的系统  

# master选举  
Zookeepr的强一致性，zookeeper的节点的全局唯一性
利用临时顺序节点，序列号小的作为master  
【临时顺序节点】  

# 分布式锁  
## 排它锁（mutex）  
所有客户端都会试图通过调用create（）接口，创建临时子节点，最终只会有一个客户端创建成功。创建成功的客户端就获得了锁。其他客户端就要在父节点上注册子节点变更的watcher监听。  
当获得锁的客户端宕机或者完成了业务逻辑，就会删除该临时节点（释放锁）  
zookeeper通知watch的客户端，这些客户端接到通知后，重新发起create（）试图获取锁  
## 共享锁（读写锁）  
1. 在获取共享锁时，所有客户端都会在/shared_lock下创建一个临时顺序节点，如果当前是读请求，那么就会创建如/shared_locl/192.168.1.1-R-0000000001的节点，如果是写请求就会创建如/shared_lock/192.168.1.1-W-0000000001的节点  
2. 客户端调用getChildren（）接口来获取所有已经创建的子节点列表，注意这里不注册任何watcher  
3. 如果无法获取共享锁(如果是读请求，为最小的节点或比自己小的节点都是读请求，就获得锁。对于写请求，若为最小的节点就获得锁)就调用exist（）来对比自己小的节点注册watcher
读请求：向比自己序号小的最后一个写请求节点注册watcher  
写请求：向比自己序号小的最后一个节点注册watcher  
4. 等待watcher通知，继续进入步骤2  

# 分布式队列  
## FIFO  
类似全写的共享锁模型  
1. 所有客户端创建持久顺序节点（Znode存储的数据就是消息队列中的消息内容，持久防止消息丢失）  
2. 获取所有子节点  
3. 确定自己的节点序号在所有子节点中的顺序。如果自己是序号最小的节点，消费该消息，删除该节点。如果不是，则对比自己小的最后一个节点注册watcher监听  
4. 接到watcher后，重复步骤2  

## Barrier  
开始时，/queue_barrier是一个已经存在的默认节点，其数据内容是一个数字n代表barrier值。所有客户端都会到/queue_barrier下创建临时子节点  
1. 通过调用getData（）获取/queue_barrier节点的数据内容:n
2. 通过调用getChildren（）接口获取/queue_barrier节点下的所有子节点，即获取队列中的所有元素，同时注册对子节点列表变更的watcher监听  
3. 统计子节点的个数，如果为n表示齐了，进入业务处理。如果不足n，就需要进入等待  
4. 接到watcher通知，重复步骤2  


# Zookeeper集群中的角色  
## leader  
事务请求【会改变服务器状态的请求称为事务请求，通常包括创建节点、更改数据、删除节点、以及创建会话等请求】的唯一调度者和处理者，保证集群事务处理的顺序性  
集群内部各服务器的调度者  

## follower  
处理客户端非事务请求，转发事务请求给leader服务器  
参与事务请求proposal的投票  
参与leader选举的投票  

## observer  
与follower的唯一区别是，observer不参与任何投票  
用于在不影响集群事务处理能力的情况下，提高非事务处理能力

# Zookeeper的一致性保证
1. 顺序一致性：来自客户端的更新将按照发送的顺序被写入到zookeeper  
递增的ZXID   
2. 原子性：更新操作要么成功，要么失败，没有中间状态  
Zookeeper通过版本（每个Znode都有版本信息）保证分布式数据的原子性操作，Zookeeper采用乐观锁，乐观锁包括数据读取，写入校验和数据写入三个阶段，版本用于写入校验  
可以在读取数据的时候先执行一下 sync 操作，即与 leader 节点先同步一下数据，再去取，这样才能保证数据的强一致性。

# Zookeeper的缺点  
1. leader重新选举期间，整个Zookeeper集群不可用，且leader选举时间很长为30-120s  
2. zookeeper不是为高可用设计的，跨机房的时候，一旦机房之间连接出现故障，zookeeper master只能顾一个机房，其他机房运行的业务模块由于没有master都只能停掉。
3. 无法保证强一致性  
4. 权限控制薄弱
# leader选举  
1. 所有服务器处于LOOKING状态，寻找leader，发起投票  
2. 第一次投票的时候投自己，即投票信息为（自己的SID，最大的ZXID）
3. 集群中的每台机器发出自己的投票后，也会接收到来自集群中其他机器的投票。找到ZXID最大的SID（包括自己），如果ZXID相等的时候，选SID大的。作为下一轮投票发出去。
4. 如果收到超过半数的相同的投票，就将该投票的SID作为leader  
因此服务器上的数据越新，ZXID越大，越可能成为leader  

# Zookeeper的数据同步过程  
选完leader就进入数据同步过程  
1. leader等待follower和observer连接  
2. follower连上leader后，将最大zxid发送给leader  
3. leader根据follower的zxid确定同步点，根据leader服务器和follower服务器之间的数据差异情况来决定同步方式    
4. 完成同步后通知follower已经成为uptodate状态  
5. follower收到uptodate消息后，又可以重新接受client的请求进行服务了  
数据同步的四种方式：
1. SNAP-全量同步  
peerLastZxid < minCommittedLog【leader服务器proposal缓存队列CommittedLog中的最小ZXID】  
leader服务器无法直接使用proposal缓存队列和follower进行数据同步。leader发送快照SNAP指令通知follower即将进行全量数据同步。leader将全量的数据发送给follower  
2. DIFF-增量同步  
peerLastZXID 介于 minCommittedLog 和 maxCommittedLog 之间  
leader首先发送一个DIFF指令，通知Lean而进入差异化数据同步阶段。针对每个proposal都会发两个数据包，一个proposal内容数据包和一个commit指令数据包，和运行时leader和follower之间的事务请求提交过程是一致的  
3. TRUNC-仅回滚同步
peerLastZXID > maxCommittedLog  
follower上有proposal并未在leader上提交【leader服务器在已经将事务记录到本地事务日志中，但并没有成功发起proposal流程就挂了。在重新选举leader后，又恢复成为follower】，需要回滚  
4. TRUNC+DIFF-回滚+增量同步  
leader a已经将事务truncA提交到本地事务日志中，但没有成功发起proposal协议进行投票就宕机了；然后集群中剔除原leader a重新选举出新leader b，又提交了若干新的提议proposal，然后原leader a重新服务又加入到集群中说明：此时a,b都有一些对方未提交的事务，若b是leader, a需要先回滚truncA然后增量同步新leader b上的数据。


# Session  
Zookeeper会为每个客户端分配一个Session，类似web服务器一样，用来标识客户端的身份  
## 一次session的创建过程  
1. 初始化阶段  
初始化Zookeeper对象  
设置会话默认watcher  
构造Zookeeper服务器地址列表管理器HostProvider  
创建并初始化客户端网络连接器ClientCnxn  
初始化SendThread（用于客户端和服务器的网络IO）和EventThread（用于事件处理）    
2. 会话创建阶段  
启动SendThread和EventThread  
获取一个服务器地址  
创建TCP连接  
够着ConnectRequest请求  
发送请求  
3. 响应处理阶段  
接收服务端响应  
处理Response  
连接成功
生成事件：SyncConnected-none代表会话创建成功（为了让上层应用感知会话创建成功）  
查询watcher   
处理事件  

## 会话的状态  
CONNECTING：开始创建Zookeeper对象，从服务器地址列表里逐个选取IP地址尝试进行网络连接，直到成功连接上，变为CONNECTED状态；网络闪断，自动进行重连再次变为CONNECTING,重新连接上又会变成CONNECTED  
RECONNECTING    
RECONNECTED  
CLOSED：会话超时、权限检查失败、客户端主动退出  

## session的作用  
客户端表示  
超时检查  
请求的顺序执行  
维护临时节点的生命周期  
watcher通知   



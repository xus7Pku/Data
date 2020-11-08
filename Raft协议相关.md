# 一、Raft算法概述

## 1、三种角色

  **Raft是一个用于管理日志一致性的协议**。它将分布式一致性分解为多个子问题：**Leader选举（Leader election）、日志复制（Log replication）、安全性（Safety）、日志压缩（Log compaction）等**。同时，Raft算法使用了更强的假设来减少了需要考虑的状态，使之变的易于理解和实现。**Raft将系统中的角色分为领导者（Leader）、跟从者（Follower）和候选者**（Candidate）：

- Leader：**接受客户端请求，并向Follower同步请求日志，当日志同步到大多数节点上后告诉Follower提交日志**。
- Follower：**接受并持久化Leader同步的日志，在Leader告之日志可以提交之后，提交日志**。
- Candidate：**Leader选举过程中的临时角色**。
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190722215348483.jpg)

  Raft要求系统在任意时刻最多只有一个Leader，正常工作期间只有Leader和Followers。Raft算法将时间分为一个个的**任期（term）**，每一个term的开始都是Leader选举。在成功选举Leader之后，Leader会在整个term内管理整个集群。如果Leader选举失败，该term就会因为没有Leader而结束。

## 2、Term

  **Raft 算法将时间划分成为任意不同长度的任期（term）**。任期用连续的数字进行表示。**每一个任期的开始都是一次选举（election），一个或多个候选人会试图成为领导人**。如果一个候选人赢得了选举，它就会在该任期的剩余时间担任领导人。在某些情况下，选票会被瓜分，有可能没有选出领导人，那么，将会开始另一个任期，并且立刻开始下一次选举。**Raft 算法保证在给定的一个任期最多只有一个领导人**。

## 3、RPC

  **Raft 算法中服务器节点之间通信使用远程过程调用（RPC）**，并且基本的一致性算法只需要两种类型的 RPC，为了在服务器之间传输快照增加了第三种 RPC。

【RPC有三种】：

- **RequestVote RPC**：**候选人在选举期间发起**。
- **AppendEntries RPC**：**领导人发起的一种心跳机制，复制日志也在该命令中完成**。
- **InstallSnapshot RPC**: 领导者使用该RPC来**发送快照给太落后的追随者**。

# 二、Leader选举

## 1、Leader选举的过程

  **Raft 使用心跳（heartbeat）触发Leader选举**。当服务器启动时，初始化为Follower。**Leader**向所有**Followers**周期性发送**heartbeat**。**如果Follower在选举超时时间内没有收到Leader的heartbeat，就会等待一段随机的时间后发起一次Leader选举**。

  **每一个follower都有一个时钟，是一个随机的值，表示的是follower等待成为leader的时间，谁的时钟先跑完，则发起leader选举**。

  **Follower将其当前term加一然后转换为Candidate。它首先给自己投票并且给集群中的其他服务器发送 RequestVote RPC**。结果有以下三种情况：

- **赢得了多数的选票，成功选举为Leader**；
- 收到了Leader的消息，表示有其它服务器已经抢先当选了Leader；
- 没有服务器赢得多数的选票，Leader选举失败，等待选举时间**超时后发起下一次选举**。
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190722215422566.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhYWlrdWFpY2h1YW4=,size_16,color_FFFFFF,t_70)

## 2、Leader选举的限制

  **在Raft协议中，所有的日志条目都只会从Leader节点往Follower节点写入，且Leader节点上的日志只会增加，绝对不会删除或者覆盖**。

  这意味着Leader节点必须包含所有已经提交的日志，即能被选举为Leader的节点一定需要包含所有的已经提交的日志。因为日志只会从Leader向Follower传输，所以如果被选举出的Leader缺少已经Commit的日志，那么这些已经提交的日志就会丢失，显然这是不符合要求的。

  这就是Leader选举的限制：**能被选举成为Leader的节点，一定包含了所有已经提交的日志条目**。

# 三、日志复制（保证数据一致性）

## 1、日志复制的过程

  **Leader选出后，就开始接收客户端的请求**。**Leader把请求作为日志条目（Log entries）加入到它的日志中，然后并行的向其他服务器发起 AppendEntries RPC复制日志条目**。当这条日志被复制到大多数服务器上，Leader将这条日志应用到它的状态机并向客户端返回执行结果。

- 客户端的每一个请求都包含被复制状态机执行的指令。
- **leader把这个指令作为一条新的日志条目添加到日志中，然后并行发起 RPC 给其他的服务器，让他们复制这条信息**。
- 假如这条日志被安全的复制，领导人就应用这条日志到自己的状态机中，并返回给客户端。
- **如果 follower 宕机或者运行缓慢或者丢包，leader会不断的重试，直到所有的 follower 最终都复制了所有的日志条目**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190810105643571.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhYWlrdWFpY2h1YW4=,size_16,color_FFFFFF,t_70)
  **简而言之，leader选举的过程是：1、增加term号；2、给自己投票；3、重置选举超时计时器；4、发送请求投票的RPC给其它节点**。

## 2、日志的组成

  **日志由有序编号（log index）的日志条目组成**。**每个日志条目包含它被创建时的任期号（term）和用于状态机执行的命令**。如果一个日志条目被复制到大多数服务器上，就被认为可以提交（commit）了。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190810105900139.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhYWlrdWFpY2h1YW4=,size_16,color_FFFFFF,t_70)

上图显示，共有 8 条日志，提交了 7 条。提交的日志都将通过状态机持久化到磁盘中，防止宕机。

## 3、日志的一致性

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190722215549343.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhYWlrdWFpY2h1YW4=,size_16,color_FFFFFF,t_70)

## （1）日志复制的两条保证

- 如果不同日志中的两个条目有着**相同的索引和任期号，则它们所存储的命令是相同的**（原因：**leader 最多在一个任期里的一个日志索引位置创建一条日志条目，日志条目在日志的位置从来不会改变**）。
- 如果不同日志中的两个条目有着**相同的索引和任期号，则它们之前的所有条目都是完全一样的**（原因：**每次 RPC 发送附加日志时**，leader 会把这条日志条目的前面的**日志的下标和任期号一起发送给 follower**，如果 **follower 发现和自己的日志不匹配，那么就拒绝接受这条日志**，这个称之为**一致性检查**）。

## （2）日志的不正常情况

  一般情况下，Leader和Followers的日志保持一致，因此 AppendEntries 一致性检查通常不会失败。然而，Leader崩溃可能会导致日志不一致：**旧的Leader可能没有完全复制完日志中的所有条目**。

  下图阐述了一些Followers可能和新的Leader日志不同的情况。**一个Follower可能会丢失掉Leader上的一些条目，也有可能包含一些Leader没有的条目，也有可能两者都会发生**。丢失的或者多出来的条目可能会持续多个任期。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019072221560351.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhYWlrdWFpY2h1YW4=,size_16,color_FFFFFF,t_70)

## （3）如何保证日志的正常复制

  Leader通过强制Followers复制它的日志来处理日志的不一致，Followers上的不一致的日志会被Leader的日志覆盖。**Leader为了使Followers的日志同自己的一致，Leader需要找到Followers同它的日志一致的地方，然后覆盖Followers在该位置之后的条目**。

  具体的操作是：**Leader会从后往前试**，每次AppendEntries失败后尝试前一个日志条目，**直到成功找到每个Follower的日志一致位置点（基于上述的两条保证），然后向后逐条覆盖Followers在该位置之后的条目**。

  总结一下就是：**当 leader 和 follower 日志冲突的时候**，leader 将**校验 follower 最后一条日志是否和 leader 匹配**，如果不匹配，**将递减查询，直到匹配，匹配后，删除冲突的日志**。这样就实现了主从日志的一致性。

# 四、安全性

  Raft增加了如下两条限制以保证安全性：

- 拥有**最新的已提交的log entry的Follower才有资格成为leader**。
- **Leader只能推进commit index来提交当前term的已经复制到大多数服务器上的日志**，旧term日志的提交要等到提交当前term的日志来间接提交（log index 小于 commit index的日志被间接提交）。
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190722215639499.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhYWlrdWFpY2h1YW4=,size_16,color_FFFFFF,t_70)

# 五、日志压缩

  在实际的系统中，**不能让日志无限增长**，否则**系统重启时需要花很长的时间进行回放**，从而影响可用性。**Raft采用对整个系统进行snapshot来解决，snapshot之前的日志都可以丢弃（以前的数据已经落盘了）**。

  **每个副本独立的对自己的系统状态进行snapshot，并且只能对已经提交的日志记录进行snapshot**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190810155624687.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhYWlrdWFpY2h1YW4=,size_16,color_FFFFFF,t_70)

【**Snapshot中包含以下内容**】：

- **日志元数据，最后一条已提交的 log entry的 log index和term**。这两个值在snapshot之后的第一条log entry的AppendEntries RPC的完整性检查的时候会被用上。
- **系统当前状态**。

  当Leader要发给某个日志落后太多的Follower的log entry被丢弃，Leader会将snapshot发给Follower。或者当新加进一台机器时，也会发送snapshot给它。发送snapshot使用InstalledSnapshot RPC。

  做snapshot既不要做的太频繁，否则**消耗磁盘带宽**， 也不要做的太不频繁，否则一旦节点重启需要回放大量日志，影响可用性。**推荐当日志达到某个固定的大小做一次snapshot**。

  做一次snapshot可能耗时过长，会影响正常日志同步。可以通过使用copy-on-write技术避免snapshot过程影响正常日志同步。

# 六、成员变更

## 1、常规处理成员变更存在的问题

  我们先将成员变更请求当成普通的写请求，由领导者得到多数节点响应后，每个节点提交成员变更日志，将从旧成员配置（Cold）切换到新成员配置（Cnew）。但每个节点提交成员变更日志的时刻可能不同，这将造成各个服务器切换配置的时刻也不同，这就有可能选出两个领导者，破坏安全性。

  考虑以下这种情况：集群配额从 3 台机器变成了 5 台，**可能存在这样的一个时间点，两个不同的领导者在同一个任期里都可以被选举成功（双主问题）**，**一个是通过旧的配置，一个通过新的配置**。

  **简而言之，成员变更存在的问题是增加或者减少的成员太多了，导致旧成员组和新成员组没有交集，因此出现了双主**。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190810162505704.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhYWlrdWFpY2h1YW4=,size_16,color_FFFFFF,t_70)

## 2、解决方案之一阶段成员变更

  **Raft解决方法是每次成员变更只允许增加或删除一个成员（如果要变更多个成员，连续变更多次）**。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190810205030207.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhYWlrdWFpY2h1YW4=,size_16,color_FFFFFF,t_70)

# 七、关于Raft的一些面试题

## 1、Raft分为哪几个部分？

  **主要是分为leader选举、日志复制、日志压缩、成员变更等**。

## 2、Raft中任何节点都可以发起选举吗？

  Raft发起选举的情况有如下几种：

- 刚启动时，所有节点都是follower，这个时候发起选举，选出一个leader；
- 当leader挂掉后，**时钟最先跑完的follower发起重新选举操作**，选出一个新的leader。
- 成员变更的时候会发起选举操作。

## 3、Raft中选举中给候选人投票的前提？

  **Raft确保新当选的Leader包含所有已提交（集群中大多数成员中已提交）的日志条目**。这个保证是在RequestVoteRPC阶段做的，candidate在发送RequestVoteRPC时，会带上自己的**last log entry的term_id和index**，follower在接收到RequestVoteRPC消息时，**如果发现自己的日志比RPC中的更新，就拒绝投票**。日志比较的原则是，如果本地的最后一条log entry的term id更大，则更新，如果term id一样大，则日志更多的更大(index更大)。

## 4、Raft网络分区下的数据一致性怎么解决？

  发生了网络分区或者网络通信故障，**使得Leader不能访问大多数Follwer了，那么Leader只能正常更新它能访问的那些Follower，而大多数的Follower因为没有了Leader，他们重新选出一个Leader**，然后这个 Leader来接受客户端的请求，如果客户端要求其添加新的日志，这个新的Leader会通知大多数Follower。**如果这时网络故障修复 了，那么原先的Leader就变成Follower，在失联阶段这个老Leader的任何更新都不能算commit，都回滚，接受新的Leader的新的更新（递减查询匹配日志）**。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190810162203146.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhYWlrdWFpY2h1YW4=,size_16,color_FFFFFF,t_70)

## 5、Raft数据一致性如何实现？

  **主要是通过日志复制实现数据一致性，leader将请求指令作为一条新的日志条目添加到日志中，然后发起RPC 给所有的follower，进行日志复制，进而同步数据**。

## 6、Raft的日志有什么特点？

  **日志由有序编号（log index）的日志条目组成，每个日志条目包含它被创建时的任期号（term）和用于状态机执行的命令**。

## 7、Raft和Paxos的区别和优缺点？

- Raft的leader有限制，**拥有最新日志的节点才能成为leader**，multi-paxos中对成为Leader的限制比较低，**任何节点都可以成为leader**。
- **Raft中Leader在每一个任期都有Term**号。

## 8、Raft prevote机制？

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190811094829159.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhYWlrdWFpY2h1YW4=,size_16,color_FFFFFF,t_70)
  **Prevote（预投票）是一个类似于两阶段提交的协议**，**第一阶段先征求其他节点是否同意选举，如果同意选举则发起真正的选举操作，否则降为Follower角色**。这样就**避免了网络分区节点重新加入集群，触发不必要的选举操作**。

## 9、Raft里面怎么保证数据被commit，leader宕机了会怎样，之前的没提交的数据会怎样？

  **leader会通过RPC向follower发出日志复制，等待所有的follower复制完成，这个过程是阻塞的**。

  **老的leader里面没提交的数据会回滚，然后同步新leader的数据**。

## 10、Raft日志压缩是怎么实现的？增加或删除节点呢？？

  在实际的系统中，**不能让日志无限增长**，否则**系统重启时需要花很长的时间进行回放**，从而影响可用性。**Raft采用对整个系统进行snapshot来解决，snapshot之前的日志都可以丢弃（以前的数据已经落盘了）**。

  **snapshot里面主要记录的是日志元数据，即最后一条已提交的 log entry的 log index和term**。

## 11、Raft里面的lease机制是什么，有什么作用？

  **租约机制确保了一个时刻最多只有一个leader，避免只使用心跳机制产生双主的问题**。**中心思想是每次租约时长内只有一个节点获得租约、到期后必须重新颁发租约**。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190811102518447.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhYWlrdWFpY2h1YW4=,size_16,color_FFFFFF,t_70)

## 12、Raft协议的leader选举，正常情况下，网络抖动造成follower发起leader选举，且该follower的Term比现有leader高，集群中所有结点的日志信息当前一致，这种情况下会选举成功吗？

  **参考网络分区的情况**。



**CAP定理**

1. Consistency:一致性
2. Availability:可用性
3. Partition-tolerance:分区容错性

CAP定理指出，在异步网络模型中，不存在一个系统可以同时满足上述3个属性。换句话说，分布式系统必须舍弃其中的一个属性。对于需要在分布式条件下运行的系统来说，如何在一致性、可用性和分区容错性中取舍，或者说要弱化哪一个属性，是首先要考虑的问题。

对于高可用性的系统来说，往往会保留强一致性。但对于强一致性的系统来说，有一类专门解决这种问题的算法——共识算法。"共识"的意思是保证所有的参与者都有相同的认知(可以理解为强一致性)。共识算法本身可以依据是否有恶意节点分为两类，大部分时候共识算法指的是没有恶意节点的那一类，即系统中的节点不会向其他节点发送恶意请求，比如欺骗请求。共识算法中最有名的是Paxos算法。其次是Raft和ZAB算法(Zookeeper中的实现)



# 1.Paxos算法存在的问题

*Paxos*算法是莱斯利·兰伯特（英语：Leslie Lamport，LaTeX中的「La」）于1990年提出的一种基于消息传递且具有高度容错特性的一致性算法。

- 难以理解

“The dirty little secret of the NSDI community is that at most five people really, truly understand every part of Paxos :-)” — NSDI reviewer

NSDI：关于网络系统设计的著名会议

 

NSDI社区的一个肮脏的小秘密是，至多有五个人真正地了解Paxos的每一个部分。

Paxos算法是如何工作的？

Paxos算法每个阶段的意图是什么？

很难产生直观的感受。

 

- 难以实现并应用于现实中的系统

“There are significant gaps between the description of the Paxos algorithm and the needs of a realworld system ... the final system will be based on an unproven protocol” — Chubby authors

Chubby：Google分布式锁服务

 

Paxos算法的描述与现实世界系统的需求之间存在明显的差距。最终开发完成的系统将构建在未经验证的协议之上。

人们在实现Paxos算法时，发现很多实现上的难题，最终开发出和Paxos算法不一样的结构。

 

# 2.Raft算法

Raft协议由Diego Ongaro和John Ousterhout（斯坦福大学）共同于**2014**年设计。

 

## 2.1 复制状态机

![img](https://img2020.cnblogs.com/blog/676975/202003/676975-20200326215929108-2011740694.png) 

 

每一个服务器存储着一个日志序列。每一个服务器按照顺序应用每条日志（或称之为执行指令），并应用到状态机中。

 

保证了日志序列相同，那么，在应用每条日志（或称之为执行每个指令）时，每台服务器的状态机状态都是一致的。

 

保证日志序列相同就是一致性算法的工作。

 

## 2.2. Raft算法

Raft选举出一个Leader，通过Leader管理日志复制来实现一致性。

Leader从客户端接收日志条目，将日志条目复制到其他服务器上。当确保安全性的时候，告诉其他的服务器应用日志条目到他们的状态机上。

 

为了可理解性，Raft对通过问题分解，将算法分为了几个独立的部分

- Leader选举
- 日志复制
- 安全性问题

### 2.2.1 安全性问题

| 特性             | 解释                                                         |
| ---------------- | ------------------------------------------------------------ |
| 选举安全原则     | 对于一个给定的任期号，最多只会有一个领导人被选举出来         |
| 领导人只附加原则 | 领导人绝对不会删除或者覆盖自己的日志，只会增加               |
| 日志匹配原则     | 如果两个日志在相同的索引位置的日志条目的任期号相同，那么我们就认为这个日志从头到这个索引位置之间全部完全相同 |
| 领导人完全原则   | 如果某个日志条目在某个任期号中已经被提交，那么这个条目必然出现在更大任期号的所有领导人中 |
| 状态机安全原则   | 如果一个领导人已经在给定的索引值位置的日志条目应用到状态机中，那么其他任何的服务器在这个索引位置不会提交一个不同的日志 |

### 2.2.2 Leader选举

一个Raft集群包含若干个服务器节点，通常是5个，允许整个系统容忍2个节点的失效。

在任何时刻，每一个服务期节点都处于以下3种状态之一：

- Leader：处理所有的客户端请求（如果客户端将请求发给了Follower，Follower会把请求重定向给Leader）
- Follower：不会发送任何请求，只会简单地响应来自Leader或Candidate的请求
- Candidate：用于选举产生新的领导人

![img](https://img2020.cnblogs.com/blog/676975/202003/676975-20200326220003772-673639676.png)

 

 

![img](https://img2020.cnblogs.com/blog/676975/202003/676975-20200326220024439-205875381.png)

Term——任期

1）每个任期最多只能有一个领导人Leader

2）有时候任期内可能没有领导人（因选举失败）

3）每台机器上维持着当前的任期号。

会在每次RPC通信时会将自己的任期号进行传递（可用于识别过时的信息）

如果RPC收到任期号大的值，变为follower。

如果RPC收到过期的任期号，返回错误信息。

 ![img](https://img2020.cnblogs.com/blog/676975/202003/676975-20200326220110180-63553133.png)

***选举安全原则：对于一个给定的任期号，最多只会有一个领导人被选举出来\***

1）每个机器只能投一次票（含投给自己）

2）获得多数选票的成为leader

 

**问题：**

如果选举过程中，一直没有一个Candidate获得大多数选票，将一直进行重复选举。

**如何保证会有一个Candidate会获取到大多数选票成为Leader？**

Raft的解决方法：随机超时时间，保证只有一个follewer超时，变成Candidate，并（在其他follower超时前）发送投票请求，得到大多数的选票，从而成为leader。

 

### 2.2.3日志复制

***领导人只附加原则：领导人绝对不会删除或者覆盖自己的日志，只会增加\***

***日志匹配原则：如果两个日志在相同的索引位置的日志条目的任期号相同，那么我们就认为这个日志从头到这个索引位置之间全部完全相同\***

客户端发送命令给Leader。

Leader把日志条目加到自己的日志里。

Leader发送AppendEntries RPC请求给所有的follower。

 

#### 日志一致性检查

![img](https://img2020.cnblogs.com/blog/676975/202003/676975-20200326220143863-2085488177.png)

AppendEntries RPC里包含着prevLogIndex，prevLogTerm。

Follower接收到后，会检查相应的日志条目是否匹配上

1）如果匹配上了（Example#1），将收到的日志条目添加到自己的日志里。

（根据【日志匹配原则】，表示follower和leader的prevLogIndex以前的所有日志条目是安全相同的）

2）如果没有匹配上（Example#2），follower将拒绝此次请求，Leader将index调小，不断进行重试，直到匹配上。

（Example#3）follower会将后面的日志条目全部删除，再将leader的日志条目添加上。

 

#### 日志提交

（日志提交：指将日志条目应用到状态机中）

一旦新的日志条目变成【已经复制到大多数follower机器上】的了。

~~（PS：关于上面所讲的“大多数follower机器”个数？~~

~~问题：可以这么理解吗？~~

~~满足【“大多数follower机器”个数 + leader > 集群中机器个数的一半】就可以。~~

~~比如，集群中机器个数为5，“大多数follower机器”个数为2就可以了。~~

~~答案：~~

~~不可以，因为如果leader宕机不重启，无法保证仍旧为大多数。~~

~~）~~

- Leader在状态机里提交自己日志条目，然后返回结果给客户端
- Leader下次发送AppendEntries RPC时，告知follower已经提交的日志条目信息(lastIndex)
- foller收到RPC后，提交到自己的状态机里

 

在Raft运行过程中，可能的各个机器日志条目状态如下。

![img](https://img2020.cnblogs.com/blog/676975/202003/676975-20200326220210073-244970589.png)

**问题：**

**领导人S1宕机了，重新进行选举，是每台机器都可以被选举成为新leader吗？**

**如果S4被选举成term 4的leader，会发生什么问题？**

 

 

答案：

当S4发送AppendEntries RPC请求给所有的follower时，

在进行日志一致性检查时，会将follower已经提交到状态机中的日志条目给删除掉！！从而违反了状态机安全原则。

***状态机安全原则：如果一个领导人已经在给定的索引值位置的日志条目应用到状态机中，那么其他任何的服务器在这个索引位置不会提交一个不同的日志***

 

 

***领导人完全原则：如果某个日志条目在某个任期号中已经被提交，那么这个条目必然出现在更大任期号的所有领导人中***

**如何保证选举出的新leader包含所有已提交的日志条目？**

 

选举规则的限制：Candidate发送的RequestVote RPC中含有lastIndex，lastTerm。

当机器收到后，如果Candidate的日志不如自己的新，将拒绝投票给它。

日志的新旧的比较：通过**比较两份日志中最后一条日志条目的索引值和任期号**定义谁的日志比较新。

- 如果两份日志最后的条目的任期号不同，那么任期号大的日志更加新。
- 如果两份日志最后的条目任期号相同，那么日志比较长的那个就更加新。

![img](https://img2020.cnblogs.com/blog/676975/202003/676975-20200326220333678-1263185553.png)

 

 

S1：选票（S1，S2，S3，S4，S5）可获得多数选票

S2：选票（S2，S4）

S3：选票（S1，S2，S3，S4，S5）可获得多数选票

S4：选票（S4）

S5：选票（S2，S4，S5）可获得多数选票

因此，S1, S3, S5可被选举为新的leader，S2，S4则不行。

 

**思考：为何通过这种方式可以保证新选举的leader一定拥有已commit的日志条目？**

答案：

leader在当日志条目【已经复制到大多数follower机器上】时，才会提交日志条目。

假设已经含有已提交日志条目的机器（可能含上一次的leader）组成的集合为Set。

根据以上选举规则的限制，可知新leader一定在Set中。
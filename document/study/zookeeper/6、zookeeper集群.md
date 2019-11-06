### 一、Zookeeper集群的特点
<img src="C:\Users\liyunhua\AppData\Roaming\Typora\typora-user-images\1572774423561.png" alt="1572774423561" style="zoom: 50%;" />

- 可靠的ZooKeeper服务
- 只要集群的大多数都准备好了，就可以使用这项服务
- 容错集各设置至少需要三个服务器，强烈建议使用奇数个数服务器
- 建议每个服务运行在单独的机器上

### 二、Zookeeper集群监控
#### 方式一：四字命令
```bash
# 测试端口是否正常 
echo ruok | telnet 127.0.0.1:2181
# 连接
echo ruok | nc 127.0.0.1:2181
```
**参数用法**

|      |                                                              |
| ---- | ------------------------------------------------------------ |
| conf | 输出相关服务配置的详细信息。                                 |
| cons | 列出所有连接到服务器的客户端的完全的连接/会话的详细信息。包括“接受/发送”的包数量、会话id、操作延迟、最后的操作执行等等信息 |
| crst | 重置当前这台服务器所有连接/会话的统计信息                    |
| dump | 列出未经处理的会话和临时节点                                 |
| envi | 输出关于服务环境的详细信息(区别于conf命令)                   |
| reqs | 列出未经处理的请求                                           |
| ruOk | 测试服务是否处于正确状态。如果确实如此，那么服务返回"imok"，否则不做任何相应 |
| srst | 重置服务器统计信息                                           |
| srvr | 列出服务器详细的信息，zk版本、接收/发送包数量、连接数、模式(leader/follower)、节点总数 |
| stat | 输出关于性能和连接的客户端的列表                             |
| wchs | 列出服务器watch的详细信息                                    |
| wchc | Bgsession列出服务器watch的详细信息，它的输出是一个与watch相关的会话的列表 |
| wchp | 通过路径列出服务器watch的详细信息。它输出一个与session相关的路径 |
| mntr | 列出集群的健康状态。包括“接受/发送”的包数量、操作延迟、当前服务模式(leader/follower)、节点总数、watch总数、临时节点总数 |


#### 方式二：JMX
JMX (Java Management Extensions) Java管理扩展，是一个
为应用程序、设备、系统等植入管理功能的框架。
<img src="D:\doc\blog\images\study\zookeeper\1572774453842.png" alt="1572774453842" style="zoom:80%;" />

**打开步骤:**
![1572774484609](D:\doc\blog\images\study\zookeeper\1572774484609.png)
![1572774496410](D:\doc\blog\images\study\zookeeper\1572774496410.png)
![1572774511724](D:\doc\blog\images\study\zookeeper\1572774511724.png)

### 三、集群的灵魂-Leader
#### (一)、每台服务器都可以成为Leader
![1572774525514](D:\doc\blog\images\study\zookeeper\1572774525514.png)

#### (二)、分布式一致性算法：Paxos算法
##### 定义
Paxos算法是Leslie Lamport于1990年提出的一种基于消息传递且具有高度容错特性的一致性算法,是分布式一致性中的经典算法。

##### Paxos算法的过程（算法描述）
Paxos算法类似于两阶段提提交，其算法执行过程分为两个阶段。具体如下：
**阶段一（prepare阶段）：**
(a) Proposer选择一个提案编号N，然后向半数以上的Acceptor发送编号为N的Prepare请求。Pareper（N）

(b) 如果一个Acceptor收到一个编号为N的Prepare请求，如果小于它已经响应过的请求，则拒绝，不回应或回复error。若N大于该Acceptor已经响应过的所有Prepare请求的编号（maxN），那么它就会将它已经接受过（已经经过第二阶段accept的提案）的编号最大的提案（如果有的话，如果还没有的accept提案的话返回{pok，null，null}）作为响应反馈给Proposer，同时该Acceptor承诺不再接受任何编号小于N的提案。

**阶段二（accept阶段）：**
(a) 如果一个Proposer收到半数以上Acceptor对其发出的编号为N的Prepare请求的响应，那么它就会发送一个针对[N,V]提案的Accept请求给半数以上的Acceptor。注意：V就是收到的响应中编号最大的提案的value（某个acceptor响应的它已经通过的{acceptN，acceptV}），如果响应中不包含任何提案，那么V就由Proposer自己决定。

(b) 如果Acceptor收到一个针对编号为N的提案的Accept请求，只要该Acceptor没有对编号大于N的Prepare请求做出过响应，它就接受该提案。如果N小于Acceptor以及响应的prepare请求，则拒绝，不回应或回复error（当proposer没有收到过半的回应，那么他会重新进入第一阶段，递增提案号，重新提出prepare请求）。

##### Paxos角色
<img src="D:\doc\blog\images\study\zookeeper\1572774540843.png" alt="1572774540843" style="zoom:80%;" />

##### Paxos流程
![1572774557638](D:\doc\blog\images\study\zookeeper\1572774557638.png)

![1572774568823](D:\doc\blog\images\study\zookeeper\1572774568823.png)

![1572774582449](D:\doc\blog\images\study\zookeeper\1572774582449.png)

![1572774598387](D:\doc\blog\images\study\zookeeper\1572774598387.png)

![1572774613515](D:\doc\blog\images\study\zookeeper\1572774613515.png)

#### (三)、ZooKeeper集群 Leader 选举
![1572774625389](D:\doc\blog\images\study\zookeeper\1572774625389.png)

**对选举Leader的要求**
- 对选举Leader节点上要持有最高的zxid
- 过半数节点同意

**内置实现的选举算法：**
- LeaderElection
- FastLeaderElection 默认的
- AuthFastLeaderElection

**选举机制中的概念:**
- 服务器id myid
- 事务id,服务器中存放的最大Zxid
- 逻辑时钟，发起的投票轮数计数
- 选举状态：
  - LOOKING,竞选状态。
  - FOLLOWING,随从状态，同步leader状态，参与投票。
  - OBSERVING,观察状态，同步leader状态，不参与投票。
  - LEADING,领导者状态。
  

**选举消息内容:**
- 服务器id
- 事务id
- 逻辑时钟
- 选举状态

**Leader选举算法**
- 选举算法：
1. 每个服务实例均发起选举自己为领导者的投票（自己的投给自己）；
2. 其他服务实例收到投票邀请时，比较发起者的数据事务ID是否比自己最新的事务ID大，大则给它投一票，小则不投票给它，相等则比较发起者的服务器ID,大则投票给它；
3. 发起者收到大家的投票反馈后，看投票数（含自己的）是否大于集群的半数，大于则胜出，担任领导者；未超过半数且领导者未选出，则再次发起投票。

- 胜出条件：
:投票赞成数大于半数则胜出的逻辑。

**Leader选举流程示例说明**

有5台服务器，每台服务器均没有数据，它们的编号分别是1,2,3,4,5,按编号依次启动，
它们的选择举过程如下：
- 服务器1启动，给自己投票，然后发投票信息，由于其它机器还没有启动所以它收不到反馈信息，服务器1的状态一直属于Looking

- 服务器2启动，给自己投票，同时与之前启动的服务器1交换结果，由于服务器2的编号大所以服务器务器2胜出，但此时投票数没有大于半数，所以两个服务器的状态依然是LOOKING

- 服务器3启动，给自己投票，同时与之前启动的服务器1,2交换信息，由于服务器3的编号最大所以服务器3胜出，此时投票数正好大于半数，所以服务器3成为领导者，服务器1,2成为小弟。

- 服务器4启动，给自己投票，同时与之前启动的服务器1,2,3交换信息，尽管服务器4的编号大，但之前服务器3已经胜出，所以服务器4只能成为小弟。

- 服务器5启动，后面的逻辑同服务器4成为小弟。
### 一、应用程序集群
<img src="D:\doc\blog\images\study\zookeeper\1572774687515.png" alt="1572774687515" style="zoom: 50%;" />

### 二、ZAB协议介绍
ZAB协议（ZooKeeper Atomic Broadcast, ZooKeeper原子消息广播协议）是专为zookeeper设计的数据一致性协议。参考Paxos来实现的。
![1572774725232](D:\doc\blog\images\study\zookeeper\1572774725232.png)
关注点数据的一致性，无关数据的准确性、权威性、实时性

### 三、ZAB协议的重要特性-有序性
![1572774742898](D:\doc\blog\images\study\zookeeper\1572774742898.png)

1.    所有事务请求转发给leader
2.    Leader分配全局单调递增事务，id(Zxid)广播事务提议
3.    Follower处理提议，做出反馈
4.    Leader收到过半数反馈，广播Commit
5.    Leader做出响应

### 四、ZK集群-群龙无首
Leader服务器出现崩溃，或者说由于网络原因导致Leader服务器失去了与过半Follower的联系，那么就会进入崩溃恢复模式。
![1572774759496](D:\doc\blog\images\study\zookeeper\1572774759496.png)

- ZAB协议规定如果一个事务Proposal在一台机器上被处理成功，那么应该在所有的机器上都被处理成功,哪怕机器出现故障崩溃。
- ZAB协议**确保**那些已经在Leader服务器上提交的事务最终被所有服务器都提交
- ZAB协议**确保**丢弃那些只在Leader服务器上被提出的事务

### 五、ZAB协议-崩溃恢复
ZAB协议需要设计的选举算法应该满足：确保提交已经被Leader提交的事务Proposal,同时丢弃已经被跳过的事务Proposal。
如果让Leader选举算法能够保证新选举出来的Leader服务器拥有集群中所有机器最高ZXID的事务Proposal,那么就可以保证这个新选举出来的Leader-定具有所有已经提交的提案。
同时，如果让具有最高编号事务Proposal的机器来成为Leader,就可以省去Leader服务器检查Proposal的提交和丢弃工作的这一步操作。


### 六、ZAB协议-数据同步

Leader选举出来后，需完成Followers与Leader的数据同步，当半数的Followers完成同步，则可以开始提供服务。同步的过程如下。

Leader服务器会为每一个Follower服务器都准备一个队列，并将那些没有被各Follower服务器同步的事务以Proposal消息的形式逐个发送给Follower服务器，并在每一个Proposal消息后面紧接着再发送一个Commit消息，以表示该事务已经被提交。等到Follower服务器将所有其尚未同步的事务Proposal都从Leader服务器上同步过来并成功应用到本地数据库中后，Leader服务器就会将该Follower服务器加入到真正的可用Follower列表中，并开始之后的其他流程。

### 七、ZAB协议-丢弃事务Proposal处理
在ZAB协议的事务编号ZXID设计中，ZXID是一个64位的数字，低32位是一个简单的单调递增的计数器，针对客户端的每一个事务请求，Leader服务器在产生一个新的事务Proposal的时候，都会对该计数器进行加1操作；高32位代表了 Leader周期纪元的编号，每当选举产生一个新的Leader服务器，就会从这个Leader服务器上取出其本地日志中最大事务Proposal的ZXID，并从该ZXID中解析出对应的纪元值，然后对其进行加1操作，之后就会以此编号作为新的纪元,并将低32位置0来开始生成新的ZXID。

基于这样的策略，当一个包含了上一个Leader周期中尚未提交过的事务Proposal的服务器启动加入到集群中，发现此时集群中已经存在leader,将自身以Follower角色连接上Leader服务器之后，Leader服务器会根据自己服务器上最后被提交的Proposal来和Follower服务器的Proposal进行比对，发现follower中有上一个leader周期的事务Proposal时，Leader会要求Follower进行一个回退操作——回退到一个确实已经被集群中过
半机器提交的最新的事务Proposal。
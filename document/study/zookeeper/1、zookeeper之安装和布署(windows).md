### 单机版Zookeeper安装和使用(windows版本)
一、安装1.6版本以上JDK，配置环境变量

二、下载地址
https://archive.apache.org/dist/zookeeper/zookeeper-3.4.13/zookeeper-3.4.13.tar.gz

三、解压后的conf目录，增加配置文件zoo.cfg
![1572763923681](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572763923681.png)

四、开启和运行zookeeper
![1572763960411](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572763960411.png)

### Zookeeper集群搭建（windows版本）
#### 一、总体配置介绍
![1572763974973](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572763974973.png)
- initLimit
集群中的follower服务器(F)与leader服务器(L)之间完成初始化同
步连接时能容忍的最多心跳数（tickTime的数量）。如果zk集群
环境数量确实很大，同步数据的时间会变长，因此这种情况下
可以适当调大该参数。
- syncLimit
集群中的follower服务器与leader服务器之间请求和应答之间能
容忍的最多心跳数（tickTime的数量）。
集群节点server.id= host:    port:port
id，通过在各自的dataDir目录下创建一个名为myid的文件来为每
台机器赋予一个服务器id。
两个端口号，第一个跟随者用来连接到领导者，第二个用来选举
领导者。

#### 二、配置步骤
- 1、将zookeeper项目复制三份，按zookeeper1、zookeeper2、
zookeeper3进行命名

- 2、进入三个zookeeper下conf目录内，分别创建zoo.cfg，具体配置如下

```
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=D://data/zookeeper//zookeeper1//data
dataLogDir=D://data/zookeeper//zookeeper1//log
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
maxClientCnxns=3
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

# cluster configuer

server.1=127.0.0.1:2881:3881
server.2=127.0.0.1:2882:3882
server.3=127.0.0.1:2883:3883
```

```
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=D://data/zookeeper//zookeeper2//data
dataLogDir=D://data/zookeeper//zookeeper2//log
# the port at which the clients will connect
clientPort=2182
# the maximum number of client connections.
# increase this if you need to handle more clients
maxClientCnxns=3
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

# cluster configuer

server.1=127.0.0.1:2881:3881
server.2=127.0.0.1:2882:3882
server.3=127.0.0.1:2883:3883
```

```
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=D://data/zookeeper//zookeeper3//data
dataLogDir=D://data/zookeeper//zookeeper3//log
# the port at which the clients will connect
clientPort=2183
# the maximum number of client connections.
# increase this if you need to handle more clients
maxClientCnxns=3
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

# cluster configuer

server.1=127.0.0.1:2881:3881
server.2=127.0.0.1:2882:3882
server.3=127.0.0.1:2883:3883
```

- 3、在上述设置的三个dataDir目录下，创建myid文件，myid的数值与zoo.cfg配置的server.1、server.2、server.3后面的数字匹配
![1572763995764](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572763995764.png)
![1572764007797](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572764007797.png)

**特别注意：在windows环境下,创建的myid文件不要带后缀.txt或其它后缀。会启动闪退，找不到myid文件，切记。**

- 4、启动三个目录中的zkServer.cmd，启动第一个时会报错，等三个全部启动完毕，恢复正常。
![1572764017574](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572764017574.png)
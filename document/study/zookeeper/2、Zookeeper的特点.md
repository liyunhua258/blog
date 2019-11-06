## Zookeeper的特点
### 一. 简单
**(一). 数据结构简单**
类似Unix文件系统树形结构，每个目录称为Znode节点，但是又不同于文件系统，既可以做目录拥有子节点，又可以做文件存放数据。
![1572764101280](D:\doc\blog\images\study\zookeeper\1572764101280.png)

1. 同节点下的子节点名称不能相同
2. 命名有规范
3. 绝对路径
4. 存放的数据大小有限制

**(二). 数据模型**
***层次名称空间***
* 类似unix文件系统，以  /   为根
* 区别：节点可以包含与之关联的数据以及子节点 (既是文件也是文件夹)
* 节点的路径总是表示为规范的、绝对的、斜杠分隔的路径。

***znode***
* 名称唯一，命名规范
* 节点类型：持久、顺序、临时、临时顺序
* 节点数据构成

**(三). 命名规范**
**节点名称除下列限制外，可以使用任何unicode字符：**

* null字符(\u0000)不能作为路径名的一部分；
* 以下字符不能使用，因为它们不能很好地显示，或者以令人困惑的方式呈现:
    \u0001 - \u0019和\u007F - \u009F。
* 不允许使用以下字符:\ud800 - uf8fff， \uFFF0 - uFFFF。
* “.”字符可以用作另一个名称的一部分，但是“.”和“..”不能单独用于指示路径上的节点，因为ZooKeeper不使用相对路径。下列内容无效:
* “/a/b/. / c”或“c / a / b / . . /”。
* “zookeeper”是保留节点名。


**(四). 操作命令**

***增删改查指令***

| 指令         | 描述                                | 使用范围    |
| ------------ | ----------------------------------- | ----------- |
| ls           | 获取子节点                          | 命令行和API |
| create       | 在zookeeper中的某个位置创建一个节点 | 命令行和API |
| delete       | 删除节点                            | 命令行和API |
| exists       | 测试节点是否存在                    | API         |
| get data     | 从指定节点读取数据                  | 命令行和API |
| set data     | 将数据存入指定节点                  | 命令行和API |
| get children | 查询指定节点之下所有的子节点        | API         |
| sync         | 等待数据进行同步                    | 命令行和API |
| stat         | 输出关于性能和连接客户端的列表      | 命令行和API |

1. ls 获取子节点
![](https://img-blog.csdn.net/20151202171835891)

2.  在zookeeper中的某个位置创建一个节点 
![](https://img-blog.csdn.net/20151202190410371)

3. get data 从指定节点读取数据
![](https://img-blog.csdn.net/20151202185805806)

4. set data 将数据存入指定节点 
![](https://img-blog.csdn.net/20151202190138792)

5. delete 删除节点
![](https://img-blog.csdn.net/20151202190512082)

6. stat 输出相关信息
![](https://img-blog.csdn.net/20151202190019428)


### 二. 有序
* #### 多种方式跟踪时间

**Zxid**
ZooKeeper中的每次更改操作都对应一个唯一的事务id，称为Zxid，它是一个全局有序的戳记，
如果zxid1小于zxid2，则zxid1发生在zxid2之前。

**Version numbers**
版本号，对节点的每次更改都会导致该节点的版本号之一增加。这三个版本号是dataVersion
(对znode数据的更改次数)、cversion(对znode子节点的更改次数)和aclVersion(对znode ACL的更改次数)。

**Ticks**
当使用多服务器ZooKeeper时，服务器使用“滴答”来定义事件的时间，如状态上传、会话超时、
对等点之间的连接超时等。滴答时间仅通过最小会话超时(滴答时间的2倍)间接公开;如果客户端请求的会话超时
小于最小会话超时，服务器将告诉客户端会话超时实际上是最小会话超时。

**Real time**
ZooKeeper除了在znode创建和修改时将时间戳放入stat结构之外，根本不使用Real time或时钟时间。

* #### 节点上的元数据信息-Stat
![1572773730390](D:\doc\blog\images\study\zookeeper\1572773730390.png)

| 属性           | 描述                                |
| -------------- | ----------------------------------- |
| cZxid          | 创建该节点的zxid                    |
| ctime          | 该节点的创建时间                    |
| mZxid          | 该节点的最后修改zxid                |
| mtime          | 该节点的最后修改时间                |
| pZxid          | 该节点的最后子节点修改zxid          |
| cversion       | 该节点的子节点变更次数              |
| dataVersion    | 该节点数据被修改的次数              |
| aclVersion     | 该节点的ACL变更次数                 |
| ephemeralOwner | 临时节点所有者会话ID，非临时节点为0 |
| dataLength     | 该节点数据长度                      |
| numChildren    | 子节点数                            |

除了ephemeralOwner、dataLength、numChildren，其他属性都体现了顺序

### 三. 可复制
#### 可以快速的搭建集群
<img src="D:\doc\blog\images\study\zookeeper\1572773756694.png" alt="1572773756694" style="zoom: 50%;" />
集群特点，保证了服务的可靠性，可靠性使其不会成为单点故障

### 四. 快速
#### ZK快在哪里?
* ZooKeeper数据加载在内存中，这意味着ZooKeeper可以获得高吞吐量和低延迟数。
* 以读取为主的工作负载中，它尤其快。
* 操作的Znode的数据大小限制1M

ZooKeeper的性能方面意味着它可以用于大型分布式系统
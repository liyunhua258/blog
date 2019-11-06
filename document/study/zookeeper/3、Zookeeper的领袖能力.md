### 领袖核心能力

#### 一. Zookeeper会话机制
<img src="D:\doc\blog\images\study\zookeeper\1572773876720.png" alt="1572773876720" style="zoom:50%;" />

* 一个客户端连接一个会话，由zk分配唯一会话id；
* 客户端以特定的时间间隔发送心跳以保持会话有效； tickTime
* 超过会话超时时间未收到客户端的心跳，则判定客户端死了；(默认2倍tickTime)
* 会话中的请求按FIFO顺序执行。

#### 二. znode—数据构成
![1572773917126](D:\doc\blog\images\study\zookeeper\1572773917126.png)
* 节点数据：存储的协调数据（状态信息、配置、位置信息等）
* 节点元数据（stat结构）
* 数据大小上限：1M

#### 三. znode—节点类型
- 持久节点 PERSISTENT ```create /app1 666```
- 临时节点 EPHEMERAL ```  create  -e /app2 888 ```
    客户端session超时这类节点就会被自动删除
- 顺序节点 PERSISTENT_SEQUENTIAL ``` create  -s /app1/cp 888 ```
  顺序自动编号持久化节点，这种节点会根据当前已存在的节点数自动加1
    - 10位十进制序号
    - 每个父节点一个计数器
    - 计数器是带符号int（4字节）到2147483647之后将溢出(导致名称“<path>-2147483648”)

> 结果：cp0000000000

- 临时顺序节点 EPHEMERAL_SEQUENTIAL ``` create  -e -s /app1/ 888 ```
    临时自动编号节点
> 结果：0000000001

#### 四. Watch监听机制
##### 两类watch
- data watch  监听 数据变更
- child watch  监听子节点变化

```
[zk: localhost:2181(CONNECTED) 17] help
    stat path [watch]
        ls path [watch]
        ls2 path [watch]
    get path [watch]
```

##### 触发watch事件

**Created event**: 
    Enabled with a call to exists.
**Deleted event**:
    Enabled with a call to exists, getData, and getChildren.
**Changed event**: 
    Enabled with a call to exists and getData.
**Child event**: 
    Enabled with a call to getChildren.
    
```
getData("/node-x",watcher)
getChildren("/node-x",watcher)
exists("/node-x",watcher)
```
##### Watch重要特性
- 一次性触发：watch触发后即被删除。要持续监控变化，则需要持续设置watch；
- 有序性：客户端先得到watch通知，后才会看到变化结果


##### Watch注意事项Watch注意事项

1. watch是一次性触发器;如果您获得了一个watch事件，并且希望得到关于未来更改的通知，则必须设置另一个watch。
2. 因为watch是一次性触发器，并且在获取事件和发送获取watch的新请求之间存在延迟，所以不能可靠地得到节点发生的每个更改。
3. 一个watch对象只会被特定的通知触发一次。如果一个watch对象同时注册了exists、getData，当节点被删除时，删除事件对exists 、getData都有效，但只会调用watch一次。

#### 五. ZooKeeper特性

1. 顺序一致性(Sequential Consistency)，保证客户端操作是按顺序生效的；

3. 原子性(Atomicity)，更新成功或失败。没有部分结果。 

5. 单个系统映像，无论连接到哪个服务器，客户端都将看到相同的内容

7. 可靠性，数据的变更不会丢失，除非被客户端覆盖修改。

9. 及时性，保证系统的客户端当时读取到的数据是最新的。
### 概述
ACL全称为Access Control List（访问控制列表），用于控制资源的访问权限。ZooKeeper使用ACL来控制对其znode（ZooKeeper数据树的数据节点）的访问。
zk利用ACL策略控制节点的访问权限，如节点数据读写、节点创建、节点删除、读取子节点列表、设置节点权限等。
在Zookeeper中，znode的ACL是没有继承关系的，每个znode的权限都是独立控制的，只有客户端满足znode设置的权限要求时，才能完成相应的操作。
Zookeeper的ACL，分为三个维度：scheme、expression、permission，通常表示为：scheme: expression: permission，
schema代表授权策略，id代表用户，permission代表权限。

#### 一、**scheme**
scheme即采取的授权策略，每种授权策略对应不同的权限校验方式。下面是zk常用的几种scheme：
- world：默认方式，相当于全世界都能访问
- auth：不使用任何id，表示任何经过身份验证的用户。
- digest：即用户名:密码这种方式认证，这也是业务系统中最常用的,使用username:password字符串生成MD5哈希，然后将其用作ACL的ID标识。通过以明文形式发送 例如：wangsaichao:123456 来完成身份验证。在ACL中使用时，表达式将是wangsaichao:G2RdrM8e0u0f1vNCj/TI99ebRMw=。
- ip：使用Ip地址认证

##### world
语法：
```bash
world:anyone:cdrwa
```
创建节点默认的scheme，所有人都可以访问。如下所示：
```shell
## 创建新的节点 /node01
[zk: localhost:2181(CONNECTED) 10] create /node01 234
Created /node01
# 查看ACL
[zk: localhost:2181(CONNECTED) 11] getAcl /node01
'world,'anyone
: cdrwa
[zk: localhost:2181(CONNECTED) 12] 
```
#####  digest
语法：
```bash
digest:username:BASE64(SHA1(password)):cdrwa
```
digest：是授权方式
username:BASE64(SHA1(password))：是id部分
cdrwa：权限部份
用户名+密码授权访问方式，也是常用的一种授权策略。id部份是用户名和密码做sha1加密再做BASE64加密后的组合，比如设置一个节点的用户名为wangsaichao，密码为123456，则表示方式为：
原：wangsaicaho:BASE64(SHA1(123456))
正确：wangsaichao:G2RdrM8e0u0f1vNCj/TI99ebRMw=。
密码加密需要用到zk的一个工具类来生成，如下所示：

```shell
java -Djava.ext.dirs=/Users/wangsaichao/Desktop/zookeeper-3.4.6/lib -cp /Users/wangsaichao/Desktop/zookeeper-3.4.6/zookeeper-3.4.6.jar org.apache.zookeeper.server.auth.DigestAuthenticationProvider wangsaichao:123456 wangsaichao:123456->wangsaichao:G2RdrM8e0u0f1vNCj/TI99ebRMw=
```
本文的Zookeeper安装在：/Users/wangsaichao/Desktop/zookeeper-3.4.6
```shell
## 查看/根节点
[zk: localhost:2181(CONNECTED) 1] ls /
[zookeeper]
## 创建节点/node
[zk: localhost:2181(CONNECTED) 2] create /node 123
Created /node
## 设置权限
[zk: localhost:2181(CONNECTED) 3] setAcl /node digest:wangsaichao:G2RdrM8e0u0f1vNCj/TI99ebRMw=:cdrwa
cZxid = 0x2
ctime = Sun Sep 09 16:25:51 CST 2018
mZxid = 0x2
mtime = Sun Sep 09 16:25:51 CST 2018
pZxid = 0x2
cversion = 0
dataVersion = 0
aclVersion = 1
ephemeralOwner = 0x0
dataLength = 3
numChildren = 0
## 获取节点刚刚设置的权限
[zk: localhost:2181(CONNECTED) 4] getAcl /node
'digest,'wangsaichao:G2RdrM8e0u0f1vNCj/TI99ebRMw=
: cdrwa
## 没有授权，创建子节点失败
[zk: localhost:2181(CONNECTED) 5] create /node/child_01 234
Authentication is not valid : /node/child_01
## 为当前session添加授权信息
[zk: localhost:2181(CONNECTED) 6] addauth digest wangsaichao:123456
## 添加授权信息后，创建子节点成功
[zk: localhost:2181(CONNECTED) 7] create /node/child_01 234 
Created /node/child_01
[zk: localhost:2181(CONNECTED) 8] 
```
##### auth
scheme为auth时,不需要id。说的不需要id,但是还需要使用一个`username:password`的expression来表示这个权限,你也可以理解其实就是id，哈哈.
```shell
## 创建一个新的节点 /test_node1
[zk: localhost:2181(CONNECTED) 23] create  /test_node1 121
Created /test_node1
## 添加授权账号
[zk: localhost:2181(CONNECTED) 24] addauth digest zookeeper:zookeeper
## 然后设置节点的ACL
[zk: localhost:2181(CONNECTED) 25] setAcl /test_node1 auth:zookeeper:zookeeper:cdrwa
cZxid = 0x13
ctime = Sun Sep 09 17:46:26 CST 2018
mZxid = 0x13
mtime = Sun Sep 09 17:46:26 CST 2018
pZxid = 0x13
cversion = 0
dataVersion = 0
aclVersion = 1
ephemeralOwner = 0x0
dataLength = 3
numChildren = 0
## 查看刚才设置的ACL
[zk: localhost:2181(CONNECTED) 26] getAcl /test_node1
'digest,'zookeeper:4lvlzsipXVaEhXMd+2qMrLc0at8=
: cdrwa
[zk: localhost:2181(CONNECTED) 27]
这里有一个坑,在使用auth授权时,一定要先执行addauth digest zookeeper:zookeeper然后再授权，否则将使用上一次的授权expression。下面举个例子。执行多次addauth digest 用户名:密码 操作,在新的节点设置auth时，将都会生效。具体例子如下：
## 创建一个新的节点 /test_node2
[zk: localhost:2181(CONNECTED) 27] create /test_node2 234
Created /test_node2
## 设置auth授权, 使用一个并不存在的 hello用户,依然成功了
[zk: localhost:2181(CONNECTED) 28] setAcl /test_node2 auth:hello:hello:cdrwa
cZxid = 0x15
ctime = Sun Sep 09 17:50:10 CST 2018
mZxid = 0x15
mtime = Sun Sep 09 17:50:10 CST 2018
pZxid = 0x15
cversion = 0
dataVersion = 0
aclVersion = 1
ephemeralOwner = 0x0
dataLength = 3
numChildren = 0
## 查看ACL 竟然是之前的zookeeper用户
[zk: localhost:2181(CONNECTED) 29] getAcl /test_node2
'digest,'zookeeper:4lvlzsipXVaEhXMd+2qMrLc0at8=
: cdrwa
## 这个时候添加 hello 用户
[zk: localhost:2181(CONNECTED) 30] addauth digest hello:hello    
## 创建新的节点 /test_node3
[zk: localhost:2181(CONNECTED) 31] create /test_node3 456                   
Created /test_node3
## 查看ACL 这个时候还是world
[zk: localhost:2181(CONNECTED) 32] getAcl /test_node3
'world,'anyone
: cdrwa
## 设置auth ACL
[zk: localhost:2181(CONNECTED) 33] setAcl /test_node3 auth:hello:hello:cdrwa
cZxid = 0x17
ctime = Sun Sep 09 17:53:14 CST 2018
mZxid = 0x17
mtime = Sun Sep 09 17:53:14 CST 2018
pZxid = 0x17
cversion = 0
dataVersion = 0
aclVersion = 1
ephemeralOwner = 0x0
dataLength = 3
numChildren = 0
## 查看ACL 发现之前的zookeeper用户也存在
[zk: localhost:2181(CONNECTED) 34] getAcl /test_node3
'digest,'zookeeper:4lvlzsipXVaEhXMd+2qMrLc0at8=
: cdrwa
'digest,'hello:uXpRvPoy9gsHHSCo8uMtZmaXPIA=
: cdrwa
[zk: localhost:2181(CONNECTED) 35]
```

##### ip
基于客户端IP地址校验，限制只允许指定的客户端能操作znode。
比如，设置某个节点只允许IP为127.0.0.1的客户端能读写该写节点的数据：ip:127.0.0.1:rw
```shell
## 给node节点添加新的ACL权限
[zk: localhost:2181(CONNECTED) 8] setAcl /node ip:127.0.0.1:cdrwa 
cZxid = 0x2
ctime = Sun Sep 09 16:25:51 CST 2018
mZxid = 0x2
mtime = Sun Sep 09 16:25:51 CST 2018
pZxid = 0x5
cversion = 1
dataVersion = 0
aclVersion = 2
ephemeralOwner = 0x0
dataLength = 3
numChildren = 1
## 查看ACL
[zk: localhost:2181(CONNECTED) 9] getAcl /node
'ip,'127.0.0.1
: cdrwa
[zk: localhost:2181(CONNECTED) 10]
```
上面主要介绍了平时常用的三种scheme，除此之外，还有host、super、auth授权策略。

#### 二、**expression**
expression是验证模式，不同的scheme，expression的值也不一样。
- scheme为digest时，expression的值为：username:BASE64(SHA1(password))，
- scheme为ip时，expression的值为客户端的ip地址。
- scheme为world时，expression的值为anyone。
- scheme为auth时,expression为 username:password。

#### 三、**permission**
可以拥有的权限，cdrwa即是permission。
如：
```bash
digest:username:BASE64(SHA1(password)):cdrwa
```
CREATE(r)：创建子节点的权限
DELETE(d)：删除节点的权限
READ(r)：读取节点数据的权限
WRITE(w)：修改节点数据的权限
ADMIN(a)：设置子节点权限的权限

```shell
## 先在当前session中添加 wangsaichao:123456 用户
[zk: localhost:2181(CONNECTED) 35] addauth digest wangsaichao:123456
## 创建一个新的节点,并添加digest ACL(只能创建和删除) 因为是digest 所以密码为加密之后的
[zk: localhost:2181(CONNECTED) 36] create /test_node5 123 digest:wangsaichao:G2RdrM8e0u0f1vNCj/TI99ebRMw=:cd
Created /test_node5
## 没有WRITE权限, 设置节点数据 失败
[zk: localhost:2181(CONNECTED) 37] set /test_node5 234
Authentication is not valid : /test_node5
## 没有ADMIN权限,设置ACL失败
[zk: localhost:2181(CONNECTED) 38] setAcl /test_node5 auth:wangsaichao:123456:cdrwa
Authentication is not valid : /test_node5
## 没有READ权限, 读取节点 失败
[zk: localhost:2181(CONNECTED) 39] get /test_node5 
Authentication is not valid : /test_node5
## 具备CREATE权限，创建子节点成功
[[zk: localhost:2181(CONNECTED) 40] create /test_node5/child_01 123
Created /test_node5/child_01
## 具备DELETE权限，可以删除子节点
[zk: localhost:2181(CONNECTED) 41] delete /test_node5/child_01
[zk: localhost:2181(CONNECTED) 42]
```

### 权限命令
ZKCli ACL相关命令
| 命令    | 使用方式 | 描述         |
| ------- | -------- | ------------ |
| getAcl  | getAcl   | 读取Acl权限  |
| setAcl  | setAcl   | 设置Acl权限  |
| addauth | addauth  | 添加认证用户 |

客户提供的标准ACLs
```java
org.apache.zookeeper.ZooDefs.Ids
ZkClient client=new ZkClient("localhost:2181");
client.create("/zk/a","124",Ids.OPEN_ACL_UNSAPE,CreateMode.PERSISTENT);
```
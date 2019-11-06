### 一、ZooKeeper 实现Master选举原理
![1572775205805](D:\doc\blog\images\study\zookeeper\1572775205805.png)

![1572775222787](D:\doc\blog\images\study\zookeeper\1572775222787.png)

### 二、实现代码
#### 一、POM配置文件
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
 <modelVersion>4.0.0</modelVersion>

 <groupId>edu.dongnao.zk</groupId>
 <artifactId>zk-study</artifactId>
 <version>0.0.1-SNAPSHOT</version>
 <packaging>jar</packaging>

 <name>zk-study</name>
 <url>http://maven.apache.org</url>

 <properties> <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
 </properties>
 <dependencies>
 <dependency> <groupId>net.killa.kept</groupId>
 <artifactId>KeptCollections</artifactId>
 <version>1.0.0</version>
 </dependency>
 <dependency> <groupId>org.apache.curator</groupId>
 <artifactId>curator-recipes</artifactId>
 <version>4.2.0</version>
 </dependency>
 <dependency> <groupId>org.apache.zookeeper</groupId>
 <artifactId>zookeeper</artifactId>
 <version>3.4.13</version>
 </dependency>
 <dependency> <groupId>com.101tec</groupId>
 <artifactId>zkclient</artifactId>
 <version>0.10</version>
 </dependency>
 <dependency> <groupId>junit</groupId>
 <artifactId>junit</artifactId>
 <version>4.2</version>
 <scope>test</scope>
 </dependency> </dependencies>
 <build> <plugins> <plugin> <groupId>org.apache.maven.plugins</groupId>
 <artifactId>maven-compiler-plugin</artifactId>
 <configuration> <source>1.8</source>
 <target>1.8</target>
 <encoding>UTF-8</encoding>
 </configuration> </plugin> </plugins> </build></project>
```

#### 二、Master选举应用
```java
package edu.dongnao.zk.zkclient;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import org.I0Itec.zkclient.IZkDataListener;
import org.I0Itec.zkclient.ZkClient;
import org.I0Itec.zkclient.exception.ZkNodeExistsException;

public class MasterElectionDemo {

   class Server {

      private String cluster, name, address;

 private final String masterPath, value;

 private String master;

 private ZkClient client; 

 public Server(String cluster, String name, String address) {
         super();
 this.cluster = cluster;
 this.name = name;
 this.address = address;
  masterPath = "/" + this.cluster + "/master";
  value = "name:" + this.name + " address:" + this.address;
  client = new ZkClient("localhost:2181");
  client.setZkSerializer(new MyZkSerializer());

  String serversPath = "/" + this.cluster + "/servers";
  client.createPersistent(serversPath, true);

  String serverPath = serversPath + "/" + name;
  client.createEphemeral(serverPath, value);

 new Thread(new Runnable() {
            @Override
  public void run() {
               electionMaster(client);
  }
         }).start();

  }

      public void electionMaster(ZkClient client) {
         try {
            client.createEphemeral(masterPath, value);
  master = client.readData(masterPath);
  System.out.println(value + "创建节点成功，成为Master");
  } catch (ZkNodeExistsException e) {
            master = client.readData(masterPath);
  System.out.println("Master为：" + master);
  }

         // 为阻塞自己等待而用
  CountDownLatch cdl = new CountDownLatch(1);

  // 注册watcher
  IZkDataListener listener = new IZkDataListener() {

            @Override
  public void handleDataDeleted(String dataPath) throws Exception {
               System.out.println("-----监听到节点被删除");
  cdl.countDown();
  }

            @Override
  public void handleDataChange(String dataPath, Object data) throws Exception {

            }
         };

  client.subscribeDataChanges(masterPath, listener);

  // 让自己阻塞
  if (client.exists(masterPath)) {
            try {
               cdl.await();
  } catch (InterruptedException e1) {
               e1.printStackTrace();
  }
         }
         // 醒来后，取消watcher
  client.unsubscribeDataChanges(masterPath, listener);
  // 递归调自己（下一次选举）
  electionMaster(client);
  }

      public void close() {
         client.close();
  }

   }

   public static void main(String[] args) {
      // 测试时，依次开启多个Server实例java进程，然后停止获取的master的节点，看谁抢到Master
  Server s1 = new MasterElectionDemo().new Server("cluster1", "server1", "192.168.1.11:8991");
  Server s2 = new MasterElectionDemo().new Server("cluster1", "server2", "192.168.1.11:8992");
  Server s3 = new MasterElectionDemo().new Server("cluster1", "server3", "192.168.1.11:8993");
  Server s4 = new MasterElectionDemo().new Server("cluster1", "server4", "192.168.1.11:8994");
  ScheduledThreadPoolExecutor scheduled = new ScheduledThreadPoolExecutor(1);
  scheduled.schedule(()->{
         System.out.println("关闭s1");
  s1.close();
  }, 5, TimeUnit.SECONDS);
  scheduled.schedule(()->{
         System.out.println("关闭s2");
  s2.close();
  }, 10, TimeUnit.SECONDS);
  scheduled.schedule(()->{
         System.out.println("关闭s3");
  s3.close();
  }, 15, TimeUnit.SECONDS);

 try {
         Thread.currentThread().join();
  } catch (InterruptedException e) {
         e.printStackTrace();
  }
   }
}
```
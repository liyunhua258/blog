### 一、Zookeeper实现分布式锁方式一

![1572774240857](D:\doc\blog\images\study\zookeeper\1572774240857.png)
原理：节点不可重名+watch
缺点：惊群效应

<img src="D:\doc\blog\images\study\zookeeper\1572774251791.png" alt="1572774251791" style="zoom: 80%;" />

- 项目pom文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>edu.dongnao.zk</groupId>
    <artifactId>zk-study02</artifactId>
    <version>1.0-SNAPSHOT</version>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>8</source>
                    <target>8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <name>zk-study02</name>
    <url>http://maven.apache.org</url>

 <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.4.13</version>
        </dependency>

        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.10</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>3.8.1</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- 分布锁一实现类
```java
package edu.dongnao.zk.zkclient;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import org.I0Itec.zkclient.IZkDataListener;
import org.I0Itec.zkclient.ZkClient;
import org.I0Itec.zkclient.exception.ZkNodeExistsException;

public class ZkDistributeLock implements Lock {

   private String lockPath;

 private ZkClient client;

 public ZkDistributeLock(String lockPath) {
      if(lockPath ==null || lockPath.trim().equals("")) {
         throw new IllegalArgumentException("patch不能为空字符串");
  }
      this.lockPath = lockPath;

  client = new ZkClient("localhost:2181");
  client.setZkSerializer(new MyZkSerializer());
  }

   @Override
  public boolean tryLock() { // 不会阻塞
  // 创建节点
  try {
         client.createEphemeral(lockPath);
  } catch (ZkNodeExistsException e) {
         return false;
  }
      return true;
  }

   @Override
  public void unlock() {
      client.delete(lockPath);
  }

   @Override
  public void lock() { // 如果获取不到锁，阻塞等待
  if (!tryLock()) {
         // 没获得锁，阻塞自己
  waitForLock();
  // 再次尝试
  lock();
  }

   }

   private void waitForLock() {
      final CountDownLatch cdl = new CountDownLatch(1);

  IZkDataListener listener = new IZkDataListener() {
         @Override
  public void handleDataDeleted(String dataPath) throws Exception {
            System.out.println("----收到节点被删除了-------------");
  cdl.countDown();
  }

         @Override
  public void handleDataChange(String dataPath, Object data)
               throws Exception {
         }
      };

  client.subscribeDataChanges(lockPath, listener);

  // 阻塞自己
  if (this.client.exists(lockPath)) {
         try {
            cdl.await();
  } catch (InterruptedException e) {
            e.printStackTrace();
  }
      }
      // 取消注册
  client.unsubscribeDataChanges(lockPath, listener);
  }

   @Override
  public void lockInterruptibly() throws InterruptedException {
      // TODO Auto-generated method stub

  }

   @Override
  public boolean tryLock(long time, TimeUnit unit)
         throws InterruptedException {
      // TODO Auto-generated method stub
  return false;
  }

   @Override
  public Condition newCondition() {
      // TODO Auto-generated method stub
  return null;
  }
}
```

- 分布式锁一测试类
```java
public static void main(String[] args) {
    int servers=2;
 for(int i=0;i<servers;i++){
        final int fi=i;
 new Thread(new Runnable(){
            public void run() {
                ZkDistributedLock zkDistributeLock =new ZkDistributedLock("/zk/lock");
  zkDistributeLock.lock();
 long millis=System.currentTimeMillis();
  System.out.println("我拿锁了，我是"+ fi+",当前时间："+millis);

 try {
                    Thread.sleep(2000L);
  } catch (InterruptedException e) {
                    e.printStackTrace();
  }

                long peris=System.currentTimeMillis()-millis;
  System.out.println("我是"+ fi+",我要释放锁了，用时："+peris);
  zkDistributeLock.unlock();
  }
        }).start();
  }
}
```


### Zookeeper实现分布锁方式二
<img src="D:\doc\blog\images\study\zookeeper\1572774316071.png" alt="1572774316071" style="zoom:80%;" />
原理：取号+最小号得+Watch

<img src="D:\doc\blog\images\study\zookeeper\1572774332747.png" alt="1572774332747" style="zoom: 80%;" />

- POM文件同上
- 分布式实现类二

``` java
package edu.dongnao.zk.zkclient;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import org.I0Itec.zkclient.IZkDataListener;
import org.I0Itec.zkclient.ZkClient;
import org.I0Itec.zkclient.exception.ZkNodeExistsException;

public class ZkDistributeImproveLock implements Lock {

   /*
 * 利用临时顺序节点来实现分布式锁
 * 获取锁：取排队号（创建自己的临时顺序节点），然后判断自己是否是最小号，如是，则获得锁；不是，则注册前一节点的watcher,阻塞等待
 * 释放锁：删除自己创建的临时顺序节点
 */  private String lockPath;

 private ZkClient client;
 private ThreadLocal<String> currentPath = new ThreadLocal<String>();

 private ThreadLocal<String> beforePath = new ThreadLocal<String>();
  // 锁重入计数器
  private ThreadLocal<Integer> reenterCount = ThreadLocal.withInitial(()->0);

 public ZkDistributeImproveLock(String lockPath) {
      if(lockPath == null || lockPath.trim().equals("")) {
         throw new IllegalArgumentException("patch不能为空字符串");
  }
      this.lockPath = lockPath;
  client = new ZkClient("localhost:2181");
  client.setZkSerializer(new MyZkSerializer());
 if (!this.client.exists(lockPath)) {
         try {
            this.client.createPersistent(lockPath, true);
  } catch (ZkNodeExistsException e) {

         }
      }
   }

   @Override
  public boolean tryLock() {
      System.out.println(Thread.currentThread().getName() + "-----尝试获取分布式锁");
 if (this.currentPath.get() == null || !client.exists(this.currentPath.get())) {
         String node = this.client.createEphemeralSequential(lockPath + "/", "locked");
  currentPath.set(node);
  reenterCount.set(0);
  }

      // 获得所有的子
  List<String> children = this.client.getChildren(lockPath);

  // 排序list
  Collections.sort(children);

  // 判断当前节点是否是最小的
  if (currentPath.get().equals(lockPath + "/" + children.get(0))) {
         // 锁重入计数
  reenterCount.set(reenterCount.get() + 1);
  System.out.println(Thread.currentThread().getName() + "-----获得分布式锁");
 return true;  } else {
         // 取到前一个
  // 得到字节的索引号
  int curIndex = children.indexOf(currentPath.get().substring(lockPath.length() + 1));
  String node = lockPath + "/" + children.get(curIndex - 1);
  beforePath.set(node);
  }
      return false;
  }

   @Override
  public void lock() {
      if (!tryLock()) {
         // 阻塞等待
  waitForLock();
  // 再次尝试加锁
  lock();
  }
   }

   private void waitForLock() {

      CountDownLatch cdl = new CountDownLatch(1);

  // 注册watcher
  IZkDataListener listener = new IZkDataListener() {

         @Override
  public void handleDataDeleted(String dataPath) throws Exception {
            System.out.println(Thread.currentThread().getName() + "-----监听到节点被删除，分布式锁被释放");
  cdl.countDown();
  }

         @Override
  public void handleDataChange(String dataPath, Object data) throws Exception {

         }
      };

  client.subscribeDataChanges(this.beforePath.get(), listener);

  // 怎么让自己阻塞
  if (this.client.exists(this.beforePath.get())) {
         try {
            System.out.println(Thread.currentThread().getName() + "-----分布式锁没抢到，进入阻塞状态");
  cdl.await();
  System.out.println(Thread.currentThread().getName() + "-----释放分布式锁，被唤醒");
  } catch (InterruptedException e) {
            e.printStackTrace();
  }
      }
      // 醒来后，取消watcher
  client.unsubscribeDataChanges(this.beforePath.get(), listener);
  }

   @Override
  public void unlock() {
      System.out.println(Thread.currentThread().getName() + "-----释放分布式锁");
 if(reenterCount.get() > 1) {
         // 重入次数减1，释放锁
  reenterCount.set(reenterCount.get() - 1);
 return;  }
      // 删除节点
  if(this.currentPath.get() != null) {
         this.client.delete(this.currentPath.get());
 this.currentPath.set(null);
 this.reenterCount.set(0);
  }
   }

   @Override
  public void lockInterruptibly() throws InterruptedException {

   }

   @Override
  public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {

      return false;
  }

   @Override
  public Condition newCondition() {
      return null;
  }
}
```

- 分布锁二测试入口
```java
public static void main(String[] args) {
    int servers=2;
 for(int i=0;i<servers;i++){
        final int fi=i;
 new Thread(new Runnable(){
            public void run() {
                ZkDistributedLock zkDistributeLock =new ZkDistributedLock("/zk/lock");
  zkDistributeLock.lock();
 long millis=System.currentTimeMillis();
  System.out.println("我拿锁了，我是"+ fi+",当前时间："+millis);

 try {
                    Thread.sleep(2000L);
  } catch (InterruptedException e) {
                    e.printStackTrace();
  }

                long peris=System.currentTimeMillis()-millis;
  System.out.println("我是"+ fi+",我要释放锁了，用时："+peris);
  zkDistributeLock.unlock();
  }
        }).start();
  }
}
```
### 一、分布式队列应用场景
**解耦**
<img src="https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572774815859.png" alt="1572774815859" style="zoom: 50%;" />

**异步**
![1572774845905](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572774845905.png)

**削峰填谷**
![1572774856591](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572774856591.png)

### 二、分布式队列实现逻辑
**先进先出**
![1572774868639](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572774868639.png)

**顺序节点**
<img src="https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572774884346.png" alt="1572774884346" style="zoom:50%;" />

**入队逻辑**
<img src="https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572774923286.png" alt="1572774923286" style="zoom:50%;" />

**出队逻辑**
<img src="https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572774944539.png" alt="1572774944539" style="zoom:50%;" />

**队列总体实现逻辑流程图**
<img src="https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572774957371.png" alt="1572774957371" style="zoom:50%;" />

### 三、JDK阻塞队列操作

■ 阻塞队列 BlockingQueue
- 阻塞队列，线程安全的
在队列为空时的获取阻塞，在队列满时的放入阻塞。
BlockingQueue方法以四种形式出现，对于不能立即满足但可能在将来某一时刻可以满足的操作，这四种形式的处理方式不同：
第一种是抛出一个异常，
第二种是返回一个特殊值（null或false,具体取决于操作），
第三种是在操作可以成功前，无限期地阻塞当前线程，
第四种是在放弃前只在给定的最大时间限制内阻塞。

插入移除方法，四种形式总结

|      | 抛出异常  | 特殊值   | 阻塞   | 超时               |
| ---- | --------- | -------- | ------ | ------------------ |
| 插入 | add(e)    | offer(e) | put(e) | offer(e,time,unit) |
| 移除 | remove()  | poll()   | take() | poll(time,unit)    |
| 检查 | element() | peek     | 不可用 | 不可用             |

### 四、代码实现
**1、POM文件**
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
 <modelVersion>4.0.0</modelVersion>

 <groupId>edu.dongnao.zk</groupId>
 <artifactId>zkstudy03</artifactId>
 <version>0.0.1-SNAPSHOT</version>
 <packaging>jar</packaging>

 <name>zkstudy03</name>
 <url>http://maven.apache.org</url>

 <properties> <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
 </properties>
 <dependencies> <dependency> <groupId>org.apache.zookeeper</groupId>
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

**2、分布式队列实现类**
```java
package edu.dongnao.zk.zkclient;
import java.util.AbstractQueue;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.stream.Collectors;
import org.I0Itec.zkclient.IZkChildListener;
import org.I0Itec.zkclient.ZkClient;
import org.I0Itec.zkclient.exception.ZkNodeExistsException;

/**
 * ZKDistributeQueue 分布式队列实现
  * 使用zk指定的目录作为队列，子节点作为任务。
  * 生产者能往队列中添加任务，消费者能往队列中消费任务。
  * 持久节点作为队列，持久顺序节点作为任务。
  * 
 */
public class ZkDistributeQueue extends AbstractQueue<String> implements BlockingQueue<String> , java.io.Serializable {
   /**
 *    */
  private static final long serialVersionUID = 1L;
  /**
 * zookeeper客户端操作实例
  */
  private ZkClient zkClient;
  /**
 * 定义在zk上的znode，作为分布式队列的根目录。
  */
  private String queueRootNode;
 private static final String default_queueRootNode = "/distributeQueue";
  /**队列写锁节点*/
  private String queueWriteLockNode;
  /**队列读锁节点*/
  private String queueReadLockNode;
  /**
 * 子目录存放队列下的元素，用顺序节点作为子节点。
  */
  private String queueElementNode;
  /**
 * ZK服务的连接字符串，hostname:port形式的字符串
  */
  private String zkConnUrl;
 private static final String default_zkConnUrl = "localhost:2181";
  /** 
* 队列容量大小，默认Integer.MAX_VALUE，无界队列。
  **/
  private static final int default_capacity = Integer.MAX_VALUE;
 private int capacity;
  /**
 * 控制进程访问的分布式锁
  */
  final Lock distributeWriteLock;
 final Lock distributeReadLock;
 public ZkDistributeQueue() {
      this(default_zkConnUrl, default_queueRootNode, default_capacity);
  }

   public ZkDistributeQueue(String zkServerUrl, String rootNodeName, int initCapacity) {
      if (zkServerUrl == null) throw new IllegalArgumentException("zkServerUrl");
 if (rootNodeName == null) throw new IllegalArgumentException("rootNodeName");
 if (initCapacity <= 0) throw new IllegalArgumentException("initCapacity");
 this.zkConnUrl = zkServerUrl;
 this.queueRootNode = rootNodeName;
 this.capacity = initCapacity;
  init();
  distributeWriteLock = new ZkDistributeImproveLock(queueWriteLockNode);
  distributeReadLock = new ZkDistributeImproveLock(queueReadLockNode);
  }

   /**
 * 初始化队列信息
  */
  private void init() {
      queueWriteLockNode = queueRootNode+"/writeLock";
  queueReadLockNode = queueRootNode+"/readLock";
  queueElementNode = queueRootNode+"/element";
  zkClient = new ZkClient(zkConnUrl);
  zkClient.setZkSerializer(new MyZkSerializer());
 if (!this.zkClient.exists(queueElementNode)) {
         try {
            this.zkClient.createPersistent(queueElementNode, true);
  } catch (ZkNodeExistsException e) {

         }
      }
   }

   /**
 * 获取队列头部元素，但不执行删除，如果队列为空则返回null。
  * 
    */
  @Override
  public String peek() {
      List<String> children = zkClient.getChildren(queueElementNode);
 if(children != null && !children.isEmpty()) {
         children = children.stream().sorted().collect(Collectors.toList());
  String firstChild = children.get(0);
  String elementData = zkClient.readData(queueElementNode+"/"+firstChild);
 return elementData;
  }
      return null;
  }
   /**
 * 往zk中添加元素
  * @param e
  */
  private void enqueue(String e) {
      zkClient.createPersistentSequential(queueElementNode+"/", e);
  }
   /**
 * 从zk中删除元素
  * @param e
  * @return
  */
  private boolean dequeue(String e) {
      boolean result = zkClient.delete(e);
 return result;
  }

   // ==============================特殊值操作============================
  /**
 * 获取并删除头部元素，如果队列为空则返回null
 */  @Override
  public String poll() {
      String firstChild = peek();
  distributeReadLock.lock();
 try {
         if (firstChild == null) {
            return null;
  }
         boolean result = dequeue(firstChild);
 if (!result) {
            return null;
  } 
      } finally {
         distributeReadLock.unlock();
  }
      return firstChild;
  }

   /**
 * 容量足够立即将指定的元素插入到队列中，成功时返回true，容量不够，则返回false。
  * 
    */
  @Override
  public boolean offer(String e) {
      checkElement(e);
  // 判断是否可以添加任务，不能则抛出异常
  distributeWriteLock.lock();
 try {
         if(size() < capacity) {
            enqueue(e);
 return true;  }
      } finally {
         distributeWriteLock.unlock();
  }
      return false;
  }

   // ==============================抛出异常操作============================
  /**
 * 获取并删除队列头部元素，当队列为空时抛出异常
  */
  @Override
  public String remove() {
      if(size() <= 0) {
         throw new IllegalDistributeQueueStateException(
               IllegalDistributeQueueStateException.State.empty);
  }
      distributeReadLock.lock();
  String firstChild;
 try {
         firstChild = poll();
 if (firstChild == null) {
            throw new IllegalDistributeQueueStateException("移除失败");
  } 
      } finally {
         distributeReadLock.unlock();
  }
      return firstChild;
  }

   /**
 * 容量足够的情况下，将指定的元素加入队列，插入成功返回true，
  * 容量不足够的情况下，抛出IllegalDistributeQueueStateException异常。
  */
  @Override
  public boolean add(String e) {
      checkElement(e);
  distributeWriteLock.lock();
 try {
         // 判断是否可以添加任务，不能则抛出异常
  if(size() >= capacity) {
            throw new IllegalDistributeQueueStateException(
                  IllegalDistributeQueueStateException.State.full);
  }
         return offer(e);
  } catch (Exception e1) {
         e1.printStackTrace();
  } finally {
         distributeWriteLock.unlock();
  }
      return false;
  }

   // 阻塞操作
  @Override
  public void put(String e) throws InterruptedException {
      checkElement(e);
  distributeWriteLock.lock();
 try {
         if(size() < capacity) {    // 容量足够
  enqueue(e);
  System.out.println(Thread.currentThread().getName() + "-----往队列放入了元素");
  }else { // 容量不够，阻塞，监听元素出队
  waitForRemove();
  put(e);
  }
      } finally {
         distributeWriteLock.unlock();
  }
   }

   /**
 * 队列容量满了，不能再插入元素，阻塞等待队列移除元素。
  */
  private void waitForRemove() {
      CountDownLatch cdl = new CountDownLatch(1);
  // 注册watcher
  IZkChildListener listener = new IZkChildListener() {
         @Override
  public void handleChildChange(String parentPath, List<String> currentChilds) throws Exception {
            if(currentChilds.size() < capacity) {  // 有任务移除，激活等待的添加操作
  cdl.countDown();
  System.out.println(Thread.currentThread().getName() + "-----监听到队列有元素移除，唤醒阻塞生产者线程");
  }
         }
      };
  zkClient.subscribeChildChanges(queueElementNode, listener);
 try {
         // 确保队列是满的
  if(size() >= capacity) {
            System.out.println(Thread.currentThread().getName() + "-----队列已满，阻塞等待队列元素释放");
  cdl.await(); // 阻塞等待元素被移除
  }
      } catch (InterruptedException e) {
         e.printStackTrace();
  }
      zkClient.unsubscribeChildChanges(queueElementNode, listener);
  }

   @Override
  public String take() throws InterruptedException {
      distributeReadLock.lock();
 try {
         List<String> children = zkClient.getChildren(queueElementNode);
 if(children != null && !children.isEmpty()) {
            children = children.stream().sorted().collect(Collectors.toList());
  String takeChild = children.get(0);
  String childNode = queueElementNode+"/"+takeChild;
  String elementData = zkClient.readData(childNode);
  dequeue(childNode);
  System.out.println(Thread.currentThread().getName() + "-----移除队列元素");
 return elementData;
  }else {
            waitForAdd(); // 阻塞等待队列有元素加入
  return take();
  } 
      } finally {
         distributeReadLock.unlock();
  }
   }

   /**
 * 队列空了，没有元素可以移除，阻塞等待元素添加到队列中。
  */
  private void waitForAdd() {
      CountDownLatch cdl = new CountDownLatch(1);
  // 注册watcher
  IZkChildListener listener = new IZkChildListener() {
         @Override
  public void handleChildChange(String parentPath, List<String> currentChilds) throws Exception {
            if(currentChilds.size() > 0) { // 有任务了，激活等待的移除操作
  cdl.countDown();
  System.out.println(Thread.currentThread().getName() + "-----监听到队列有元素加入，唤醒阻塞消费者线程");
  }
         }
      };
  zkClient.subscribeChildChanges(queueElementNode, listener);
 try {
         // 确保队列是空的
  if(size() <= 0) {
            System.out.println(Thread.currentThread().getName() + "-----队列已空，等待元素加入队列");
  cdl.await(); // 阻塞等待
  System.out.println(Thread.currentThread().getName() + "-----队列已有元素，线程被唤醒");
  }
      } catch (InterruptedException e) {
         e.printStackTrace();
  }
      zkClient.unsubscribeChildChanges(queueElementNode, listener);
  }

   private static void checkElement(String v) {
        if (v == null) throw new NullPointerException();
 if("".equals(v.trim())) {
           throw new IllegalArgumentException("不能使用空格");
  }
        if(v.startsWith(" ") || v.endsWith(" ")) {
           throw new IllegalArgumentException("前后不能包含空格");
  }
    }

   /**
 * 返回队列中已存在的元素数量
  */
  @Override
  public int size() {
      int size = zkClient.countChildren(queueElementNode);
 return size;
  }

   /**
 * 暂不支持迭代子
  */
  @Override
  public Iterator<String> iterator() {
      throw new UnsupportedOperationException();
  }

   // 暂不支持
  @Override
  public boolean offer(String e, long timeout, TimeUnit unit) throws InterruptedException {
      return false;
  }

   @Override
  public String poll(long timeout, TimeUnit unit) throws InterruptedException {
      return null;
  }

   /**
 * 返回剩余容量，没有限制返回Integer.MAX_value。
  */
  @Override
  public int remainingCapacity() {
      int remaining = capacity - size();
 return remaining;
  }

   @Override
  public int drainTo(Collection<? super String> c) {
      return drainTo(c, size());
  }

   /**
 * 从队列移除指定大小的元素，并将移除的元素添加到指定的集合中。
  */
  @Override
  public int drainTo(Collection<? super String> c, int maxElements) {
      if(c == null) {
         throw new NullPointerException();
  }
      int transferredSize = 0;
  List<String> children = zkClient.getChildren(queueElementNode);
 if(children != null && !children.isEmpty()) {
         List<String> removeElements = children.stream().sorted().limit(maxElements).collect(Collectors.toList());
  transferredSize = removeElements.size();
  c.addAll(removeElements);
  removeElements.forEach((e)->{
            zkClient.delete(e);
  });
  }
      return transferredSize;
  }
}
```
**3、消费者测试类**
```java
package edu.dongnao.zk.test;
import static edu.dongnao.zk.test.DistributeQueueProducerTest.capacity;
import static edu.dongnao.zk.test.DistributeQueueProducerTest.queueRootNode;
import static edu.dongnao.zk.test.DistributeQueueProducerTest.zkConnUrl;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.TimeUnit;
import edu.dongnao.zk.zkclient.ZkDistributeQueue;

public class DistributeQueueConsumerTest {

   public static void main(String[] args) {
      satrtConsumer();
  }

   public static void satrtConsumer() {
      // 服务集群数
  int service = 2;
  // 并发数
  int requestSize = 2;
  CyclicBarrier requestBarrier = new CyclicBarrier(requestSize * service);
  // 多线程模拟分布式环境下消费者
  for (int i = 0; i < service; i++) {
         new Thread(new Runnable() {    // 进程模拟线程
  public void run() {
               // 模拟分布式集群的场景
  BlockingQueue<String> queue = new ZkDistributeQueue(zkConnUrl, queueRootNode, capacity);

  System.out.println(Thread.currentThread().getName() + "---------消费者服务器，已准备好---------------");
                for(int i =0; i < requestSize; i++) {  // 操作模拟线程
  new Thread(new Runnable() {
                     @Override
  public void run() {
                        try {
                           // 等待service台服务，requestSize个请求 一起出发
  requestBarrier.await();
  } catch (InterruptedException | BrokenBarrierException e) {
                           e.printStackTrace();
  }
                        while(true) {
                           try {
                              queue.take();
  System.out.println(Thread.currentThread().getName() + "-----进入睡眠状态");
  TimeUnit.SECONDS.sleep(3);
  System.out.println(Thread.currentThread().getName() + "-----睡眠状态，醒来");
  } catch (InterruptedException e) {
                              e.printStackTrace();
  }
                        }
                     }

                  }, Thread.currentThread().getName()+"-request-" + i).start();
  }
            }
         }, "consummerServer-" + i).start();
  }

      try {
         Thread.currentThread().join();
  } catch (InterruptedException e) {
         e.printStackTrace();
  }
   }

}
```

**4、生产者测试类**
```java
package edu.dongnao.zk.test;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.TimeUnit;
import edu.dongnao.zk.zkclient.ZkDistributeQueue;

public class DistributeQueueProducerTest {
   public static final String queueRootNode = "/distributeQueue";
 public static final String zkConnUrl = "localhost:2181";
 public static final int capacity = 20;
 public static void main(String[] args) {
      startProducer();
  }

   public static void startProducer() {
      // 服务集群数
  int service = 2;
  // 并发数
  int requestSize = 2;
  CyclicBarrier requestBarrier = new CyclicBarrier(requestSize * service);
  // 多线程模拟分布式环境下生产者
  for (int i = 0; i < service; i++) {
         new Thread(new Runnable() {
            public void run() {
               // 模拟分布式集群的场景
  BlockingQueue<String> queue = new ZkDistributeQueue(zkConnUrl, queueRootNode, capacity);
  System.out.println(Thread.currentThread().getName() + "---------生产者服务器，已准备好---------------");
                for(int i =0; i < requestSize; i++) {
                  new Thread(new Runnable() {
                     @Override
  public void run() {
                        try {
                           // 等待service台服务，requestSize个请求 一起出发
  requestBarrier.await();
  } catch (InterruptedException | BrokenBarrierException e) {
                           e.printStackTrace();
  }
                        while(true) {
                           try {
                              queue.put("123");
  System.out.println(Thread.currentThread().getName() + "-----进入睡眠状态");
  TimeUnit.SECONDS.sleep(3);
  System.out.println(Thread.currentThread().getName() + "-----睡眠状态，醒来");
  } catch (InterruptedException e) {
                              e.printStackTrace();
  }
                        }
                     }

                  }, Thread.currentThread().getName()+"-request-" + i).start();
  }
            }
         }, "producerServer-" + i).start();
  }

      try {
         Thread.currentThread().join();
  } catch (InterruptedException e) {
         e.printStackTrace();
  }
   }

}
```
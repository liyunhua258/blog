### 一、Zookeeper实现配置中心原理和操作
<img src="https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572775072261.png" alt="1572775072261" style="zoom:50%;" />
<img src="https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572775093939.png" alt="1572775093939" style="zoom:50%;" />

<img src="https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572775104542.png" alt="1572775104542" style="zoom:50%;" />
<img src="https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/zookeeper/1572775114040.png" alt="1572775114040" style="zoom:50%;" />


### 二、POM配置文件
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

### 三、配置中心实现类
```java
package edu.dongnao.zk.zkclient;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.Set;
import java.util.concurrent.ScheduledFuture;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.stream.Collectors;
import org.I0Itec.zkclient.IZkChildListener;
import org.I0Itec.zkclient.IZkDataListener;
import org.I0Itec.zkclient.ZkClient;
import org.I0Itec.zkclient.exception.ZkNoNodeException;
import org.I0Itec.zkclient.exception.ZkNodeExistsException;

/**
 * ConfigureCenter * 配置中心实现
  */
public class ZkConfigureCenter implements ConfigureWriter, ConfigureReader {

   private String confRootPath;
 private String confFilePath;
 private String fileLockPath;
 private static final String default_confRootPath = "/distributeConfigure";

 private ZkClient client;
 public ZkConfigureCenter() {
      this(default_confRootPath);
  }

   public ZkConfigureCenter(String path) {
      if(path == null || path.trim().equals("")) {
         throw new IllegalArgumentException("patch不能为空字符串");
  }
      confRootPath = path;
  confFilePath = confRootPath+"/cnfFile";
  fileLockPath = confRootPath+"/writeLock";
  client = new ZkClient("localhost:2181");
  client.setZkSerializer(new MyZkSerializer());
 if (!this.client.exists(confFilePath)) {
         try {
            this.client.createPersistent(confFilePath, true);
  } catch (ZkNodeExistsException e) {

         }
      }
   }

   private void checkElement(String v) {
        if (v == null) throw new NullPointerException();
 if("".equals(v.trim())) {
           throw new IllegalArgumentException("不能使用空格");
  }
        if(v.startsWith(" ") || v.endsWith(" ")) {
           throw new IllegalArgumentException("前后不能包含空格");
  }
    }

   @Override
  public String createCnfFile(String fileName, Properties items) {
      checkElement(fileName);
  // 创建配置文件Node
  String cfgNode = confFilePath+"/"+fileName;
 if(client.exists(cfgNode)) {
         throw new IllegalArgumentException("["+fileName+"]文件已存在！");
  }
      client.createPersistent(cfgNode, true);
  // 创建配置文件中的配置项
  if(items == null) {return cfgNode;}
      Lock distributeWriteLock = new ZkDistributeImproveLock(fileLockPath+"/"+fileName);
  distributeWriteLock.lock();
 try {
         items.keySet().iterator();
  Set<Map.Entry<Object, Object>> entrySet = items.entrySet();
 for (Map.Entry<Object, Object> entry : entrySet) {
            System.out.println(entry.getKey() + "=" + entry.getValue());
  String cfgItemNode = cfgNode +"/"+ entry.getKey().toString();
  client.createPersistent(cfgItemNode, entry.getValue());
  } 
      } finally {
         distributeWriteLock.unlock();
  }
      return cfgNode;
  }

   @Override
  public void deleteCnfFile(String fileName) {
      checkElement(fileName);
  String cfgNode = confFilePath+"/"+fileName;
  Lock distributeWriteLock = new ZkDistributeImproveLock(fileLockPath+"/"+fileName);
  distributeWriteLock.lock();
 try {
         client.deleteRecursive(cfgNode);
  } finally {
         distributeWriteLock.unlock();
  }
   }

   @Override
  public void modifyCnfItem(String fileName, Properties items) {
      checkElement(fileName);
  // 获取子节点信息
  String cfgNode = confFilePath+"/"+fileName;
  // 简单粗暴的实现
  if(items == null) {throw new NullPointerException("要修改的配置项不能为空");}
      items.keySet().iterator();
  Set<Map.Entry<Object, Object>> entrySet = items.entrySet();
  Lock distributeWriteLock = new ZkDistributeImproveLock(fileLockPath+"/"+fileName);
  distributeWriteLock.lock();
 try {
           // 获取zk中已存在的配置信息
  List<String> itemNodes = client.getChildren(cfgNode);
  Set<String> existentItemSet = itemNodes.stream().collect(Collectors.toSet());
 for (Map.Entry<Object, Object> entry : entrySet) {
            System.out.println(entry.getKey() + "=" + entry.getValue());
  String itemName = entry.getKey().toString();
  String itemData = entry.getValue().toString();
  String cfgItemNode = cfgNode + "/" + itemName;
 if(existentItemSet.contains(itemName)) {// zk中存在的配置项
  String itemNodeData = client.readData(cfgItemNode);
 if(! eql(itemNodeData, itemData)) { // 数据不一致才需要修改
  client.writeData(cfgItemNode, itemData);
  }
               existentItemSet.remove(itemName); // 剩下的就是需要删除的配置项
  } else { // zk中不存在的配置项，新的配置项
  client.createPersistent(cfgItemNode, itemData);
  }
         }

         // existentItemSet中剩下的就是需要删除的
  if(!existentItemSet.isEmpty()) {
            for(String itemName : existentItemSet) {
               String cfgItemNode = cfgNode + "/" + itemName;
  client.delete(cfgItemNode);
  }
         }
      } finally {
         distributeWriteLock.unlock();
  }
   }

   private boolean eql(String a, String b) {
      if(a == null && b == null) {
         return true;
  }
      if(a != null) {
         return a.equals(b);
  }
      if(b != null) {
         return b.equals(a);
  }
      return false;
  }

   @Override
  public Properties loadCnfFile(String fileName) {
      if(! fileName.startsWith("/")) {
         fileName = confFilePath+"/"+fileName;
  }
      return loadNodeCnfFile(fileName);
  }

   private Properties loadNodeCnfFile(String cfgNode) {
      checkElement(cfgNode);
 if(! client.exists(cfgNode)) {
         throw new ZkNoNodeException(cfgNode);
  }
      // 获取子节点信息
  List<String> itemNodes = client.getChildren(cfgNode);
  // 读取配置信息，并装载到Properties中
  if(itemNodes == null || itemNodes.isEmpty()) {
         return new Properties();
  }
      Properties file = new Properties();
  itemNodes.stream().forEach((e)->{
         String itemNameNode = cfgNode + "/" + e;
  String data = client.readData(itemNameNode, true);
  file.put(e, data);
  });
 return file;
  }

   @Override
  public void watchCnfFile(String fileName, ChangeHandler changeHandler) {
      if(! fileName.startsWith("/")) {
         fileName = confFilePath+"/"+fileName;
  }
      final String fileNodePath = fileName;
  // 读取文件
  Properties p = loadNodeCnfFile(fileNodePath);
 if(p != null) {
         // 合并5秒配置项变化，5秒内变化只触发一次处理事件
  int waitTime = 5;
 final ScheduledThreadPoolExecutor scheduled = new ScheduledThreadPoolExecutor(1);
  scheduled.setRemoveOnCancelPolicy(true);
 final List<ScheduledFuture<?>> futureList = new ArrayList<ScheduledFuture<?>>();
  Set<Map.Entry<Object, Object>> entrySet = p.entrySet();
 for (Map.Entry<Object, Object> entry : entrySet) {
            System.out.println("监控："+fileNodePath+"/"+entry.getKey().toString());
  client.subscribeDataChanges(fileNodePath+"/"+entry.getKey().toString(), new IZkDataListener() {
               @Override
  public void handleDataDeleted(String dataPath) throws Exception {
                  System.out.println("触发删除："+dataPath);
  triggerHandler(futureList, scheduled, waitTime, fileNodePath, changeHandler);
  }

               @Override
  public void handleDataChange(String dataPath, Object data) throws Exception {
                  System.out.println("触发修改："+dataPath);
  triggerHandler(futureList, scheduled, waitTime, fileNodePath, changeHandler);
  }
            });
  }
         client.subscribeChildChanges(fileNodePath, new IZkChildListener() {
            @Override
  public void handleChildChange(String parentPath, List<String> currentChilds) throws Exception {
               System.out.println("触发子节点："+parentPath);
  triggerHandler(futureList, scheduled, waitTime, fileNodePath, changeHandler);
  }
         });
  }

   }

   /**
 * 合并修改变化事件，5秒钟内发生变化的合并到一个事件进行
  * @param futureList 装有定时触发任务的列表
  * @param scheduled 定时任务执行器
  * @param waitTime 延迟时间，单位秒
  * @param fileName zk配置文件的节点
  * @param changeHandler 事件处理器
  */
  private void triggerHandler(List<ScheduledFuture<?>> futureList, ScheduledThreadPoolExecutor scheduled, int waitTime, String fileName, ChangeHandler changeHandler) {
      if(futureList != null && !futureList.isEmpty()) {
         for(int i = 0 ; i < futureList.size(); i++) {
            ScheduledFuture<?> future = futureList.get(i);
 if(future != null && !future.isCancelled() && !future.isDone()) {
               future.cancel(true);
  futureList.remove(future);
  i--;
  }
         }
      }
      ScheduledFuture<?> future = scheduled.schedule(()->{
         Properties p = loadCnfFile(fileName);
  changeHandler.itemChange(p);
  }, waitTime, TimeUnit.SECONDS);
  futureList.add(future);
  }

}
```

### 四、配置读取器
```java
package edu.dongnao.zk.zkclient;
/**
 * ConfigureReader * */

import java.util.Properties;

/**
 * 配置文件读取器
  * ConfigureReader
 */public interface ConfigureReader {
   /**
 * 读取配置文件
  * @param fileName 配置文件名称
  * @param ChangeHandler 配置发生变化的处理器
  * @return 如果存在文件配置，则返回Properties对象，不存在返回null
 */  Properties loadCnfFile(String fileName);
  /**
 * 监听配置文件变化，此操作只需要调用一次。
  * @param fileName
  * @param changeHandler
  */
  void watchCnfFile(String fileName, ChangeHandler changeHandler);
  /**
 * 配置文件变化处理器
  * ChangeHandler
 */  interface ChangeHandler {
      /**
 * 配置文件发生变化后给一个完整的属性对象
  * @param newProp
  */
  void itemChange(Properties newProp);
  }
}
```

### 五、配置写入器
```java
package edu.dongnao.zk.zkclient;
import java.util.Properties;

/**
 * ConfigureCenter * */
public interface ConfigureWriter {
   /**
 * 创建一个新的配置文件
  * @param fileName 文件名称
  * @param items 配置项
  * @return 新文件的在zk上的路径
  */
  String createCnfFile(String fileName, Properties items);
  /**
 * 删除一个配置文件
  * @param fileName
  */
  void deleteCnfFile(String fileName);
  /**
 * 修改一个配置文件
  * @param fileName
  * @param items
  */
  void modifyCnfItem(String fileName, Properties items);
  /**
 * 加载配置文件
  * @param fileName
  * @return
  */
  Properties loadCnfFile(String fileName);
}
```
### 六、测试类
```java
package edu.dongnao.zk.test.zkclient;

import java.util.Properties;
import edu.dongnao.zk.zkclient.ConfigureReader;
import edu.dongnao.zk.zkclient.ConfigureReader.ChangeHandler;
import edu.dongnao.zk.zkclient.ConfigureWriter;
import edu.dongnao.zk.zkclient.ZkConfigureCenter;

/**
 * ConfigureWriterTest * */
public class ConfigureTest {

   public static void main(String[] args) {
      // 模拟运维人员创建配置文件，引用ConfigureWriter接口操作
  ConfigureWriter writer = new ZkConfigureCenter();
  String fileName = "trade-application.properties";
  writer.deleteCnfFile(fileName); // 测试，确保配置中心没有这个问题
  Properties items = new Properties();
  items.put("abc.gc.a", "123");
  items.put("abc.gc.b", "3456");
  // 创建配置文件，内容为 properties items的内容。
  String znodePath = writer.createCnfFile(fileName, items);
  System.out.println("new file: "+znodePath);

new Thread(()->{
         readCnf();
  }).start();
 try {
         Thread.sleep(3000L);
  } catch (InterruptedException e) {
         e.printStackTrace();
  }

      // 3秒后修改文件内容，有新增、有删除、有修改
  items.put("abc.gc.a", "haha"); // 修改
  items.put("abc.gc.c", "xx"); // 新增
  items.remove("abc.gc.b"); // 删除
  writer.modifyCnfItem(fileName, items);

try {
         Thread.currentThread().join();
  } catch (InterruptedException e) {
         e.printStackTrace();
  }
   }

   /**
 * 模拟应用程序加载配置文件，监听配置文件的变化
  */
  public static void readCnf() {
      // 应用引用ConfigureReader接口进行操作
  System.out.println("读取并监听配置文件");
  ConfigureReader reader = new ZkConfigureCenter();
  String fileName = "trade-application.properties";
  Properties p = reader.loadCnfFile(fileName); // 读取配置文件
  System.out.println(p);
  // 监听配置文件
  reader.watchCnfFile(fileName, new ChangeHandler() {
         @Override
  public void itemChange(Properties newProp) {
            System.out.println("发现数据发生变化："+ newProp);
  }
      });
 try {
         Thread.currentThread().join();
  } catch (InterruptedException e) {
         e.printStackTrace();
  }
   }
}
```


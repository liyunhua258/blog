## SpringBoot解密第二篇
## 一、知识点，springboot实现自动配置的注解
```
@ConditionalOnClass  //当类路径下有指定的类的条件下
@ConditionalOnMissingClass //当类路径下没有指定的类的条件下
@ConditionalOnBean        //当容器里有指定的bean条件下
@ConditionalOnMissingBean    //当容器里没有指定的bean条件下
@ConditionalOnProperty    //指定的属性是否有指定的值
@ConditionalOnResource        //类路径是否有指定的值
@ConditionalOnWebApplication    //当前项目是Web项目的条件下
@ConditionalOnNotWebApplication    //当前项目不是Web项目的条件下
@ConditionalOnExpression    //基于SpEL表达式作为判断条件
@ConditionalOnJava        //基于JVM版本作为判断条件
@ConditionalOnJndi    //在JNDI存在的条件下查找指定的位置
@ConditionalOnResource //类路径是否有指定的值
@ConditionalOnSingleCandiate//当指定bean在容器中只有一个，或者虽然有多个但是指定首选的bean

@AutoConfigureAfter        //在加载配置的类之后再加载当前类
@AutoConfigureBefore    //在加载配置的类之前就加载当前类
@AutoConfigureOrder        //指定顺序，参数的数值越小越先初始化
```
## 二、 开发自己的Spring-Boot-Starter
### Starter是一个集成接合器，需完成两件事，引入相关的jar和自动配置
**按照Spring Boot 规范，需要以下两件事**
* 1、starter.jar完成引入相关的jar
* 2、autoConfigure.jar完成自动配置
当然，如何将autoConfigure写在starter内，也是可以的。

### starter的命名则范
**Spring提供的starter:**
* spring-boot-starer-XXX-x.y.z.jar
* spring-boot-XXX-autoconfigure-x.y.z.jar

**第三方的starter:**
* XXX-spring-boot-starter-x.y.z.jar 
* XXX-spring-boot-autoconfigure-x.y.z.jar

**注意**：
第三方包和spring的starter包，路径稍不同，第三方包的项目我(如：XXX项目名)在前面，而spring包项目名，则在后面。

### 动手制作一个第三方的lib包的starter
* **1、准备第三方的jar**
创建一个maven工程spb-common

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
 <modelVersion>4.0.0</modelVersion>

 <groupId>org.study.spb</groupId>
 <artifactId>spb-common</artifactId>
 <version>1.0-SNAPSHOT</version>

</project>
```
![](index_files/6d5f0e71-3741-48c2-8bb1-ccf1f0eadce9.png)

创建一个服务类CommonService，可以简单一点，就一个方法，service()，然后两个属性name、desc和getter、setter方法。
``` java
package org.study.spb.common;
public class CommonService {
    private String name;
 private String desc;

 public String service(){
        return this.toString();
  }

    public String getName() {
        return name;
  }

    public void setName(String name) {
        this.name = name;
  }

    public String getDesc() {
        return desc;
  }

    public void setDesc(String desc) {
        this.desc = desc;
  }

    @Override
  public String toString() {
        return this.name+","+this.desc;
  }
}
```

部署并生成jar包到仓库，所以需要使用install命令
```
mvn clean install
```

* **2、制作starter**
* 1)、建工程
按第三方命名规范，创建spb-common-springboot-starter项目名

* 2)、引入spring-boot-start、spring-boot-autoconfigure、第三方jar

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
<modelVersion>4.0.0</modelVersion>
<groupId>org.study.spb</groupId>
<artifactId>spb-common-springboot-starter</artifactId>
<version>1.0-SNAPSHOT</version>

<parent>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-starter-parent</artifactId>
 <version>2.2.0.RELEASE</version>
 <relativePath/> <!-- lookup parent from repository -->
</parent>

<dependencies>
    
  <!--引入spb-common包-->
  <dependency>
 <groupId>org.study.spb</groupId>
 <artifactId>spb-common</artifactId>
 <version>1.0-SNAPSHOT</version>
 </dependency>
    
  <!-- 以下三个包，制作starter包必须要用的-->
 <!--spring-boot-starter基础包 -->
  <dependency>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-starter</artifactId>
 <version>2.2.0.RELEASE</version>
 </dependency>  

    <!--spring-boot-autoconfigure包，自动注入基础包 -->
  <dependency>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-autoconfigure</artifactId>
 <version>2.2.0.RELEASE</version>
 </dependency>  

    <!--pring-boot-configuration-processor包，自动生成元信息，可实现自动键入application.yml -->
  <dependency>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-configuration-processor</artifactId>
 <version>2.2.0.RELEASE</version>
 </dependency></dependencies>
</project>
```

* 3)、如需要生成配置元信息，加入spring-boot-configuration-processor依赖
如上图POM文件，已引入此包

* 4)、编写自动配置类

在org.study.spb目录下创建SpbCommonProperties配置参数类
```java
package org.study.spb;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@ConfigurationProperties(prefix = "spb.common")
public class SpbCommonProperties {
    /**
 * 名称
  */
  private String name;
  /**
 * 描述
  */
  private String desc;
  /**
 * 地址
  */
  private String url;

 public String getName() {
        return name;
  }

    public void setName(String name) {
        this.name = name;
  }

    public String getDesc() {
        return desc;
  }

    public void setDesc(String desc) {
        this.desc = desc;
  }

    public String getUrl() {
        return url;
  }

    public void setUrl(String url) {
        this.url = url;
  }
}
```

在org.study.spb目录下，创建SpbCommonAutoConfiguration配置类，用于自动装配CommonService类bean到spring 容器。
@EnableConfigurationProperties注解用于加载SpbCommonProperties配置参数类

```java
package org.study.spb;

import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.study.spb.common.CommonService;

@Configuration
@EnableConfigurationProperties({SpbCommonProperties.class})
public class SpbCommonAutoConfiguration {

    @Bean
  public CommonService getCommonService(SpbCommonProperties spbCommonProperties){
        CommonService commonService=new CommonService();
  commonService.setName(spbCommonProperties.getName());
  commonService.setName(spbCommonProperties.getDesc());
 return commonService;
  }
}
```

* 5)、配置发现配置文件：META-INF/spring.factories
在resources目录下创建META-INF目录，在META-INF下创建spring.factories文件

```xml
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.study.spb.SpbCommonAutoConfiguration
```

* 6)、打包发布
部署并生成jar包到仓库，所以需要使用install命令
```
mvn clean install
```

* 7)、测试调用情况
测试application的参数是否可用，如下图，能正常显示。还具备自动提示功能，因为引入了spring-boot-configuration-processor自动json元信息功能。
![1572762675657](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572762675657.png)
![1572762690061](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572762690061.png)
![1572762704468](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572762704468.png)
![1572762722155](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572762722155.png)

测试调用情况是否正确，创建测试类SpbCommonStarterTest，测试通过，能正常调用

![1572762735246](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572762735246.png)
![1572762745787](D:\doc\blog\images\study\springboot\1572762745787.png)

[示例代码|chap03|github](https://github.com/liyunhua258/Study/tree/master/springboot2-chap03)
[示例代码|spb-common-springboot-starter|github](https://github.com/liyunhua258/Study/tree/master/spb-common-springboot-starter)
[示例代码|spb-common|github](https://github.com/liyunhua258/Study/tree/master/spb-common)
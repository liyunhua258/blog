## SpringBoot解密第一篇
## 一、感受SpringBoot Starter
### 1、SpringBoot特性的优点有哪些？
引用官方：
Spring Boot makes it easy to create stand-alone, production-grade Spring based Applications that you can "just run".

* Create stand-alone Spring applications
* Embed Tomcat, Jetty or Undertow directly (no need to deploy WAR files)    
**使创建一个基于Spring的应用很简单**
* Provide opinionated 'starter' dependencies to simplify your build configuration    
**Starter简化依赖配置**
* Automatically configure Spring and 3rd party libraries whenever possible    
**自动配置**
* Provide production-ready features such as metrics, health checks and externalized configuration    
* Absolutely no code generation and no requirement for XML configuration    
**零XML配置**

### 2、对比：如果不使用SpringBoot，搭建一个SSM工程需要多久？
1、加入相关的jar包
2、配置web.xml，加载spring和是spring mvc
3、配置数据库连接、配置spring事务
4、配置加载配置文件的读取，开启注释
5、配置日志文件
...
配置完成之后部署tomcat调试

**需要步骤和配置很多，期间还会碰到很多问题，其中难点是什么？**
* 包依赖
* bean配置

这些是不是必需做的？SpringBoot中，这些是不是starter帮我们做了？

**解决问题的办法：Starter**
* starter引入相关的jar
* starter自动完成bean配置

## 二、SpringBoot Starter 如何自动添加依赖包的？
按步骤来解读spring-boot-starter-parent内部奥密。
**步骤一**
* 新SpringBoot项目的POM文件都继承了spring-boot-starter-parent，点击parent标签spring-boot-starter-parent，可打开spring-boot-starter-parent的POM文件
![1572760289553](D:\doc\blog\images\study\springboot\1572760289553.png)

**步骤二**
* 查看spring-boot-starter-parentPOM文件，可以得知，
springboot自动对properties配置文件，做了一些默认配置，如：JDK默1.8、编码默认UTF-8，同时自动加载resources目录的配置文件，如：application.yml、application.yaml、application.properties文件
![1572760312973](D:\doc\blog\images\study\springboot\1572760312973.png)
![1572760330343](D:\doc\blog\images\study\springboot\1572760330343.png)

**步骤三**
* 继续打开spring-boot-starter-parentPOM文件的parent标签的spring-boot-dependencies，可点击进入spring-boot-dependencies的POM文件。
从spring-boot-dependenciesPOM文件，我们读取到大量依赖包maven信息，而且依赖包固化了版本号，所以我们使用parent标签方法继承spring-boot-starter-parentPOM文件，就等于在项目中自动添加了图中所有依赖包。
![1572760369889](D:\doc\blog\images\study\springboot\1572760369889.png)
![1572760384873](D:\doc\blog\images\study\springboot\1572760384873.png)
![1572760398129](D:\doc\blog\images\study\springboot\1572760398129.png)
...
![1572760410982](D:\doc\blog\images\study\springboot\1572760410982.png)

## 三、SpringBoot Starter 如何自动处理依赖关系的？
以mybatis为例子，解读如何mybatis集成springboot，与datasource如何建立关系的？

**步骤一**
* 添加springboot对第三方mybatis的集成包，mybatis-spring-boot-starter
![1572760432848](D:\doc\blog\images\study\springboot\1572760432848.png)

**步骤二**
* 在maven依赖包中找到mybatis-spring-boot-starter架包，打开POM文件
![1572760445560](D:\doc\blog\images\study\springboot\1572760445560.png)

**步骤三**
* mybatis-spring-boot-starterPOM文件列出的依赖包，如下
```
// springboot
org.springframework.boot.spring-boot-starter
// springboot jdbc
org.springframework.boot.spring-boot-starter-jdbc
// springboot autoconfigure
org.mybatis.spring.boot.mybatis-spring-boot-autoconfigure
//mybatis原生包
org.mybatis.mybatis
//mybatis spring 集成包
org.mybatis.mybatis-spring
```
![1572760462643](D:\doc\blog\images\study\springboot\1572760462643.png)
![1572760472416](D:\doc\blog\images\study\springboot\1572760472416.png)

**步骤四**
* springboot能与mybatis实现自动依赖，关键就在mybatis-spring-boot-autoconfigure这个架包上，在maven依赖包找到它，如图所示，
![1572760487114](D:\doc\blog\images\study\springboot\1572760487114.png)

**步骤五** 
* 查看MybatisAutoConfiguration这个类，通过构造器实现对象注入，然后，我们重点看下此类上的注解。
![1572760499111](D:\doc\blog\images\study\springboot\1572760499111.png)

**步骤六** 
* 解读MybatisAutoConfiguration类上的注解
```
@ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})
@ConditionalOnBean({DataSource.class})
@EnableConfigurationProperties({MybatisProperties.class})
@AutoConfigureAfter({DataSourceAutoConfiguration.class})
```

@ConditionalOnClass 
表示SqlSessionFactory、SqlSessionFactoryBean这两个类都存在时，才加载这个配置
@ConditionalOnBean 表示DataSource这个bean对象在spring容器存在时，才加载这个类配置
@EnableConfigurationProperties 这个类自动装配时，把mybatis配置文件加载，并自动注入spring容器
@AutoConfigureAfter 等DataSourceAutoConfiguration自动注入后，才加载这个类配置

SqlSessionFactory和SqlSessionFactoryBean在spring容器，已经存在，关键要看@DataSourceAutoConfiguration是如何实现自动注入的。

**步骤七**
* 解读DataSourceAutoConfiguration类上的注解
![1572760517332](D:\doc\blog\images\study\springboot\1572760517332.png)

**步骤八**
* 打开import注释导入的DataSourcePoolMetadataProvidersConfiguration类，进一步解读
![1572760528885](D:\doc\blog\images\study\springboot\1572760528885.png)
![1572760542415](D:\doc\blog\images\study\springboot\1572760542415.png)
![1572760552884](D:\doc\blog\images\study\springboot\1572760552884.png)

从源码中，可以看出，dbcp2和tocat连接池，在项目类路径下都不存在，只有HikariDataSource可以。由此可见，springboot2采用HikariDataSource作为默认连接池。Hikari与dbcp2、tomcat、cp03、BoneCP等连接池比较，无论是速度、稳定、体积等指标上都完胜。

**总结：**
1、DataSourcePoolMetadataProvidersConfiguration类的内部类HikariPoolDataSourceMetadataProviderConfiguration配置类自动注入spring 容器
2、DataSourceAutoConfiguration配置类，满足DataSource接口类、EmbeddedDatabaseType枚举类存在，import外部导入 DataSourcePoolMetadataProvidersConfiguration类后，实现自动装配bean
3、MybatisAutoConfiguration配置类，满足SqlSessionFactory、SqlSessionFactoryBean类存在，DataSourceAutoConfiguration自动装配后，也实现自动装配bean

## 四、SpringBoot Starter 的bean需要的参数，是如何规定并获取的？
还是以mybatis为例子，控索mybatis配置文件和datasource配置文件，如何自动注入的。

* 1、application.yml中mybatis.mapper-locations，mybatis.config-location与MybatisProperties类中的属性相对应，其它属性也全都对应
![1572760566607](D:\doc\blog\images\study\springboot\1572760566607.png)
![1572760579258](D:\doc\blog\images\study\springboot\1572760579258.png)
![1572760589692](D:\doc\blog\images\study\springboot\1572760589692.png)

* 2、spring.datasource.url，spring.datasource.username，spring.datasource.password与DataSourceProperties类中的属性相对应，其它属性也全都对应
![1572760601708](D:\doc\blog\images\study\springboot\1572760601708.png)
![1572760614390](D:\doc\blog\images\study\springboot\1572760614390.png)
![1572760625426](D:\doc\blog\images\study\springboot\1572760625426.png)

## 五、SpringBoot Starter 的bean是如何被发现并自动装配的？

### 1、解读入口类@SpringBootApplication注解
@SpringBootApplication 是SpringBoot 的核心注解，它是一个组合注解，源码如下：
![1572760639003](D:\doc\blog\images\study\springboot\1572760639003.png)
@SpringBootApplication 注解主要组合了@SpringBootConfiguration、@EnableAutoConfiguration、@ComponentScan、@ConfigurationPropertiesScan四个注解。
@SpringBootConfiguration其实就是特殊的@Configuration注解，
@ComponentScan、@ConfigurationPropertiesScan为类文件扫描和配置参数文件自动扫描，那么@EnableAutoConfiguration是干什么的？

### 2、解读@EnableAutoConfiguration注解
**步骤一**
请在maven依赖包中找到spring-boot-autoconfigure包META-INF目录下的spring.factories文件打开，看到很多类似配置文件的键值结构。
找到#Auto Configure注释下，org.springframework.boot.autoconfigure.EnableAutoConfiguration自动注入配置类，信息类太多，下面我们以mybatis为例子。
![1572760653270](D:\doc\blog\images\study\springboot\1572760653270.png)
![1572760664005](D:\doc\blog\images\study\springboot\1572760664005.png)

**步骤二**
在maven依赖包中找到mybatis-spring-boot-autoconfigure包META-INF目录下的spring.factories文件打开
![1572760676917](D:\doc\blog\images\study\springboot\1572760676917.png)
![1572760687012](D:\doc\blog\images\study\springboot\1572760687012.png)
![1572760698598](D:\doc\blog\images\study\springboot\1572760698598.png)

**注意：**
spring.factories中的键值结构，值中除了有逗号分隔符外，还有/，/其实就是一个换行符,因为一行太长不好展示。

### 总结：
@SpringBootApplication组合注解类中包含 @EnableAutoConfiguration注解类。@EnableAutoConfiguration则是扫描项目中所有META-INF下面的spring.factories文件，如果该文件有org.springframework.boot.autoconfigure.EnableAutoConfiguration这个类的配置，则会将键值结构的值(值为扫描类的逗号分隔的拼装)，加入扫描中。是不是感觉so easy?
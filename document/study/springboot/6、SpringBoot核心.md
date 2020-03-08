# 一、 基础配置
## 1、入口类和@SpringBootApplication
@SpringBootApplication 是SpringBoot 的核心注解，它是一个组合注解，源码如下：
![1572763355207](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763355207.png)
@SpringBootApplication 注解主要组合了@Configuration、@EnableAutoConfiguration、@ComponentScan三个注解，@EnableAutoConfiguration让SpringBoot根据类路径中的jar包依赖为前项目自动配置，自动扫描@SpringBootApplication所在类的同级包以及下级包里的Bean，建议入口类放置在groupId+arctifactID组合的包名下。

## 2、关闭特定的自动配置
使用@SpringBootApplication注解的exclude参数，eg:
`@SpringBootApplication(exclude={DataSourceAutoConfiguration.class})`

## 3、定制Banner
SpringBoot启动时会有一个默认的启动图案，如图
![1572763366252](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763366252.png)
1、修改Banner
(1)、在src/main/resources新建一个banner.txt
(2)、通过`http://patorjk.com/software/taag` 网站生成字符，如敲入的为"WISELY"，将网站生成的字符复制到banner.txt中。
2、关闭banner
(1)、main里的内容修改为
`
SpringApplication app=new SpringApplication(Ch522Application.class);
app.setShowBanner(false);
app.run(args);
`
(2)或者使用fluent API修改为
`
new SpringApplicationBuilder(Ch522Application.class)
.showBanner(false)
.run(args)
`

# 4、SpringBoot的配置文件
SpringBoot 使用一个全局的配置文件application.properties或application.yml，在resource目录或类路径的config下。
1、修改全局配置，tomcat端口号，8080改为9090，并将默诉访问路径/改为/helloboot
，可以这样改。
application.properties
`server.port=9090
server.servlet.context-path=/helloboot
`
application.yml
`
server:
    port:9090
    contextPath:/helloboot
`

# 5、Starter pom
1、官方Spring Boot，详见书面
2、第三方Starter pom，详见书面

# 6、使用Xml配置
通过Spring提供的@ImportResource来加载xml配置
`
@ImportResource({"classpath:some-context.xml","classpath:another-context.xml"})`

# 二、外部配置
## 1、命令行参数配置
springboot可基于jar命令运行，如
`java -jar xx.jar`
修改tomcat端口号，命令执行
`java -jar xx.jar --server.port=9090`

## 2、常规属性配置
在springboot里，我们只需在在application.properties定义属性，直接使用@value注入即可。
(1)、application.properties增加属性
`book.author=liynhua
book.name=spring boot
`
(2)、修改入口类
![](index_files/b6bb5616-4995-43ae-89f9-8cf2024b1c22.png)

## 3、类型安全配置
(1)、添加配置， 即在 application.properties上添加
`author.name=liyunhua`
`author.age=34`

类型安全的Bean，代码如下：
![1572763407335](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763407335.png)

检验代码：
![1572763426742](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763426742.png)

(2)、单独定义properties文件，添加配置,比如新建一个author.properties文件，内容还是author.name和author.age，通过 @ConfigurationProperties如何加载文件：

![1572763441053](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763441053.png)
# 三、日志配置
SpringBoot支持Java Util Logging,Log4J,Log4J2和Logback作为日志框架，无论使用哪种日志框架，Spring Boot 已为当前使用日志框架的控制台输出及文件输出做好的配置。
默认情况下，Spring Boot使用Logback作为日志框架
配置日志级别：
`logging.file=D:/mylog/log.log`
配置日志文件，格式为logging.level.包名=级别：
`logging.level.org.springframework.web=DEBUG`
# 四、Profile配置
Profile是spring用来针对不同的环境对不同的配置提供支持的，全局Profile配置使用application-{profile}.properties，通过在application.properties中设置spring.profile.active=prod来指定活动的profile。
![1572763454262](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763454262.png)
![1572763467821](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763467821.png)
![1572763477178](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763477178.png)


# 五、Spring的运行原理
## 1、运作原理
研究下@SpringBootApplication注解，这是一个组合注解，看源码：
![1572763487754](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763487754.png)
关键功能是@Import注解导入的配置功能，EnableAutoConfigurationImportSelector使用SpringFctoriesLoadFactoryNames方法扫描具有 META-INF/spring.factories文件的jar包，而我们的spring-boot-autoconfigure-2.0.4.RELEASE.jar就有。配置如下：
![1572763500080](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763500080.png)

## 2、核心注解
 在org.springframework.boot.autoconfigure.condition包下，条件注解如下：
![1572763518119](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763518119.png)

 ![1572763529988](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763529988.png)



这些注解都是组合了@Condetional元注解，只是使用了不同的条件(Condition)，分析下@ConditionalOnWebApplication注解

![](index_files/30e5bb64-e8a1-4b43-b247-9b8ecae5bbae.jpg)

`从源码中可以看出，注解的实现类为OnWebApplicationCondition.class类，该类继承了SpringBootCondition，实现方法getMatchOutcome，核心方法为isWebApplication，具体详解请参与源码。

## 3、实例分析
![1572763541661](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763541661.png)

![1572763552408](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763552408.png)

![1572763566777](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763566777.png)
![1572763576571](https://liyunhua.oss-cn-hangzhou.aliyuncs.com/blog/images/study/springboot/1572763576571.png)

## 4、
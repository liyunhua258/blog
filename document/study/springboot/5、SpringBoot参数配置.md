## Spring-Boot 数配置规则详解
### 参数配置文件的约定名称为:
> application.properties / application.yml

### 参数配置文件的约定放置位置（优先级从高到低）为:
>    1 运行程序的当前工作目录下的config子目录file:./config
>    2 运行程序的当前工作目录file:./
>    3 classpath:/config 子目录
>    4 classpath:/ 根目录

### ConfigFileApplicationListener
SpringBoot项目，通过ConfigFileApplicationListener类，完成对参数配置文件的加载。
![1572763252364](D:\doc\blog\images\study\springboot\1572763252364.png)

### 指定spring.config.name、spring.config.location属性值的三种方式：
> 操作系统环境变量：spring.config.name=xxxx 
> JVM参数：-Dspring.config.name=xxxx 
> 程序命令行参数：--spring.config.name=xxxx    

--用来说明不是一个普通程序参数，而是一个配置参数

```shell
java -jar myproject.jar --spring.config.nane=myproject
java -jar myproject.jar --spring.config.location=
classpath:/default.properties,classpath:/override.properties
```

### spring.config.location 属性特别说明
> spring.config.location指定配置文件所在目录（以/结尾）或文件名，可多值（逗号间隔）
> 查找的顺序是配置的反序

### spring.config.name 属性特别说明
> spring.config.name可多值（逗号间隔）
> 属性查找的顺序是配置的反序

### spring.config.additional-location 属性说明
>用来指定补充的配置文件目录或配置文件
>优先級高于spring.config. location

### 除了可在参数配置文件中配置外，还有其他方式配置参数吗?
![1572763269072](D:\doc\blog\images\study\springboot\1572763269072.png)
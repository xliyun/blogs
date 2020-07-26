# 

* 各种日志技术的关系和作用
* 通过源码来分析spring的日志技术
* commons-logging源码分析
* 通过源码分析mybaits的日志技术
* 架构系统时候如何选择、优化日志技术

## 主流的log技术名词

1. log4j
```
<!--<dependency>-->
    <!--<groupId>log4j</groupId>-->
    <!--<artifactId>log4j</artifactId>-->
    <!--<version>1.2.12</version>-->
<!--</dependency>-->
```
可以不需要依赖第三方的技术
直接记录日志

2. jcl

jakartaCommonsLoggingImpl

```
<dependency>
    <groupId>commons-logging</groupId>
    <artifactId>commons-logging</artifactId>
    <version>1.1.1</version>
</dependency>
```
jcl他不直接记录日志,他是通过第三方记录日志(jul)
如果使用jcl来记录日志,在没有log4j的依赖情况下,是用jul

如果有了log4j则使用log4j

```
jcl=Jakarta commons-logging ,是apache公司开发的一个抽象日志通用框架,本身不实现日志记录,但是提供了记录日志的抽象方法即接口(info,debug,error.......),底层通过一个数组存放具体的日志框架的类名,然后循环数组依次去匹配这些类名是否在app中被依赖了,如果找到被依赖的则直接使用,所以他有先后顺序。
```
![图片](https://images-cdn.shimo.im/yFBaGU5euKMuCqtu/jcl1.png!thumbnail)

```
上图为jcl中存放日志技术类名的数组，默认有四个，后面两个可以忽略。
```
![图片](https://images-cdn.shimo.im/WUYYvMBBkx4eL28s/jcl0.png!thumbnail)

```
上图81行就是通过一个类名去load一个class，如果load成功则直接new出来并且返回使用。
如果没有load到class这循环第二个，直到找到为止。
```
![图片](https://images-cdn.shimo.im/8MzEr8n4FycGtto5/jcl2.png!thumbnail)

```
可以看到这里的循环条件必须满足result不为空，
也就是如果没有找到具体的日志依赖则继续循环，如果找到则条件不成立，不进行循环了。
总结：顺序log4j>jul
```
3. jul

java自带的一个日志记录的技术,直接使用

4. log4j2
5. **slf4j**

**slf4j他也不记录日志,通过绑定器绑定一个具体的日志记录来完成日志记录**

```
log4j---<artifactId>slf4j-log4j12</artifactId>
```
6. logback
7. simple-log
## 各种日志技术的关系和作用

![图片](https://images-cdn.shimo.im/HCYt068pb1YY5GZZ/log.png!thumbnail)

## spring日志技术分析

### spring5.*日志技术实现

spring5使用的spring的jcl(spring改了jcl的代码)来记录日志的,但是jcl不能直接记录日志,采用循环优先的原则，在LogFactory中默认JUC，如果依赖中有log4j2，就先用，如果我们想要用log4j1，就先绑定slf4j，然后使用slf4j绑定log4j1

```xml
<!-- slf4j绑定log4j，自带slf4j -->
<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.7.28</version>
</dependency>
<dependency>
  <groupId>log4j</groupId>
  <artifactId>log4j</artifactId>
  <version>1.2.17</version>
</dependency>
```
----
```java
public abstract class LogFactory {
   private static LogApi logApi = LogApi.JUL;
   static {
      ClassLoader cl = LogFactory.class.getClassLoader();
      try {
         // Try Log4j 2.x API
         cl.loadClass("org.apache.logging.log4j.spi.ExtendedLogger");
         logApi = LogApi.LOG4J;
      }
      catch (ClassNotFoundException ex1) {
         try {
            // Try SLF4J 1.7 SPI
            cl.loadClass("org.slf4j.spi.LocationAwareLogger");
            logApi = LogApi.SLF4J_LAL;
         }
         catch (ClassNotFoundException ex2) {
            try {
               // Try SLF4J 1.7 API
               cl.loadClass("org.slf4j.Logger");
               logApi = LogApi.SLF4J;
            }
            catch (ClassNotFoundException ex3) {
               // Keep java.util.logging as default
            }
         }
      }
   }
```
###  spring4.*日志技术实现

spring4当中依赖的是原生的jcl

## mybatis的日志技术实现

### 初始化

**org.apache.ibatis.logging.LogFactory**

```
tryImplementation(new Runnable() {
  @Override
  public void run() {
    useSlf4jLogging();
  }
});
```
关键代码  **   if (logConstructor == null)****,**没有找到实现则继续找
```
private static void tryImplementation(Runnable runnable) {
  if (logConstructor == null) {
    try {
      runnable.run();
    } catch (Throwable t) {
      // ignore
    }
  }
}
```
![图片](https://images-cdn.shimo.im/pNtdoqBmmD0mUXLE/my0.png!thumbnail)![图片](https://images-cdn.shimo.im/5bPdBJeaxZ0vZT3J/my1.png!thumbnail)

### 具体实现类

mybatis提供很多日志的实现类,用来记录日志,取决于初始化的时候load到的class

![图片](https://images-cdn.shimo.im/26jOIrgomZIDKJ5f/my2.png!thumbnail)

```
上图红色箭头可以看到JakartaCommonsLoggingImpl
中引用了jcl 的类,如果在初始化的时候load到类为JakartaCommonsLoggingImpl
那么则使用jcl 去实现日志记录,但是也是顺序的,顺序参考源码
```
### 自己模拟实现mybaits的日志实现

```
mybatis的官网关于日志的介绍
定义org.apache.ibatis.session.Configuration
参考org.apache.ibatis.logging.commons.JakartaCommonsLoggingImpl
分析为什么jcl不记录日志,修改代码
```
## mybatis和spring5整合后日期问题

spring5 + mybatis + log4j === mybatis没有日志

mybatis + log4j  === mybatis有日志

为什么呢？

mybatis+log4j，mybatis的LogFactory会在初始化的时候，按照顺序找slf4j,jcl,log4j2,log4j

所以入如果有log4j的依赖，就会有log4j日志

spring5 + mybatis + log4j 因为spring5默认是jcl日志，mybatis在执行下面的代码的时候就会采用jcl，而且spring5内部默认使用jul，jul默认只打印info以上的级别，而mybatis打印sql是info以下的，会判断日志logisDebugEnabled()，是不是DEBUGGER级别，jul的日志级别改不了。如果是spring4的jcl会去默认使用log4j，log4j是可以配置日志级别

```plain
private static Constructor<? extends Log> logConstructor;
static {
  tryImplementation(new Runnable() {
    @Override
    public void run() {
      useSlf4jLogging();
    }
  });
  tryImplementation(new Runnable() {
    @Override
    public void run() {
      useCommonsLogging();
    }
  });
  tryImplementation(new Runnable() {
    @Override
    public void run() {
      useLog4J2Logging();
    }
  });
  tryImplementation(new Runnable() {
    @Override
    public void run() {
      useLog4JLogging();
    }
  });
  tryImplementation(new Runnable() {
    @Override
    public void run() {
      useJdkLogging();
    }
  });
  tryImplementation(new Runnable() {
    @Override
    public void run() {
      useNoLogging();
    }
  });
}
```
## 架构系统如何考虑日志

old:jcl+log4j

new:slf4j+jul


---




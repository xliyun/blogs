# what is IOC

```
控制反转（Inversion of Control，缩写为IoC），是面向对象编程中的一种设计原则，可以用来减低计算机代码之间的耦合度。其中最常见的方式叫做依赖注入（Dependency Injection，简称DI），还有一种方式叫“依赖查找”（Dependency Lookup）
```
## Dependency Injection

依赖注入

关于什么是依赖

关于注入和查找以及拖拽

# 为什么要使用spring IOC

spring体系结构----IOC的位置  自己看官网

```
在日常程序开发过程当中，我们推荐面向抽象编程，面向抽象编程会产生类的依赖，当然如果你够强大可以自己写一个管理的容器，但是既然spring以及实现了，并且spring如此优秀，我们仅仅需要学习spring框架便可。
当我们有了一个管理对象的容器之后，类的产生过程也交给了容器，至于我们自己的app则可以不需要去关系这些对象的产生了。
```
# spring实现IOC的思路和方法

```
spring实现IOC的思路是提供一些配置信息用来描述类之间的依赖关系，然后由容器去解析这些配置信息，继而维护好对象之间的依赖关系，前提是对象之间的依赖关系必须在类中定义好，比如A.class中有一个B.class的属性，那么我们可以理解为A依赖了B。既然我们在类中已经定义了他们之间的依赖关系那么为什么还需要在配置文件中去描述和定义呢？
spring实现IOC的思路大致可以拆分成3点
```
1. 应用程序中提供类，提供依赖关系（属性或者构造方法）
2. 把需要交给容器管理的对象通过配置信息告诉容器（xml、annotation，javaconfig）
3. 把各个类之间的依赖关系通过配置信息告诉容器
```
配置这些信息的方法有三种分别是xml，annotation和javaconfig
维护的过程称为自动注入，自动注入的方法有两种构造方法和setter
自动注入的值可以是对象，数组，map，list和常量比如字符串整形等
```
# spring编程的风格

初始化spring环境有两种方法

xml                        ClassPathXmlApplicationContext    类的扫描  单独bean的注册（在xml里面写了bean就同时完成声明和注册）

javaconfig              AnnotationConfigApplicationContext 类的扫描 类的定义 (注册需要扫描)

annotationConfigApplicationContext.rigister()

annotation     需要xml方式或者javaconfig方式帮忙开启注解

所谓的初始化spring环境--->把我们交给spring管理的类实例化

```
schemal-based-------xml
annotation-based-----annotation
java-based----java Configuration
```
# 
# 

注入的两种方法

```
spring注入详细配置（字符串、数组等）参考文档：
https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-properties-detailed
```
## Constructor-based Dependency Injection

```
构造方法注入参考文档：
https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-constructor-injection
```
![图片](https://images-cdn.shimo.im/No88OKjKqfQIA1rz/image.png!thumbnail)


## Setter-based Dependency Injection

```
setter参考文档：
https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-setter-injection
```
![图片](https://images-cdn.shimo.im/WClSBbHC63UiHqpU/image.png!thumbnail)

# 自动装配

上面说过，IOC的注入有两个地方需要提供依赖关系，一是类的定义中，二是在spring的配置中需要去描述。自动装配则把第二个取消了，即我们仅仅需要在类中提供依赖，继而把对象交给容器管理即可完成注入。

在实际开发中，描述类之间的依赖关系通常是大篇幅的，如果使用自动装配则省去了很多配置，并且如果对象的依赖发生更新我们可以不需要去更新配置，但是也带来了一定的缺点

自动装配的优点参考文档：

[https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-autowire](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-autowire)

缺点参考文档：

[https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-autowired-exceptions](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-autowired-exceptions)

作为我来讲，我觉得以上缺点都不是缺点

## 自动装配的方法

```
no
byName
byType
constructor
自动装配的方式参考文档：
https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-autowire
```
![图片](https://images-cdn.shimo.im/qyZgYw9KH7Iap0vt/image.png!thumbnail)

## xml的自动装配

### byName

xml中配置

```
<!--default-autowire="byName"  从indexSerice中找set+bean名 dao就找setDao dao2就找setDao2 -->
<bean id="dao" class="com.xliyun.spring.autowire.IndexDaoImpl">
</bean>

<bean id="dao2" class="com.xliyun.spring.autowire.IndexDaoImpl2">
</bean>

<bean id="indexService" class="com.xliyun.spring.autowire.IndexService">

</bean>
```
indexServie中只要有setDao2和xml文件中的dao2对应
```
public class IndexService {
    private IndexDao dao;
    public IndexService(){
   }
    public void service(){
        dao.test();
    }

    public void setDao2(IndexDao dao) {
        this.dao = dao;
    }
}
```
### byType

```
<!--default-autowire="byType"  如果有两个类型相同的bean，就会报错，spring不知道应该自动装配哪个  -->
<bean id="dao" class="com.xliyun.spring.autowire.IndexDaoImpl">
</bean>

<bean id="dao2" class="com.xliyun.spring.autowire.IndexDaoImpl2">
</bean>

<!-- autowire是可以单独指定的 -->
<bean id="indexService" class="com.xliyun.spring.autowire.IndexService">
</bean>
```
### 单独指派

在indexService上加autowire="constructor"

```
<!--default-autowire="byType"  如果有两个类型相同的bean，就会报错，spring不知道应该自动装配哪个  -->
<!--default-autowire="byName"  从indexSerice中找set+bean名 dao就找setDao dao2就找setDao2 -->
<bean id="dao" class="com.xliyun.spring.autowire.IndexDaoImpl">
</bean>

<bean id="dao2" class="com.xliyun.spring.autowire.IndexDaoImpl2">
</bean>

<!-- autowire是可以单独指定的 -->
<bean id="indexService" class="com.xliyun.spring.autowire.IndexService" autowire="constructor">

</bean>
```
## 注解的自动装配

spring提供了@Repository, @Service，@Controller， @Component 可以注解到任意被spring管理的类上面，@Component是上面那些的父类，虽然现在他们没有什么差异，后面sping可能会用这些注解做一些差异化。

### @Autowired

注解里面用@Autowired是默认使用byType方式的，如果一个接口有多个实现，spring在自动装载类的时候会报错。

byType没找到的话会根据属性名去找，比如有IndexDaoImpl和IndexDaoImpl2两个实现，一开始会根据类型去找，有两个实现，再按照属性名去找IndexDaoImpl

### @Resource

1. @Resource默认是根据属性名字去匹配，比如有IndexDaoImpl和IndexDaoImpl2两个实现，spring就会根据属性名去加载IndexDaoImpl
2. 当然@Resource也可以直接指定类型
3. 也可以指定name，比如IndexDaoImpl的beanName就是首字符变为小写的indexDaoImpl
```
@Service
public class IndexService {

    //@Resource(type = IndexDaoImpl.class)
    @Resource(name = "indexDaoImpl")
    private IndexDao indexDaoImpl;


    public IndexService(){

    }

    public IndexService(IndexDao dao){
        this.indexDaoImpl = dao;
    }

    public void service(){
        indexDaoImpl.test();
    }

    public void setDaoxxx(IndexDao dao) {
        this.indexDaoImpl = dao;
    }
}
```
## spring bean名称重写

# Using depends-on 

```
<bean id="beanOne" class="ExampleBean" depends-on="manager"/>
<bean id="manager" class="ManagerBean" />
```
# spring懒加载

官网已经解释的非常清楚了：

用xml或者@lazy注解

```
https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-lazy-init
<bean id="lazy" class="com.something.ExpensiveToCreateBean" lazy-init="true"/>
<bean name="not.lazy" class="com.something.AnotherBean"/>
```
```
值得提醒的是，如果你想为所有的对都实现懒加载可以使用官网的配置
```
# ![图片](https://images-cdn.shimo.im/AL7NwUqEre0woKxB/image.png!thumbnail)

# springbean的作用域

```
文档参考：
https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-scopes
```
![图片](https://images-cdn.shimo.im/z7bJcesJ7cMJeMvM/image.png!thumbnail)

xml定义方式

```
<bean id="accountService" class="com.something.DefaultAccountService" scope="singleton"/>
```
annotation的定义方式
```
@Service
@Scope("prototype")
public class IndexService {

    //@Resource(type = IndexDaoImpl.class)
    @Resource(name = "indexDaoImpl")
    private IndexDao indexDaoImpl;
}
```
## Singleton Beans with Prototype-bean Dependencies

一个单例的service依赖了一个原型的bean，这个原型的bean就失去了意义，spring官网称这个为method-injection

```
意思是在Singleton 当中引用了一个Prototype的bean的时候引发的问题
官网引导我们参考https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-method-injection
```
## method-injection问题的解决方法

### 在类中拿到application对象，自行实例化bean

```
@Service("indexService2")
public class IndexService2  implements ApplicationContextAware {

    @Resource(name = "indexDaoImpl")
    private IndexDao indexDaoImpl;

    private ApplicationContext applicationContext;

    public void service(){
        IndexDao dao= (IndexDao) applicationContext.getBean("indexDaoImpl");
        System.out.println("service的hashCode:"+this.hashCode());
        System.out.println("indexDao的hashCode:"+dao.hashCode());
    }

    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext=applicationContext;
    }
}
```
### Loop Up Method方法解决

```
<bean id="commandManager" class="fiona.apple.CommandManager">
    <lookup-method name="createCommand" bean="myCommand"/>
</bean>
```
### Loop Up注解

网上参考的方法

```
@Service("indexService3")
public class IndexService3 {
    @Lookup(value = "indexDaoImpl")
    public IndexDao getIndexDaoImpl(){
        return null;
    }

    @Resource
    private IndexDao indexDaoImpl;


    public void service(){
        System.out.println("service的hashCode: "+this.hashCode());
        System.out.println("indexDao的hashCode: "+getIndexDaoImpl().hashCode());
    }
    
}
```
官方注解方法
```
@Service("indexService4")
public abstract class IndexService4 {

    @Lookup(value = "indexDaoImpl")
    protected abstract IndexDao getIndexDaoImpl();

    public void service(){
        System.out.println("service的hashCode: "+this.hashCode());
        System.out.println("indexDao的hashCode: "+getIndexDaoImpl().hashCode());
    }

}
```
# spring声明周期和回调

```
参考文档：
https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-lifecycle
1、Methods annotated with @PostConstruct
2、afterPropertiesSet() as defined by the InitializingBean callback interface
3、A custom configured init() method
```
## spring初始化和销毁接口

```
@Repository
public class IndexDaoImpl1 implements IndexDao, InitializingBean, DisposableBean {
    public IndexDaoImpl1(){
        System.out.println("Constructor");
    }
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("初始化bean");
    }
    @Override
    public void destroy() throws Exception {
        System.out.println("销毁bean");
    }
}
```
## 在xml文件中配置默认初始化函数

在xml文件中配置初始化函数的名字，然后就可以在bean中使用

```
<beans default-init-method="init">
    <bean id="blogService" class="com.something.DefaultBlogService" init-method="initMethodName">
        <property name="blogDao" ref="blogDao" />
    </bean>
</beans>
```
## 使用注解

```
@Repository
public class IndexDaoImpl1 implements IndexDao {

    public IndexDaoImpl1(){
        System.out.println("Constructor");
    }
    
    @PostConstruct
    public void initMethod(){
        System.out.println("使用注解 ==> 初始化bean时调用");
    }

    @PreDestroy
    public void destroyMethod(){
        System.out.println("使用注解 ==> 销毁bean时调用");
    }
}
```
# 扫描包的排除

```
@ComponentScan(value = "com.liyun",excludeFilters  = {@ComponentScanFilter(type = FilterType.REGEX,pattern = "com.liyun.ceshi.*")}）

//REGEX是正则表达式
@ComponentScan(basePackages = "org.example",
        includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),//需要被排除的某个包里的某个类
        excludeFilters = @Filter(Repository.class))
```
**basePackages**：Spring将扫描的基础package名，Spring会扫描该包以及其子孙包下的所有类
 

**useDefaultFilters**：默认为true，此时Spring扫描类时发现如果其被标注为 @Component、@Repository、@Service、@Controller则自动实例化为bean并将其添加到上下文中，如果设置为false，即使将其标注为@Component或者其他，Spring都会忽略

 

**includeFilters**： 指定扫描时需要实例化的类型，我们可以从名字看到这是一个Filter，你可以自己定义该Filter，Spring为我们提供了一套方便的实现，我们可以根据标注、类、包等相关信息决定当扫描到该类时是否需要实例化该类，需要注意的是如果你仅仅想扫描如@Controller不仅要加includeFilters，还需要将useDefaultFilters设置为false

 

**excludeFilter**：指定扫描到某个类时需要忽略它，实现和上一个Filter一样，区别只是如果Filter匹配，Spring会忽略该类

# 重复了的类型的处理

1. @Primary：在众多相同的bean中，优先选择用@Primary注解的bean（该注解加在各个bean上）  
2. @Qualifier：在众多相同的bean中，@Qualifier指定需要注入的bean（该注解跟随在@Autowired后）  

 

@Profile环境切换

@Bean加在方法上，方法返回的对象会加载到spring容器当中

[https://blog.csdn.net/ysl19910806/article/details/91646554](https://blog.csdn.net/ysl19910806/article/details/91646554)

# spring互相引用

scope = single 的模式由于先 new bean采用缓存的原因，不会有这个问题，scope = prototype 原型模式在使用的时候才会生成bean，就会存在互相引用的问题

[https://blog.csdn.net/w1lgy/article/details/81086171](https://blog.csdn.net/w1lgy/article/details/81086171)

# @profile注解的使用

可以加在javaconfig配置文件上，也可以加在bean上

```
public static void main(String[] args) {
    AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext();

    //选择profile的环境
    annotationConfigApplicationContext.getEnvironment().setActiveProfiles("dao1");
    annotationConfigApplicationContext.register(Spring.class);
    //重新扫描一遍
    annotationConfigApplicationContext.refresh();

   String indexDaoName = annotationConfigApplicationContext.getBean(IndexDao.class).getClass().getSimpleName();
    System.out.println(indexDaoName);

}
```
# beanFacotry和factoryBean

### beanFacotry

```plain
//beanfactory就是产生bean的工厂
BeanFactory beanFactory = new ClassPathXmlApplicationContext("classpath:spring.xml");
```
### factoryBean

一些第三方的类，比如Mybatis，外部依赖很复杂，我们的mybatis生效的时候需要一个SqlSessionFactory的类，但是Mybatis作为第三方，我们在xml中不好配置SqlSessionFactory依赖的类，所以让mybatis自己配置好这个类，然后继承factoryBean接口实现一个SqlSessionFactoryBean，在这个类中把一些需要我们动手配置的依赖注入进去。

注意factoryBean不需要注解

### 被factoryBean代理的类

```plain
public class TempDaoFacotryBean {
    private String msg1;
    private String msg2;
    private String msg3;
    public void test(){
        System.out.println("Facotry Bean ");
    }
    public String getMsg1() {
        return msg1;
    }
    public void setMsg1(String msg1) {
        this.msg1 = msg1;
    }
    public String getMsg2() {
        return msg2;
    }
    public void setMsg2(String msg2) {
        this.msg2 = msg2;
    }
    public String getMsg3() {
        return msg3;
    }
    public void setMsg3(String msg3) {
        this.msg3 = msg3;
    }
}
```
factoryBean类
```plain
/**
 * 如果你的类实现了FactoryBean
 * 那么spring容器当中存在两个对象
 * 一个叫getObject()返回的对象
 * 还有一个是当前对象
 *
 * getObject得到的对象存的是当前类指定的名字
 * 当前对象是 "&"+当前类的名字
 */
@Component("DaoFactoryBean")
public class DaoFacotryBean implements FactoryBean {
    String msg;
    public String getMsg() {
        return msg;
    }
    public void setMsg(String msg) {
        this.msg = msg;
    }
    public void testBean(){
        System.out.println("DaoFacotryBean");
    }
    @Override
    public Object getObject() throws Exception {
        TempDaoFacotryBean tempDaoFacotryBean = new TempDaoFacotryBean();
        String[] arr = msg.split(",");
        tempDaoFacotryBean.setMsg1(arr[0]);
        tempDaoFacotryBean.setMsg2(arr[1]);
        tempDaoFacotryBean.setMsg3(arr[2]);
        return tempDaoFacotryBean;
    }
    @Override
    public Class<?> getObjectType() {
        return TempDaoFacotryBean.class;
    }
    @Override
    public boolean isSingleton() {
        return true;
    }
}
```
调用
```plain
public static void main(String[] args) {
    //beanfactory就是产生bean的工厂
    //BeanFactory beanFactory = new ClassPathXmlApplicationContext("classpath:spring.xml");
    AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext(SpringConfig.class);
    TempDaoFacotryBean tempDaoFacotryBean = (TempDaoFacotryBean) annotationConfigApplicationContext.getBean("DaoFactoryBean");
    tempDaoFacotryBean.test();
    DaoFacotryBean daoFacotryBean = (DaoFacotryBean) annotationConfigApplicationContext.getBean("&DaoFactoryBean");
    daoFacotryBean.testBean();
}
```
# 导入bd的方法

1.register()注册一个 （需要一个类）

2.被@Scan扫描到的 （需要一个类）

3.ImportBeanDefinitionRegistrar （需要一个类）


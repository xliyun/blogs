# Mybatis源码分析

## mybatis的流程分析

<u>！！如果不想看mybatis源码部分，只想看区别，**直接去看 mybatis和spring+mybatis在调用过程中的区别** 标题</u>

**首先mybatis的源码分为两种情况：**

1. 单独的mybatis
2. 和spring整合的mybatis的源码

这两种情况下的源码分析有点不同，比如如果是分析mybatis+spring的模式那么mybatis的开始是从spring初始化的时候开始的。

**这两种方式最后调用的都是实现了sqlSession接口的类的selectMap (查询多条时执行的方法)之类的接口**

```java
result = sqlSession.<K, V>selectMap(command.getName(), param, method.getMapKey(), rowBounds);
```

## 单独的mybatis

默认情况下sqlSessionFactory.openSession()返回的是DefaultSqlSession类的对象

```java
        //原生mybatis
        String resource = "mybatis-config.xml";
        InputStream inputStream = null;
        try {
            inputStream = Resources.getResourceAsStream(resource);
        } catch (IOException e) {
            e.printStackTrace();
        }
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        SqlSession sqlSession = sqlSessionFactory.openSession();//这里返回的是DefaultSqlSession
        sqlSession.getConfiguration().addMapper(CityMapper.class);
        CityMapper cityMapper2 = sqlSession.getMapper(CityMapper.class);
        cityMapper2.list();
```

### sqlSession.getMapper方法获取代理对象

CityMapper cityMapper2 = sqlSession.getMapper(CityMapper.class);

DefaultSqlSession的getMapper方法

```java
  @Override
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }
```

Configuration的getMapper方法

```java
 public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```

mapperRegistry的getMapper方法

```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
        //在这里
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

最后返回一个mapper的动态代理MapperProxy自己本身就实现了InvocationHandler

```java
  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```

### 执行mapper代理对象的方法

cityMapper2.list();执行的是MapperProxy.invoke()方法

```java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
      //这里的sqlSession是DefaultSqlSession
    return mapperMethod.execute(sqlSession, args);
  }
```

mapperMethod.execute，我的示例走result = executeForMany(sqlSession, args);

```java
  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
      Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
            //我的示例走这条
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

执行

```java
  private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    Object param = method.convertArgsToSqlCommandParam(args);
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      result = sqlSession.<E>selectList(command.getName(), param, rowBounds);
    } else {
        //
      result = sqlSession.<E>selectList(command.getName(), param);
    }
    // issue #510 Collections & arrays support
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
      if (method.getReturnType().isArray()) {
        return convertToArray(result);
      } else {
        return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
      }
    }
    return result;
  }
```

最后执行sqlSession的selectList

```java
  @Override
  public <E> List<E> selectList(String statement, Object parameter) {
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
  }

  @Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

## 和spring整合的mybatis的源码

扫描出来的mapper都被MapperFactoryBean通过sqlSessionTemplate变成MapperProxy这种代理类

当从spring中获取Bean时，获取到的就是这种代理类

当我们执行查询方法的时候，代理类invoke代码中通过

```java
         //和spring整合的mybatis
        AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        CityMapper cityMapper = annotationConfigApplicationContext.getBean(CityMapper.class);
        System.out.println(cityMapper.list());
```

### spring初始化时，扫描出来的mapper都被封装到MapperFactoryBean当中

 **以下是AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext(AppConfig.class);代码执行过程**

@MapperScan注解通过@Import(MapperScannerRegistrar.class)，将所有的mapper扫描出来，扫描出来的mapper都存放在MapperFactoryBean当中，然后通过MapperFactoryBean返回MapperProxy代理对象（**只不过是和sqlSessionTemplate绑定，由sqlSessionTemplate增强**），当我们调用CityMapper cityMapper = annotationConfigApplicationContext.getBean(CityMapper.class);时，**和单独的mybatis一样，返回的就是这个代理对象**

**MapperFactoryBean的getObject()方法**

getObject()就相当于 CityMapper cityMapper2 = sqlSession.getMapper(CityMapper.class);

getSqlSession获取到的是SqlSessionTemplate

```java
  @Override
  public T getObject() throws Exception {
      //getSqlSession()返回的是SqlSessionTemplate
      //mapperInterface就是我示例用的CityMapper
    return getSqlSession().getMapper(this.mapperInterface);
  }
```

SqlSessioniTemplate的getMapper()方法

```java
  @Override
  public <T> T getMapper(Class<T> type) {
    return getConfiguration().getMapper(type, this);
  }
```

Configuration的getMapper()方法

```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```

**在MapperRegistry的getMapper()方法中完成代理，返回MapperProxy**

```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

### **Mapper代理类和调用方法时执行的代码**

mapperInterface就是我们的CityMapper，这里就是要为我们的mapper实现一个代理类

jdk动态代理需要的InvocationHandler由MapperProxy实现

jdk动态代理的代理类在实现方法的时候，是调用InvocationHandler的invoke方法，也就是说，

```java
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```

我们调用CityMapper的list()方法最后是执行Invoke方法中的代码，mapperMethod当中记录着要执行方法的信息

```java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        //判断对象是不是当前对象
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
      //这里传递的sqlSession是SqlSessionTemplate
    return mapperMethod.execute(sqlSession, args);
  }
```

MapperMethod当中的execute方法

```java
  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
      Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
            //我的示例是走这条语句
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

MapperMethod查询多条数据executeForMany()

```java
  private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    Object param = method.convertArgsToSqlCommandParam(args);
      //判断是否select后是否有字段绑定，如果是select * 走下面的else
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      result = sqlSession.<E>selectList(command.getName(), param, rowBounds);
    } else {
        //SqlSessionTemplate
      result = sqlSession.<E>selectList(command.getName(), param);
    }
    // issue #510 Collections & arrays support
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
      if (method.getReturnType().isArray()) {
        return convertToArray(result);
      } else {
        return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
      }
    }
    return result;
  }
```

SqlSessionTemplate的selectList()方法，sqlSessionProxy是 代理对象，代理DefaultSqlSession,这个代理对象的InvocationHandler的实现类是SqlSessionTemplate的内部类， selectList最后的实现还是要看SqlSessionTemplate的invoke方法

```java
   private final SqlSession sqlSessionProxy;
  @Override
  public <E> List<E> selectList(String statement, Object parameter) {
    return this.sqlSessionProxy.<E> selectList(statement, parameter);
  }
```

SqlSessionTemplate中的内部类，这里的重点是，执行完成之后closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);所以spring5和mybatis整合之后，以及缓存会失效

```java
  private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
```

## mybatis和spring+mybatis在调用过程中的区别

**mybatis:**

sqlSession==>defaultSqlSession==>defaultSqlSession.selectlist==>sql

**springmybats:**

sqlSession==>sqlSessionTemplate==>sqlSessionTemplate==>proxy.invoke==>selectlist==>sql

**最终的区别在于：**

mybatis通过mapper代理类执行的是DefaultSqlSession的selectList()方法

```java
  @Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

spring+mybatis通过mapper代理类执行的是SqlSessionTemplate的selectList()方法，调用的是SqlSessionTemplate的代理类的selectList方法

```
  @Override
  public <E> List<E> selectList(String statement, Object parameter) {
    return this.sqlSessionProxy.<E> selectList(statement, parameter);
  }
```

最后这个代理类的invoke是在SqlSessionTemplate的子类中实现

```java
  private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
            //关闭了一级缓存
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
```



## 首先考虑spring + mybatis的时候他们的关联点有哪些？

1. @MapperScan
2. @Bean SqlSessionFactoryBean

利用的就是spring当中的Import和ImportBeanDefinitionRegistrar技术来对spring进行了扩展，在对比一下@BeanSqlSessionFactoryBean就会知道spring会首先去执行ImportBeanDefinitionRegistrar当中的registerBeanDefinition方法。



## mybatis的一级缓存的各种问题

### spring当中为什么失效

因为mybatis和spring的集成包当中扩展了一个类SqlSessionTemplate，这个类在spring容器启动的时候被注入给了mapper这个类替代了原来的DefaultSqlSession,SqlSessionTemplate当中的所有查询方法不是直接查询，而是经过一个代理对象，代理对象增强了查询的方法，主要是关闭了session



spring把SqlSession做了一个代理，这个代理当中放了一个SqlSessionFactory，每次执行完毕的时候代理对象把SqlSession给关了。

### mybatis的一级缓存技术的底层原理



## spring+mybatis为什么要关闭一级缓存

因为mybatis的SqlSession类暴露给了我们，我们可以通过sqlSession直接关闭，而spring+mybatis的SqlSessionTemplate的接口并没有暴露给我们。

一级缓存是在我们的线程当中的。


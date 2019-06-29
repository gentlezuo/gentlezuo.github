---
title: Mybaits源码解析-2-SQL执行
date: 2019-05-11 12:33:59
tags: mybatis
category: mybatis
---

# SQL执行过程

<!--more-->
一个例子

```java
public void querytOne() throws IOException {
    SqlSession sqlSession = sqlSessionFactory.openSession();
    try {
        //1
        ManageMapper mapper = sqlSession.getMapper(ManageMapper.class);
        //2
        Manage manage = mapper.selectById(1234567890);
    }finally {
        sqlSession.close();
    }
}

```


sql的执行如下：先根据反射获取一个Mapper对象，代理传入的类；然后根据sql的类型，比如select，返回的是map还是list还是void，选择不同的执行方法，解析参数的信息，判断是否有可以从缓存中获取数据等等，最后将查询的值经过结果处理器处理，将查询的数据放入缓存中，返回。


从1开始，获得一个Manage的代理类:根据配置文件以及sqlSession以及之前的运行状态获取
```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    //获得一个工厂对象
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
        //3获取代理对象
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```
在3处
```java
public T newInstance(SqlSession sqlSession) {
    //MapperProxy是一个实现了InvocationHandler接口的类
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    //4
    return newInstance(mapperProxy);
  }
```

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache;

  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }

  @Override
  //拦截的实际逻辑
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
    //在调用之前才会new一个MapperMethod，并且将其保存在缓存中
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }

  private MapperMethod cachedMapperMethod(Method method) {
      //从缓存中拿，如果没有就new一个，并将其放入map中
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }

```

4 使用jdk反射new Instance
```java
protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
```
此时已经拿到了一个Mapper，Mapper中包含：  
![](/Mybaits源码解析-2-SQL执行/Mapper1.png) 

在2处，接下来执行调用的方法，由于代理，与是调用`invoke`方法   
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        //如果是Object或者默认的方法，直接放行，不进行其他操作
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    //否则开始检查是否有MapperMethod对象的缓存，如果有直接获取，否则创建一个。
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    //5接下来开始执行sql语句
    return mapperMethod.execute(sqlSession, args);
  }
```

先了解一下MapperMethod类   
```java
public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, mapperInterface, method);
  }
```
SqlCommand包含了关于这条sql的信息:  
```java
public static class SqlCommand {

    //sql语句的name
    private final String name;
    //sql语句的type，是一个枚举类型，包括：UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH;
    private final SqlCommandType type;
    //省略
```
MethodSignature包含了关于这个方法的详细详细：
```java
public static class MethodSignature {
    //返回的结果集的类型：many，map，空，游标等等
    private final boolean returnsMany;
    private final boolean returnsMap;
    private final boolean returnsVoid;
    private final boolean returnsCursor;
    //返回的类的类型
    private final Class<?> returnType;
    private final String mapKey;
    private final Integer resultHandlerIndex;
    private final Integer rowBoundsIndex;
    //参数节气器
    private final ParamNameResolver paramNameResolver;
    //省略其他
```
在5处 execute方法：   
```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    //判断sql的类型，比如如果是isnert，就使用sqlSession的insert方法
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
      //在select中还需要判断返回的什么
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
            //解析传入的参数
          Object param = method.convertArgsToSqlCommandParam(args);
          //调用selectOne，实际上是调用selectList取第一个
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

```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    //根据很多条件计算出缓存的key，然后判断是否可以重缓存中那值而不经过数据库
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}

```

此时可以看见Mapper对象更新了   
![](/Mybaits源码解析-2-SQL执行/Mapper对象.png)


最后sqlSession.close();会处理是否提交，是否回滚，是否将一级缓存的值放入二级缓存中等等。




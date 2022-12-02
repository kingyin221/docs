## 模块化Key

EasyCache 框架的基本用法在「快速入门」章节中已经说明，这里主要介绍缓存服务的一些特性。

### 缓存Key的构成

在EasyCache 中，Key主要由三个部分构成，分别是「模块」「方法名」「参数」，这三部分共同组成缓存的唯一键。

若没有指定模块则默认以 `ky-cache`作为模块名，若没有指定方法名（一般不指定）则默认取当前方法名，参数部分是构成Key的重点，可用是一个或多个对象，但是参数过多会导致缓存Key过大，在实际应用建议中指定5个基本类型参数参与构建Key。

例如：

推荐的使用方式，当然，methodName属性可用不指定，这里仅演示

```java
    @EasyCache(module = "user", methodName = "listUser", params = {
            @Param(name = "id", ref = "query.id"),
            @Param(name = "projectId", ref = "query.projectId")
    })
    public List<User> listUsers(UserQuery query) {
        ...
    }
```

错误的使用方式，在这种情况下产生的Key不能确定长度，极其不推荐！！

```java
    @EasyCache(module = "user", methodName = "listUser")
    public List<User> listUsers(UserQuery query) {
        ...
    }
```

外部使用，Module注解和MethodName可以控制Key的模块名和方法名，优先级高于在EasyCache中使用

```java
    @Module("ky-user")
		@MethodName("getUser")
    @EasyCache(module = "ky-test",methodName = "getOne")
    public User userTest(int id, @Ignore String username) {
        System.out.println("无返回值方法："+id+" "+username);
    }
```

注：这里的最终模块名和方法名分别是`ky-user`和`getUser`，参数仅有`id`一个，`username`忽略

### 自定义Key

EasyCache提供了自定义Key格式的服务

第一步：创建Key继承`MethodEasyCacheKey`

```java
public class KingyinKey extends MethodEasyCacheKey {
    @Override
    public EasyCacheKey parse(String sources) {
        return super.parse(sources);
    }

    @Override
    public String coalescence() {
        return super.coalescence();
    }
}
```

第二步：使用Key

```java
cache.setCacheMethodKeyAdapter(new CacheMethodKeyAdapter() {
    @Override
    public EasyCacheKey parse(CacheMethod sources) {
        return new KingyinKey.Builder().build(sources);
    }
});
```

Lambda形式

```java
cache.setCacheMethodKeyAdapter(sources -> new KingyinKey.Builder().build(sources));
```



## 缓存引擎

EasyCache 支持自定义缓存引擎，并且系统默认实现了三种引擎。

| 类型            | 名称         |
| --------------- | ------------ |
| InnerCache      | 内置引擎     |
| RedisCache      | 远程引擎     |
| ~~Level2Cache~~ | 二级缓存引擎 |

注：二级缓存已废弃，将在未来版本中重构

### 内置缓存引擎-Inner

内置引擎跟随应用一同启动，持久化周期为10分钟，实现单JVM读写锁，可在单机部署中避免缓存穿透。

### 引擎配置

默认引擎设置，存储路径为`/resources/easycache.rdb`

```java
// inner 引擎（默认）
AnnotationEasyCache cache = new AnnotationEasyCache();
```

自定义持久化路径

```java
// inner 引擎（指定持久化路径）
AnnotationEasyCache cache = new AnnotationEasyCache("/db.rdb",Serializer.GSON);
```

自定义Inner引擎

```java
InnerCache engine = new InnerCache.Builder()
    .map() // 自定义存储
    .serializer() // 自定义序列化
    .load() // rdb加载路径
    .persist() // 持久化策略
    .addRemoveListener() // 删除监听器
    .addSlowListener() // 慢日志监听器
    .build();
AnnotationEasyCache cache = new AnnotationEasyCache(engine,Serializer.GSON);
```

### 远程缓存引擎-Redis

框架天然适配Redis引擎，可配置Jedis实现存储引擎替换，远程缓存引擎实现多JVM读写锁（即分布式锁），在微服务中避免缓存穿透。

自定义Redis引擎

```java
JedisPoolConfig config = new JedisPoolConfig();
config.setMinEvictableIdleTime(Duration.ofMillis(60000));
config.setTimeBetweenEvictionRuns(Duration.ofMillis(30000));
config.setMaxWait(Duration.ofMillis(100000));
RedisCache engine = new RedisCache.Builder()
  .host("127.0.0.1").port(6379)
  .config(config)
  .build();
AnnotationEasyCache cache = new AnnotationEasyCache(engine,Serializer.GSON);
```

### 自定义引擎

创建类继承`CacheAdapter`并实现抽象方法即可完成自定义缓存引擎

## 异步删除

EasyCache 支持以异步的方式删除本地缓存或远程缓存，权衡利弊之下异步删除对于大部分系统而言是比较有益的，只要不涉及时效性要求极高的系统那都可以使用EasyCache框架。

## 异常转换

EasyCache 提供了异常转换过滤器，默认实现了以下三种异常过滤器

| 类型                       | 名称           | 描述                                                         |
| -------------------------- | -------------- | ------------------------------------------------------------ |
| IgnoreExceptionHandler     | 忽略异常过滤器 | 当异常发生时并不做任何处理，坚持写入缓存（异常存入缓存，以应对恶意攻击造成的缓存击穿） |
| PrudentAllExceptionHandler | 谨慎异常过滤器 | 当异常发生时对所有异常谨慎，都不写入缓存，在开发测试时可以使用 |
| NetworkExceptionFilter     | 网络异常过滤器 | 当且仅当发生网络异常时，不写入缓存，其他异常都写入缓存（可以有效的避免网络波动） |

注：除了系统实现的异常过滤器，还支持自定义异常过滤器并以注解的方式加入缓存流程，其主要实现来源于`AnnotationInvokeExceptionHandler`

异常转换需要配合过滤器和结果执行器共同完成，过滤器链路负责分析筛选异常，对选中的异常进行结果转换，默认情况下的结果转换是`OriginalResultConverter` （返回原对象）即不做处理

### 异常过滤

方式一：在EasyCache注解中定义

```java
    @EasyCache(exception = {
            @InvokeException(filter = NetworkExceptionFilter.class)
    })
    public User getOne() throws SocketTimeoutException {
        throw new SocketTimeoutException();
    }
```

方式二：直接在方法上指定

```java
    @InvokeException(filter = NetworkExceptionFilter.class)
    public User getOne() throws SocketTimeoutException {
        throw new SocketTimeoutException();
    }
```

注：二者的区别在于前者可以定义多个过滤器，而后置只能定义一个过滤器，**后者的优先级高于前者**！

### 结果转换

该特殊由InvokeException的属性result决定

使用方式

第一步：创建自定义结果转换器

```java
public class EmptyUser implements ResultConverter {
    // 返回空对象
    @Override
    public Object conversion(Object object) {
        return new User();
    }
}
```

第二步：在InvokeException使用

```java
@InvokeException(filter = NetworkExceptionFilter.class, result = EmptyUser.class)
public User getOne() throws SocketTimeoutException {
    throw new SocketTimeoutException();
}
```

注：异常过滤器在实际使用中需要自定义，并且确保只使用一个异常过滤器！

运行结果

> 返回 User(username=null, age=null, signature=null)

## 动态过期策略

EasyCache 默认提供了动态过期策略由处理器`AnnotationExpireAutoSetPostProcess`实现，默认以72小时为基础过期时间，动态因子控制在[0.7~1]之间，有效避免缓存雪崩

```java
// 基本过期时间24小时
// 动态因子0.5（过期时间控制在12～24小时内）
@EasyCache(expire = @Expire(time = 24, unit = TimeUnit.HOURS, factor = 0.5))
public User getOne(String username) {
    return dao.get(username);
}
```


## 多维模块化Key

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

## 缓存引擎

EasyCache 支持自定义缓存引擎，并且系统默认实现了三种引擎。

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
RedisCache redis = new RedisCache.Builder()
  .host("127.0.0.1").port(6379)
  .config(config)
  .build();
AnnotationEasyCache cache = new AnnotationEasyCache(engine,Serializer.GSON);
```

### 自定义引擎

创建类继承`CacheAdapter`并实现抽象方法即可完成自定义缓存引擎

注：二级缓存已废弃

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


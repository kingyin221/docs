## 配置参数

EasyCache 提供了引擎的基本配置，在Maven项目中需要手动设置配置项，而在Boot项目中由YAML文件配置，具体配置如下

### engine配置

分别介绍Maven和Boot两种引擎配置方式

Maven项目配置

默认提供GSON序列化

```java
// JSON序列化
Serializer.GSON
```

```java
// 适配引擎，序列化
AnnotationEasyCache(CacheAdapter cacheAdapter, ValueSerializer serializer);
// 路径，序列化
AnnotationEasyCache(String path, ValueSerializer serializer)
```

Boot项目配置

serializer：gson | ~~json~~

> 推荐使用gson，gson能够保证序列化的正确性而json则不能

module: inner | redis | ~~level2~~

> 默认inner，为了系统的稳定不建议使用level2存储引擎

```yaml
easycache:
  engine:
    serializer: gson
    module: inner
```

### Inner配置

分别介绍Maven和Boot两种引擎配置方式

Maven项目配置

默认实现以`ConcurrentHashMap`实现，最大存储值为`Integer.MAX_VALUE`，默认存储路径为`/resources/easycache.rdb`

```java
InnerCache engine = new InnerCache.Builder()
    .map() // 自定义存储
    .size() // 存储大小
    .serializer() // 自定义序列化
    .load() // rdb加载路径
    .persist() // 持久化策略
    .addRemoveListener() // 删除监听器
    .addSlowListener() // 慢日志监听器
    .build();
```

Boot项目配置

path：路径

size：存储大小

```yaml
  inner:
    path: /..../cache.rdb
    size: -1
```

### redis配置

分别介绍Maven和Boot两种引擎配置方式

Maven项目配置

关于更多连接池设置可以参考`JedisPoolConfig` https://blog.csdn.net/u011271894/article/details/120787584

```java
JedisPoolConfig config = new JedisPoolConfig();
// MinEvictableIdleTime
config.setMinEvictableIdleTime(Duration.ofMillis(60000));
// TimeBetweenEvictionRuns
config.setTimeBetweenEvictionRuns(Duration.ofMillis(30000));
// MaxWait
config.setMaxWait(Duration.ofMillis(100000));
RedisCache engine = new RedisCache.Builder()
  // redis连接地址
  .host("127.0.0.1")
  // redis连接端口
  .port(6379)
  .config(config)
  .build();
```

Boot项目配置

host：连接地址，默认127.0.0.1

port：端口，默认6379

```yaml
  redis:
    host: 127.0.0.1
    port: 6379
    password: xxxxxxx
    minEvictableIdleTime: 60000
    timeBetweenEvictionRuns: 30000
    maxWait: 100000
    maxIdle: 100
    minIdle: 10
```


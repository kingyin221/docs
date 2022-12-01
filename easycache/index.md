# EasyCache 方式快速入门

本文档将演示如何应用 EasyCache 在Java项目进行缓存增强。 本例将在本地Maven项目中创建测试类并完成缓存查询和更新操作。

## 创建工程

需要安装 JDK 8 及以上 和 Maven 3 以上

新建一个Maven工程，并引入EasyCache依赖

```xml
<dependency>
    <groupId>club.kingyin</groupId>
    <artifactId>easycache</artifactId>
    <version>1.0.3</version>
</dependency>
```

注：最新版本可用从 https://gitee.com/kingyinOS/easycache/releases 里找到

## 缓存入门

第一步： 编写业务类

```java
/**
 * 模拟简单业务类
 */
public class UserService {
    private static final Map<String, User> dao = new HashMap<>();

    // 构造三条模拟数据
    static {
        dao.put("username1", new User("username1", 18, "这个家伙很懒"));
        dao.put("username2", new User("username2", 19, "这个家伙很懒"));
        dao.put("username3", new User("username3", 20, "这个家伙很懒"));
    }

    @EasyCache
    public User getOne(String username) {
        return dao.get(username);
    }

}
```

注：在getOne方法上使用了@EasyCache注解，其作用是将getOne方法中的参数username当作缓存Key，将方法返回值User当作数据存储在存储引擎中。

第二步：获取代理类

```java
/**
 * 测试类
 */
public class Application {

    public static void main(String[] args) {
        // 创建基于注解实现的EasyCache
        AnnotationEasyCache cache = new AnnotationEasyCache();

        // 模拟业务类
        UserService userService = new UserService();

        // 代理增强
        userService = (UserService) cache.proxy(userService);

        System.out.println("第一次调用：" + userService.getOne("username1"));

        System.out.println("第二次调用：" + userService.getOne("username1"));

    }
}
```

## 运行结果

启动Application后控制台打印如下...截取此过程部分日志

```bash
2022-12-01 21:56:24.282 [main] DEBUG c.k.e.c.h.MinRandomTimerPostProcess - 【getOne】 随机过期时间 69，单位 DAYS，最小系数 0.7
2022-12-01 21:56:24.305 [main] DEBUG c.k.easycache.method.MethodEasyCache - 缓存 [ky-cache]-[getOne()]-{"username":"username1"}-[]，源数据加载，并写入缓存 User(username=username1, age=18, signature=这个家伙很懒)
2022-12-01 21:56:24.306 [main] DEBUG c.k.e.p.a.AbstractEasyCacheMethod - 缓存方法执行 [getOne]
2022-12-01 21:56:24.311 [main] DEBUG c.k.e.c.a.handler.TimerWithExecution - 【执行耗时-正常】：72 ms
第一次调用：User(username=username1, age=18, signature=这个家伙很懒)

2022-12-01 21:56:24.313 [main] DEBUG c.k.easycache.method.MethodEasyCache - 缓存 [ky-cache]-[getOne()]-{"username":"username1"}-[]，存在 User(username=username1, age=18, signature=这个家伙很懒)
2022-12-01 21:56:24.313 [main] DEBUG c.k.e.p.a.AbstractEasyCacheMethod - 缓存方法执行 [getOne]
2022-12-01 21:56:24.313 [main] DEBUG c.k.e.c.a.handler.TimerWithExecution - 【执行耗时-正常】：2 ms
第二次调用：User(username=username1, age=18, signature=这个家伙很懒)
```

## 缓存删除

第一步：关联删除注解，在UserService中添加更新方法

```java
@EasyCache(removes = {
  @Remove(methodName = "getOne", prams = {
    @Param(name = "username",ref = "user.username")
  })
})
public void updateByUsername(User user) {
  dao.put(user.getUsername(), user);
}
```

注：Remove注解可以实现删除指定方法产生的缓存，此处即删除getOne方法中username和updateByUsername方法中user.username字段相同的缓存。

第二步：删除缓存

```java
System.out.println("第一次调用：" + userService.getOne("username1"));

System.out.println("更新数据：" + userService.updateByUsername(User.builder().username("username1").age(20).build()));

// 异步删除，等待一秒钟后获取
TimeUnit.SECONDS.sleep(1);
System.out.println("第二次调用：" + userService.getOne("username1"));
```

## 运行结果

启动Application后控制台打印如下...截取此过程部分日志

```bash
2022-12-01 22:23:46.858 [main] DEBUG c.k.e.c.h.MinRandomTimerPostProcess - 【getOne】 随机过期时间 68，单位 DAYS，最小系数 0.7
2022-12-01 22:23:46.883 [main] DEBUG c.k.easycache.method.MethodEasyCache - 缓存 [ky-cache]-[getOne()]-{"username":"username1"}-[]，源数据加载，并写入缓存 User(username=username1, age=18, signature=这个家伙很懒)
2022-12-01 22:23:46.884 [main] DEBUG c.k.e.p.a.AbstractEasyCacheMethod - 缓存方法执行 [getOne]
2022-12-01 22:23:46.887 [main] DEBUG c.k.e.c.a.handler.TimerWithExecution - 【执行耗时-正常】：73 ms
第一次调用：User(username=username1, age=18, signature=这个家伙很懒)

2022-12-01 22:23:46.906 [main] DEBUG c.k.e.p.a.AbstractEasyCacheMethod - 原始方法执行 [updateByUsername]
2022-12-01 22:23:46.910 [main] DEBUG c.k.e.c.a.handler.TimerWithExecution - 【执行耗时-正常】：4 ms
更新数据：true
2022-12-01 22:23:46.912 [ky-pool-0] INFO  c.k.e.key.AbstractEasyCacheKey - 解析生成默认key [ky-cache]-[getOne()]-{}-[]
2022-12-01 22:23:46.913 [ky-pool-0] DEBUG c.k.easycache.cache.AbstractCache - 删除缓存 [ky-cache]-[getOne()]-{"username":"username1"}-[]

2022-12-01 22:23:47.919 [main] DEBUG c.k.e.c.h.MinRandomTimerPostProcess - 【getOne】 随机过期时间 68，单位 DAYS，最小系数 0.7
2022-12-01 22:23:47.919 [main] DEBUG c.k.easycache.method.MethodEasyCache - 缓存 [ky-cache]-[getOne()]-{"username":"username1"}-[]，源数据加载，并写入缓存 User(username=username1, age=20, signature=null)
2022-12-01 22:23:47.919 [main] DEBUG c.k.e.p.a.AbstractEasyCacheMethod - 缓存方法执行 [getOne]
2022-12-01 22:23:47.919 [main] DEBUG c.k.e.c.a.handler.TimerWithExecution - 【执行耗时-正常】：2 ms
第二次调用：User(username=username1, age=20, signature=null)
```

注：EasyCache异步删除缓存，所以在上面的案例中更新缓存后等待一秒钟后继续调用getOne方法。

更多示例参考高级用法

---

# SpringBoot 方式快速入门

本文档将演示如何应用 EasyCache 在SpringBoot项目进行缓存增强。 本例将在本地SpringBoot项目中创建测试类并完成缓存查询和更新操作。

## 创建工程

需要安装 JDK 8 及以上

新建一个SpringBoot工程，并引入EasyCache Starter依赖

```xml
<dependency>
		<groupId>club.kingyin</groupId>
		<artifactId>easycache-spring-boot-starter</artifactId>
		<version>1.1.2</version>
</dependency>
```

注：最新版本可用从 https://gitee.com/kingyinOS/easycache-spring-boot-starter/releases 里找到

## 缓存入门

注：SpringBoot使用方式和EasyCache使用方式完全相同，无需任何改动，在包含@Component源注解的类方法上直接使用注解即可！

步骤：业务类增强

```java
/**
 * 模拟简单业务类
 */
@Service
public class UserService {
    private static final Map<String, User> dao = new HashMap<>();

    // 构造三条模拟数据
    static {
        dao.put("username1", new User("username1", 18, "这个家伙很懒"));
        dao.put("username2", new User("username2", 19, "这个家伙很懒"));
        dao.put("username3", new User("username3", 20, "这个家伙很懒"));
    }

    @EasyCache
    public User getOne(String username) {
        return dao.get(username);
    }

    @EasyCache(removes = {
            @Remove(methodName = "getOne", prams = {
                    @Param(name = "username",ref = "user.username")
            })
    })
    public Boolean updateByUsername(User user) {
        dao.put(user.getUsername(), user);
        return true;
    }

}
```

更多示例参考高级用法

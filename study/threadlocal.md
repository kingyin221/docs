## JDK

jdk自带的ThreadLocal不支持父子线程之间的值传递。

## Alibaba

阿里巴巴开源项目**[transmittable-thread-local](https://github.com/alibaba/transmittable-thread-local)**解决了在父子线程或是线程池中值传递的问题

### Maven依赖

```java
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.13.2</version>
</dependency>
```

可以在 [search.maven.org](https://search.maven.org/artifact/com.alibaba/transmittable-thread-local) 查看可用的版本。

```java
transmittable-thread-local
```

### 快速入门

在普通线程池中可以通过`TtlRunnable`，`TtlCallable`装饰原有的任务，即可实现线程池的传值增强。

```java
TransmittableThreadLocal<String> context = new TransmittableThreadLocal<>();

// =====================================================

// 在父线程中设置
context.set("value-set-in-parent");

Callable call = new CallableTask();
// 额外的处理，生成修饰了的对象ttlCallable
Callable ttlCallable = TtlCallable.get(call);
executorService.submit(ttlCallable);

// =====================================================

// Call中可以读取，值是"value-set-in-parent"
String value = context.get();
```

### Demo

```java
public class RainstormContext {

    /**
     * alibaba开源TTL工具，支持父子线程值传递，支持线程池值传递
     * 注意：如果是普通线程池需要对任务进行装饰（TtlRunnable.get(task)｜TtlCallable.get(task)）
     */
    private final static TransmittableThreadLocal<RainstormSdkContext> CONTEXT = new TransmittableThreadLocal<>();

    /**
     * 获取context，若async参数设置为true则context的修改会同步到所有父子线程，
     * async设置为false则context仅在当前线程修改有效不会同步到其他线程
     * async：true->返回context引用
     * async：false->返回context深拷贝对象（context需要实现序列化接口）
     * @param async context是否线程同步
     * @return 当前父线程或者当前线程的context
     */
    public RainstormSdkContext getContext(boolean async) {
        RainstormSdkContext sdkContext = CONTEXT.get();
        log.info("获取RAINSTORM_CONTEXT 线程同步：{}",async);
        if (sdkContext == null) {
            log.warn("RAINSTORM_CONTEXT 获取为空");
        }
        return async?sdkContext:deepCopy(sdkContext);
    }

    /**
     * 默认获取context方法
     * @return context引用对象
     */
    public RainstormSdkContext getContext() {
        return getContext(true);
    }

    private static RainstormSdkContext deepCopy(RainstormSdkContext context) {
        if (context == null)
            return null;
        try {
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(bos);
            oos.writeObject(context);
            ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
            ObjectInputStream ois = new ObjectInputStream(bis);
            return (RainstormSdkContext) ois.readObject();
        } catch (Exception e) {
            log.warn("RAINSTORM_CONTEXT 深拷贝失败 {}",context);
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 刷新context，需要传入新的context，暂时不支持局部刷新
     * 原则上不允许在业务中刷新context！
     * @param context 新context对象
     */
    protected void setContext(RainstormSdkContext context) {
        log.info("刷新 RAINSTORM_CONTEXT：{}->{}", getContext(),context);
        CONTEXT.set(context);
    }
    /**
     * 刷新context，需要传入无参数的函数（函数返回一个新的context），暂时不支持局部刷新
     * 原则上不允许在业务中刷新context！
     * @param fn 无参函数
     */
    protected void setContext(Function<Void,RainstormSdkContext> fn) {
        setContext(fn.apply(null));
    }


    public static void main(String[] args) {
        RainstormContext rainstormContext = new RainstormContext();
        RainstormSdkContext sdkContext = new RainstormSdkContext();
        sdkContext.setLang("中文");
        rainstormContext.setContext(sdkContext);
        DefaultEventLoopGroup executors = new DefaultEventLoopGroup();
        rainstormContext.setContext(c -> {
            RainstormSdkContext sdkContext1 = new RainstormSdkContext();
            sdkContext1.setLang("猪皮");
            return sdkContext1;
        });

        log.debug("语言 {}", rainstormContext.getContext().getLang());
        RainstormSdkContext context = rainstormContext.getContext(false);
        new Thread(()->{
            log.debug("语言 {}", rainstormContext.getContext().getLang());
            sdkContext.setLang("中文");
        }).start();
        context.setLang("日语");
        log.debug("语言 {}", rainstormContext.getContext().getLang());
        executors.execute(TtlRunnable.get(()->{
            log.debug("语言 {}", rainstormContext.getContext().getLang());
            new DefaultEventLoopGroup().execute(()->{
                new Thread(()-> log.debug("语言 {}", rainstormContext.getContext().getLang())).start();
                log.debug("语言 {}", rainstormContext.getContext().getLang());
            });
        }));
        log.debug("语言 {}", rainstormContext.getContext().getLang());
    }
}

```

执行结果

```bash
club.kingyin.test.context.impl.RainstormContext
17:02:13.126 [main] INFO club.kingyin.test.context.impl.RainstormContext - 获取RAINSTORM_CONTEXT 线程同步：true
17:02:13.129 [main] WARN club.kingyin.test.context.impl.RainstormContext - RAINSTORM_CONTEXT 获取为空
17:02:13.129 [main] INFO club.kingyin.test.context.impl.RainstormContext - 刷新 RAINSTORM_CONTEXT：null->RainstormSdkContext(sdkContext=null, appHighwayContext=null, appAtopContext=null, uid=null, lang=中文, domain=null, projectId=null, clientId=null)
17:02:13.156 [main] DEBUG io.netty.util.internal.logging.InternalLoggerFactory - Using SLF4J as the default logging framework
17:02:13.157 [main] DEBUG io.netty.channel.MultithreadEventLoopGroup - -Dio.netty.eventLoopThreads: 16
17:02:13.168 [main] DEBUG io.netty.util.internal.InternalThreadLocalMap - -Dio.netty.threadLocalMap.stringBuilder.initialSize: 1024
17:02:13.168 [main] DEBUG io.netty.util.internal.InternalThreadLocalMap - -Dio.netty.threadLocalMap.stringBuilder.maxSize: 4096
17:02:13.173 [main] INFO club.kingyin.test.context.impl.RainstormContext - 获取RAINSTORM_CONTEXT 线程同步：true
17:02:13.173 [main] INFO club.kingyin.test.context.impl.RainstormContext - 获取RAINSTORM_CONTEXT 线程同步：true
17:02:13.173 [main] INFO club.kingyin.test.context.impl.RainstormContext - 刷新 RAINSTORM_CONTEXT：RainstormSdkContext(sdkContext=null, appHighwayContext=null, appAtopContext=null, uid=null, lang=中文, domain=null, projectId=null, clientId=null)->RainstormSdkContext(sdkContext=null, appHighwayContext=null, appAtopContext=null, uid=null, lang=猪皮, domain=null, projectId=null, clientId=null)
17:02:13.173 [main] INFO club.kingyin.test.context.impl.RainstormContext - 获取RAINSTORM_CONTEXT 线程同步：true
17:02:13.173 [main] DEBUG club.kingyin.test.context.impl.RainstormContext - 语言 猪皮
17:02:13.173 [main] INFO club.kingyin.test.context.impl.RainstormContext - 获取RAINSTORM_CONTEXT 线程同步：false
17:02:13.184 [main] INFO club.kingyin.test.context.impl.RainstormContext - 获取RAINSTORM_CONTEXT 线程同步：true
17:02:13.184 [main] DEBUG club.kingyin.test.context.impl.RainstormContext - 语言 猪皮
17:02:13.184 [Thread-0] INFO club.kingyin.test.context.impl.RainstormContext - 获取RAINSTORM_CONTEXT 线程同步：true
17:02:13.185 [Thread-0] DEBUG club.kingyin.test.context.impl.RainstormContext - 语言 猪皮
17:02:13.187 [main] INFO club.kingyin.test.context.impl.RainstormContext - 获取RAINSTORM_CONTEXT 线程同步：true
17:02:13.188 [main] DEBUG club.kingyin.test.context.impl.RainstormContext - 语言 猪皮
17:02:13.188 [defaultEventLoopGroup-2-1] INFO club.kingyin.test.context.impl.RainstormContext - 获取RAINSTORM_CONTEXT 线程同步：true
17:02:13.188 [defaultEventLoopGroup-2-1] DEBUG club.kingyin.test.context.impl.RainstormContext - 语言 猪皮
17:02:13.193 [defaultEventLoopGroup-3-1] INFO club.kingyin.test.context.impl.RainstormContext - 获取RAINSTORM_CONTEXT 线程同步：true
17:02:13.193 [defaultEventLoopGroup-3-1] DEBUG club.kingyin.test.context.impl.RainstormContext - 语言 猪皮
17:02:13.194 [Thread-1] INFO club.kingyin.test.context.impl.RainstormContext - 获取RAINSTORM_CONTEXT 线程同步：true
17:02:13.194 [Thread-1] DEBUG club.kingyin.test.context.impl.RainstormContext - 语言 猪皮

```



## 原理

暂未更新

---
title: Spring 中如何配置默认异步线程池
tags:
  - spring boot
  - 异步
status: doing
createDate: 2024-01-29
---

[[Java-04b.Spring 中通过 TaskDecorator 配置默认异步线程池]]

## 前言

虽然在 SpringBoot 2.7.x 中已经有关于异步线程池的默认配置，但如果还是要自定义的需求，仍然值得学习了解一下。

例如：想要在多线程池中添加 traceId；使用 `transmittable-thread-local` 来代替默认的 `ThreadLocal`。

## 多线程日志追踪工具类

### MdcUtil

```java
public class MdcUtil {  
    public static final String TRACE_ID = "traceId";  
  
    public static String generateTraceId() {  
        return UUID.randomUUID().toString().replace("-", "");  
    }  
  
    public static String getTraceId() {  
        return MDC.get(TRACE_ID);  
    }  
  
    public static void setTraceId(String traceId) {  
        MDC.put(TRACE_ID, traceId);  
    }  
  
    public static void setContextMap(Map<String, String> context) {  
        MDC.setContextMap(context);  
    }  
  
    public static void removeTraceId() {  
        MDC.remove(TRACE_ID);  
    }  
  
    public static void clear() {  
        MDC.clear();  
    }  
}
```


### ThreadMdcUtil

```java
public class ThreadMdcUtil {  
    public static void setTraceIdIfAbsent() {  
        if (MdcUtil.getTraceId() == null) {  
            MdcUtil.setTraceId(MdcUtil.generateTraceId());  
        }  
    }  
  
    public static <T> Callable<T> wrap(final Callable<T> callable, final Map<String, String> context) {  
        return () -> {  
            if (context == null) {  
                MdcUtil.clear();  
            } else {  
                MdcUtil.setContextMap(context);  
            }  
            setTraceIdIfAbsent();  
            try {  
                return callable.call();  
            } finally {  
                MdcUtil.clear();  
            }  
        };  
    }  
  
    public static Runnable wrap(final Runnable runnable, final Map<String, String> context) {  
        return () -> {  
            if (context == null) {  
                MdcUtil.clear();  
            } else {  
                MdcUtil.setContextMap(context);  
            }  
            //设置traceId  
            setTraceIdIfAbsent();  
            try {  
                runnable.run();  
            } finally {  
                MdcUtil.clear();  
            }  
        };  
    }  
}
```

## 自定义 ThreadPoolTaskExecutor

```java
/**  
 * 日志追踪线程池配置  
 *  
 * @author fengxc 
 */
 public class CustomThreadPoolTaskExecutor extends ThreadPoolTaskExecutor {  
  
    @Override  
    public void execute(@NotNull Runnable task) {  
        super.execute(ThreadMdcUtil.wrap(task, MDC.getCopyOfContextMap()));  
    }  
  
    @NotNull  
    @Override    
    public Future<?> submit(@NotNull Runnable task) {  
        return super.submit(ThreadMdcUtil.wrap(task, MDC.getCopyOfContextMap()));  
    }  
  
    @NotNull  
    @Override    
    public <T> Future<T> submit(@NotNull Callable<T> task) {  
        return super.submit(ThreadMdcUtil.wrap(task, MDC.getCopyOfContextMap()));  
    }  
}
```

## 继承 AsyncConfigurerSupport 实现默认的异步线程池

```java
@EnableAsync  
@SpringBootConfiguration  
@EnableConfigurationProperties(TaskExecutionProperties.class)  
public class ThreadPoolConfig extends AsyncConfigurerSupport {  
  
    @Resource  
    private TaskExecutionProperties properties;  
  
    /**  
     * 重写默认线程池配置，@Async异步会使用这个线程池  
     */  
    @Override  
    public Executor getAsyncExecutor() {  
        TaskExecutionProperties.Pool pool = properties.getPool();  
        TaskExecutorBuilder builder = new TaskExecutorBuilder();  
        builder = builder.queueCapacity(pool.getQueueCapacity());  
        builder = builder.corePoolSize(pool.getCoreSize());  
        builder = builder.maxPoolSize(pool.getMaxSize());  
        builder = builder.allowCoreThreadTimeOut(pool.isAllowCoreThreadTimeout());  
        builder = builder.keepAlive(pool.getKeepAlive());  
        Shutdown shutdown = properties.getShutdown();  
        builder = builder.awaitTermination(shutdown.isAwaitTermination());  
        builder = builder.awaitTerminationPeriod(shutdown.getAwaitTerminationPeriod());  
        builder = builder.threadNamePrefix(properties.getThreadNamePrefix());  
  
        CustomThreadPoolTaskExecutor executor = builder.build(CustomThreadPoolTaskExecutor.class);  
        executor.initialize();  
  
        return TtlExecutors.getTtlExecutor(executor);  
    }  
  
    @Override  
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {  
        return new SimpleAsyncUncaughtExceptionHandler();  
    }  
  
}
```
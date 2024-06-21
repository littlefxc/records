
---
title: Spring 中通过 TaskDecorator 配置默认异步线程池
tags:
  - spring boot
  - 异步
status: doing
createDate: 2024-01-29
---

[[Java-04a.Spring 中通过 ThreadPoolTaskExecutor 和 AsyncConfigurerSupport 配置默认异步线程池]]

## 前言

在Spring框架中，TaskDecorator 是一个接口，它可以用来自定义由 ThreadPoolTaskExecutor 或其他任务执行器管理的任务的装饰行为。这通常用于在执行任务之前和之后添加某些上下文相关的行为，比如设置线程上下文或者清理资源。

例如，在执行异步操作时，你可能需要将主线程的一些上下文信息（比如用户身份验证令牌或请求上下文信息）传递给执行异步操作的线程。TaskDecorator 就可以在这种场景下发挥作用。

## 自定义 TaskDecorator

```java
public class ThreadPoolContextTaskDecorator implements TaskDecorator {  
  
    @Override  
    public Runnable decorate(Runnable runnable) {  
       return TtlRunnable.get(ThreadMdcUtil.wrap(runnable, MDC.getCopyOfContextMap()));  
    }  
  
}
```

## 可以继承 AsyncConfigurerSupport 实现默认的异步线程池

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
        
        builder.taskDecorator(new ThreadPoolContextTaskDecorator());  // 重点是这里
  
        ThreadPoolTaskExecutor executor = builder.build();  
        executor.initialize();  
  
        return executor;  
    }  
  
    @Override  
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {  
        return new SimpleAsyncUncaughtExceptionHandler();  
    }  
  
}
```

## 也可以直接定义一个 TaskDecorator Bean

### 查看源码 TaskExecutionAutoConfiguration

```java
@ConditionalOnClass(ThreadPoolTaskExecutor.class)  
@AutoConfiguration  
@EnableConfigurationProperties(TaskExecutionProperties.class)  
public class TaskExecutionAutoConfiguration {  
  
    /**  
     * Bean name of the application {@link TaskExecutor}.  
     */    
    public static final String APPLICATION_TASK_EXECUTOR_BEAN_NAME = "applicationTaskExecutor";  
  
    @Bean  
    @ConditionalOnMissingBean    
    public TaskExecutorBuilder taskExecutorBuilder(TaskExecutionProperties properties,  
          ObjectProvider<TaskExecutorCustomizer> taskExecutorCustomizers,  
          ObjectProvider<TaskDecorator> taskDecorator) {  
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
       builder = builder.customizers(taskExecutorCustomizers.orderedStream()::iterator);  
       
       // spring-boot-autoconfigure 这里提供了 TaskDecorator 的入口
       // 只需要提供一个 TaskDecorator 的Bean 就可以复用 Spring Boot 中的异步线程池啦
       builder = builder.taskDecorator(taskDecorator.getIfUnique());

       return builder;  
    }  
  
    @Lazy  
    @Bean(name = { APPLICATION_TASK_EXECUTOR_BEAN_NAME,  
          AsyncAnnotationBeanPostProcessor.DEFAULT_TASK_EXECUTOR_BEAN_NAME })  
    @ConditionalOnMissingBean(Executor.class)  
    public ThreadPoolTaskExecutor applicationTaskExecutor(TaskExecutorBuilder builder) {  
       return builder.build();  
    }  
  
}
```
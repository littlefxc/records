---
title: 线程池如何传递ThreadLocal
status: Done
Tags:
  - Java
  - ThreadLocal
---

## 前言

在做分布式链路追踪系统的时候，需要解决异步调用透传上下文的需求，特别是传递traceId，本文就线程池透传几种方式进行分析。

其他典型场景例子：

1. 分布式跟踪系统 或 全链路压测（即链路打标）
2. 日志收集记录系统上下文
3. `Session`级`Cache`
4. 应用容器或上层框架跨应用代码给下层`SDK`传递信息

## 1、JDK对跨线程传递ThreadLocal的支持

首先看一个最简单场景，也是一个错误的例子。

```java
void testThreadLocal(){
    ThreadLocal<Object> threadLocal = new ThreadLocal<>();
    threadLocal.set("not ok");
    new Thread(()->{
        System.out.println(threadLocal.get());
    }).start();
}
```

java中的threadlocal，是绑定在线程上的。你在一个线程中set的值，在另外一个线程是拿不到的。

上面的输出是:

`null`

### 1.1 InheritableThreadLocal 例子

JDK考虑了这种场景，实现了InheritableThreadLocal ,不要高兴太早,这个只是支持父子线程，线程池会有问题。

我们看下InheritableThreadLocal的例子：

```
        InheritableThreadLocal<String> itl = new InheritableThreadLocal<>();
        itl.set("father");
        new Thread(()->{
            System.out.println("subThread:" + itl.get());
            itl.set("son");
            System.out.println(itl.get());
        }).start();

        Thread.sleep(500);//等待子线程执行完

        System.out.println("thread:" + itl.get());

```

上面的输出是：

`subThread:father` //子线程可以拿到父线程的变量

`son`

`thread:father` //子线程修改不影响父线程的变量

### 1.2 InheritableThreadLocal的实现原理

有同学可能想知道InheritableThreadLocal的实现原理，其实特别简单。就是Thread类里面分开记录了ThreadLocal、InheritableThreadLocal的ThreadLocalMap，初始化的时候，会拿到parent.InheritableThreadLocal。直接上代码可以看的很清楚。

```java
class Thread {

    // ...

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;

  // ...

  if (inheritThreadLocals && parent.inheritableThreadLocals != null)
    this.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
}
```

`JDK`的`[InheritableThreadLocal](<https://docs.oracle.com/javase/10/docs/api/java/lang/InheritableThreadLocal.html>)`类可以完成父线程到子线程的值传递。但对于使用线程池等会池化复用线程的执行组件的情况，线程由线程池创建好，并且线程是池化起来反复使用的；这时父子线程关系的`ThreadLocal`值传递已经没有意义，应用需要的实际上是把 任务提交给线程池时的`ThreadLocal`值传递到 任务执行时。

## 2、日志MDC/Opentracing的实现

如果你的应用实现了Opentracing的规范，比如通过`skywalking`的agent对线程池做了拦截，那么自定义Scope实现类，可以跨线程传递MDC，然后你的义务可以通过设置MDC的值，传递给子线程。

代码如下：

```java
this.scopeManager = scopeManager;
this.wrapped = wrapped;
this.finishOnClose = finishOnClose;
this.toRestore = (OwlThreadLocalScope)scopeManager.tlsScope.get();
scopeManager.tlsScope.set(this);
if (wrapped instanceof JaegerSpan) {
    this.insertMDC(((JaegerSpan)wrapped).context());
} else if (wrapped instanceof JaegerSpanWrapper) {
    this.insertMDC(((JaegerSpanWrapper)wrapped).getDelegated().context());
}
```

## 3、阿里transmittable-thread-local

github地址：[https://github.com/alibaba/transmittable-thread-local](https://github.com/alibaba/transmittable-thread-local)

TransmittableThreadLocal（TTL）是框架/中间件缺少的Java™std lib（简单和0依赖），提供了增强的InheritableThreadLocal，即使使用线程池组件也可以在线程之间传输值。

### 3.1 transmittable-thread-local 官方readme参考

使用类`[TransmittableThreadLocal](<https://github.com/alibaba/transmittable-thread-local/blob/master/src/main/java/com/alibaba/ttl/TransmittableThreadLocal.java>)`来保存值，并跨线程池传递。

`TransmittableThreadLocal`继承`InheritableThreadLocal`，使用方式也类似。相比`InheritableThreadLocal`，添加了

1. `copy`方法用于定制 任务提交给线程池时 的`ThreadLocal`值传递到 任务执行时 的拷贝行为，缺省传递的是引用。注意：如果跨线程传递了对象引用因为不再有线程封闭，与`InheritableThreadLocal.childValue`一样，使用者/业务逻辑要注意传递对象的线程
2. `protected`的`beforeExecute`/`afterExecute`方法执行任务(`Runnable`/`Callable`)的前/后的生命周期回调，缺省是空操作。

### 3.2 transmittable-thread-local 代码例子

方式一：TtlRunnable封装：

```java
ExecutorService executorService = Executors.newCachedThreadPool();
TransmittableThreadLocal<String> context = new TransmittableThreadLocal<>();

// =====================================================
// 在父线程中设置
context.set("value-set-in-parent");

// 额外的处理，生成修饰了的对象ttlRunnable
Runnable ttlRunnable = TtlRunnable.get(() -> {
    System.out.println(context.get());
});
executorService.submit(ttlRunnable);

```

方式二：ExecutorService封装：

```java
ExecutorService executorService = ...
// 额外的处理，生成修饰了的对象executorService
executorService = TtlExecutors.getTtlExecutorService(executorService);
```

方式三：使用java agent，无代码入侵

这种方式，实现线程池的传递是透明的，业务代码中没有修饰`Runnable`或是线程池的代码。即可以做到应用代码 无侵入。

```java
ExecutorService executorService = Executors.newCachedThreadPool();
TransmittableThreadLocal<String> context = new TransmittableThreadLocal<>();
// =====================================================
// 在父线程中设置
context.set("value-set-in-parent");

executorService.submit(() -> {
    System.out.println(context.get());
});

```

## 4、grpc的实现

grpc是一种分布式调用协议和实现，也封装了一套跨线程传递上下文的实现。

`io.grpc.Context` 表示上下文，用来在一次grpc请求链路中传递用户登录信息、tracing信息等。

Context常用用法如下。首先获取当前context,这个一般是作为参数传过来的，或通过current()获取当前的已有context。

然后通过attach方法，绑定到当前线程上，并且返回当前线程

```java
public Runnable wrap(final Runnable r) {
    return new Runnable() {
        @Override
        public void run() {
            Context previous = attach();
            try {
                r.run();
            } finally {
                detach(previous);
            }
        }
    };
}
```

Context的主要方法如下

- attach() attach Context自己，从而进入到一个新的scope中，新的scope以此Context实例作为current，并且返回之前的current context
- detach(Context toDetach) attach()方法的反向方法，退出当前Context并且detach到toDetachContext，每个attach方法要对应一个detach，所以一般通过try finally代码块或wrap模板方法来使用。
- static storage() 获取storage，Storage是用来attach和detach当前context用的。

线程池传递实现：

```java
ExecutorService executorService = Executors.newCachedThreadPool();
Context.withValue("key","value");

execute(Context.current().wrap(() -> {
            System.out.println(Context.current().getValue("key"));
        }));

```

## 5、总结

以上总结的四种实现跨线程传递的方法，最简单的就是自己定义一个Runnable，添加属性传递即可。如果考虑通用型，需要中间件封装一个Executor对象，类似transmittable-thread-local的实现，或者直接使用transmittable-thread-local。

实践的项目中，考虑周全，要支持span、MDC、rpc上下文、业务自定义上下文，可以参考以上方法封装。

## 参考资料

- [grpc源码分析1-context](https://www.codercto.com/a/66559.html)
- [threadlocal变量透传，这些问题你都遇到过吗？](https://cloud.tencent.com/developer/article/1492379)

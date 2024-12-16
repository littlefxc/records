---
title: Java中如何保障线程安全的常用方法
status: Done
Tags:
  - Java
---

## 概述

  

Java 高并发场景下保证线程的常用方法有以下几种：

1. 无状态

2. 不可变

3. 安全的发布

4. volatile

5. synchronized

6. lock

7. cas

8. threadlocal

  

## 无状态

  

我们都知道只有多个线程访问公共资源的时候，才可能出现数据安全问题，那么如果我们没有公共资源，是不是就没有这个问题呢？

  

```java

public class NoStatusService {

  

    public void add(String status) {

        System.out.println("add status:" + status);

    }

  

    public void update(String status) {

        System.out.println("update status:" + status);

    }

}

```

  

## 不可变

  

如果多个线程访问公共资源是不可变的，也不会出现数据的安全性问题。

  

```java

public class NoChangeService {

    public static final String DEFAULT_NAME = "abc";

  

    public void add(String status) {

        System.out.println("add status:" + status);

    }

}

```

  

## 安全的发布

  

如果类中有公共资源，但是没有对外开放访问权限，即对外安全发布，也没有线程安全问题。

  

```java

public class SafePublishService {

    private String name;

  

    public String getName() {

        return name;

    }

  

    public void add(String status) {

        System.out.println("add status:" + status);

    }

}

```

  

## volatile

  

如果有些公告资源只是一个开关，只要求可见性，不要求原子性，这样可以用 volatile 关键字定义来解决问题。

  

```java

public class FlagService {

    public volatile boolean flag = false;

  

    public void change() {

        if (flag) {

            System.out.println("return");

            return;

        }      

       flag = true;

        System.out.println("change");

    }

}

```

  

## synchronized

  

使用 JDK 内部提供的同步机制，这也是使用比较多的手段，分为**方法同步**和**代码块同步**，我们优先使用影响范围较小、性能更好的一种，即代码块同步。每个对象内部都有一把锁，只有拿到那把锁的线程，才能进入代码块里，代码块执行完之后，会自动释放锁。

  

```java

public class SyncService {

    private int age = 1;

  

    public synchronized void add(int i) {

        age = age + i;        

        System.out.println("age:" + age);

    }

  

    public void update(int i) {

        synchronized (this) {

            age = age + i;                          

            System.out.println("age:" + age);

        }    

     }

}

```

  

## lock

  

除了使用 synchronized 关键字实现同步功能外，JDK 还提供了 lock 显示锁的方式。它包含：可重入锁、读写锁等更多更强大的功能。

  

```java

public class LockService {

    private ReentrantLock reentrantLock = new ReentrantLock();

    public int age = 1;

    public void add(int i) {

        try {

            reentrantLock.lock();

            age = age + i;          

            System.out.println("age:" + age);

        } finally {

            reentrantLock.unlock();        

        }    

   }

}

```

  

## cas

  

JDK除了使用锁的机制解决多线程情况下数据安全问题之外，还提供了cas机制。这种机制是使用 CPU 中比较和交换指令的原子性，JDK 里面是通过 Unsafe 类实现的。cas需要四个值：旧数据、期望数据、新数据和地址，比较旧数据和期望的数据如果一样的话，就把旧数据改成新数据，当前线程不断自旋，一直到成功为止。不过可能会出现aba问题，需要使用AtomicStampedReference增加版本号解决。其实，实际工作中很少直接使用Unsafe类的，一般用atomic包下面的类即可。

  

```java

public class AtomicService {

    private AtomicInteger atomicInteger = new AtomicInteger();

    public int add(int i) {

        return atomicInteger.getAndAdd(i);

    }

}

```

  

## threadlocal

  

除了上面几种解决思路之外，JDK还提供了另外一种用空间换时间的新思路：threadlocal。它的核心思想是：共享变量在每个线程都有一个副本，每个线程操作的都是自己的副本，对另外的线程没有影响。特别注意，使用threadlocal时，使用完之后，要记得调用 remove 方法，不然可能会出现内存泄露问题。

  

```java

public class ThreadLocalService {

    private ThreadLocal<Integer> threadLocal = new ThreadLocal<>();

  

    public void add(int i) {

        Integer integer = threadLocal.get();

        threadLocal.set(integer == null ? 0 : integer + i);

    }

  

}

```
---
{}
---


## CountDownLatch

> A synchronization aid that allows one or more threads to wait until
> 
> - a set of operations being performed in other threads completes.

翻译：CountDownLatch是一个异步辅助类，它能让一个和多个线程处于等待状态，直到其他线程完成了一些列操作。

比如某个线程需要其他线程执行完毕才能执行其他的：

```java 
//调用await()方法的线程会被挂起，它会等待直到count值为0才继续执行
public void await() throws InterruptedException { };
//和await()类似，只不过等待一定的时间后count值还没变为0的话就会继续执行
public boolean await(long timeout, TimeUnit unit) throws InterruptedException { };  
//将count值减1
public void countDown() { };  
```

示例:

```java
public class CountDownLautchTest {


    public static void main(String[] args) {
       final CountDownLatch latch = new CountDownLatch(2);
       new Thread(() -> {

           System.out.println("子线程1执行开始");
           try {
               Thread.sleep(3000);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
           System.out.println("子线程1执行结束");
           latch.countDown();

       }).start();


        new Thread(() -> {

            System.out.println("子线程2执行开始");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("子线程2执行结束");
            latch.countDown();

        }).start();

        try {
            latch.await();
            System.out.println("所有子线程执行完毕了。。。");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

执行结果

```
子线程 2 开始执行
子线程 1 开始执行
子线程 1 结束执行
子线程 2 结束执行
所有子线程执行完毕了。。。
```

## CyclicBarrier

> A synchronization aid that allows a set of threads to all wait for each other to reach a common barrier point. CyclicBarriers are useful in programs involving a fixed sized party of threads that must occasionally wait for each other. The barrier is called _cyclic_ because it can be re-used after the waiting threads are released.

翻译：回环栏栅这个类，让所有的现场都去相互等待，知道它们都到达了一个栏栅的点。

字面意思回环栅栏，通过它可以实现让一组线程等待至某个状态之后再全部同时执行。叫做回环是因为当所有等待线程都被释放以后，CyclicBarrier可以被重用。我们暂且把这个状态就叫做barrier，当调用await()方法之后，线程就处于barrier了。

```java
public CyclicBarrier(int parties, Runnable barrierAction) {}
 
public CyclicBarrier(int parties) {}
```

参数parties指让多少个线程或者任务等待至barrier状态；参数barrierAction为当这些线程都达到barrier状态时会执行的内容。

CyclicBarrier最重要的是中最重要的方法就是await方法，

```java
public int await() throws InterruptedException, BrokenBarrierException { };
```

用来挂起当前线程，直至所有线程都到达barrier状态再同时执行后续任务；

```java
/**  
 * 它允许一组线程互相等待，直到所有线程都到达某个公共屏障点 (barrier point)。一旦所有线程都到达屏障点，它们就可以继续执行后续任务。  
 * CyclicBarrier 的名字中的 "Cyclic" 表示这个屏障是可以循环使用的，即在一组线程通过后，它可以被重置以便下一组线程使用。  
 * @author JoJo  
 */
 public class CyclicBarrierTest extends Thread {  
  
    public static void main(String[] args) {  
       Runnable barrierAction = () -> {  
          System.out.println("---------------------------------------");  
          System.out.println(Thread.currentThread().getName() + " 所有线程执行完之后, 执行屏障自己的任务...");  
          System.out.println("---------------------------------------");  
       };  
       CyclicBarrier cyclicBarrier = new CyclicBarrier(3, barrierAction);  
       for (int i = 1; i <= 3; i++) {  
          new WorkThread("线程" + i, cyclicBarrier).start();  
       }  
    }  
  
    static class WorkThread extends Thread {  
  
       private CyclicBarrier cyclicBarrier;  
  
       public WorkThread (String name, CyclicBarrier cyclicBarrier) {  
          setName(name);  
          this.cyclicBarrier = cyclicBarrier;  
       }  
  
       @Override  
       public void run() {  
          System.out.println(getName() + "执行一些任务...");  
  
          try {  
             Thread.sleep(3000);  
             System.out.println(getName() + "任务执行完毕, 等待其它线程...");  
             cyclicBarrier.await();  
          } catch (InterruptedException | BrokenBarrierException e) {  
             e.printStackTrace();  
          }  
  
          System.out.println(getName() + "通过屏障");  
       }  
  
    }  
  
}
```

控制台打印:

```
线程1执行一些任务...
线程2执行一些任务...
线程3执行一些任务...
线程2任务执行完毕, 等待其它线程...
线程1任务执行完毕, 等待其它线程...
线程3任务执行完毕, 等待其它线程...
---------------------------------------
线程3 所有线程执行完之后, 执行屏障自己的任务...
---------------------------------------
线程2通过屏障
线程1通过屏障
线程3通过屏障
```

## Semaphore

>A counting semaphore. Conceptually, a semaphore maintains a set of
- permits. Each {@link #acquire} blocks if necessary until a permit is
- available, and then takes it. Each {@link #release} adds a permit,
- potentially releasing a blocking acquirer.
- However, no actual permit objects are used; the {@code Semaphore} just
- keeps a count of the number available and acts accordingly.

翻译：计数信号量从概念上讲，信号量维护着一组信号许可证。 每个{@link #acquire}都会根据需要进行阻止，直到获得许可可用，然后把它。 每个{@link #release}都会添加一个许可证，潜在地释放阻止的收购方。但是，没有使用实际的许可证对象; 只是信号量而已保持可用数量的计数，并采取相应的行动。

Semaphore翻译成字面意思为 信号量，Semaphore可以控同时访问的线程个数，通过 acquire() 获取一个许可，如果没有就等待，而 release() 释放一个许可。

使用Semaphore时，你需要在需要访问共享资源的代码段前后使用`acquire()`和`release()`方法来获取和释放信号量。这样可以确保在超过信号量允许的线程数量时，限制并发访问共享资源的线程数量，实现线程间的同步和互斥。

```java
//参数permits表示许可数目，即同时可以允许多少线程进行访问
public Semaphore(int permits) {          
    sync = new NonfairSync(permits);
}
//这个多了一个参数fair表示是否是公平的，即等待时间越久的越先获取许可
public Semaphore(int permits, boolean fair) {    
    sync = (fair)? new FairSync(permits) : new NonfairSync(permits);
}
```

下面说一下Semaphore类中比较重要的几个方法，首先是acquire()、release()方法：

```java
//获取一个许可
public void acquire() throws InterruptedException { }     
//获取permits个许可
public void acquire(int permits) throws InterruptedException {}    
//释放一个许可
public void release() {}   
//释放permits个许可
public void release(int permits) { }    
```

acquire()用来获取一个许可，若无许可能够获得，则会一直等待，直到获得许可。

release()用来释放许可。注意，在释放许可之前，必须先获获得许可。

这4个方法都会被阻塞，如果想立即得到执行结果，可以使用下面几个方法：

```java
//尝试获取一个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire() { };    
//尝试获取一个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException { };  
//尝试获取permits个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire(int permits) { }; 
//尝试获取permits个许可，若在
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) throws InterruptedException { }; 
```

另外还可以通过availablePermits()方法得到可用的许可数目。

```java
/**  
 * 使用Semaphore时，你需要在需要访问共享资源的代码段前后使用acquire()和release()方法来获取和释放信号量。  
 * 这样可以确保在超过信号量允许的线程数量时，限制并发访问共享资源的线程数量，实现线程间的同步和互斥。  
 * @author JoJo  
 */
 public class SemaphoreTest {  
  
    public static void main(String[] args) {  
       Semaphore semaphore = new Semaphore(3);  
       for (int i = 1; i <= 10; i++) {  
          String workerName = "工人" + i;  
          new Thread(new Worker(semaphore), workerName).start();  
       }  
    }  
  
    static class Worker implements Runnable {  
  
       private Semaphore semaphore;  
  
       public Worker(Semaphore semaphore) {  
          this.semaphore = semaphore;  
       }  
  
       @Override  
       public void run() {  
          try {  
             String name = Thread.currentThread().getName();  
             semaphore.acquire();  
             System.out.println(name + "占用一个机器生产...");  
             Thread.sleep(2000);  
             System.out.println(name + "释放出机器");  
             semaphore.release();  
          } catch (InterruptedException e) {  
             e.printStackTrace();  
          }  
       }  
  
    }  
}
```

## 总结

- CountDownLatch和CyclicBarrier都能够实现线程之间的等待，只不过它们侧重点不同：
    CountDownLatch一般用于某个线程A等待若干个其他线程执行完任务之后，它才执行；
    而CyclicBarrier一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行；
    另外，CountDownLatch是不能够重用的，而CyclicBarrier是可以重用的。
- Semaphore其实和锁有点类似，它一般用于控制对某组资源的访问权限。
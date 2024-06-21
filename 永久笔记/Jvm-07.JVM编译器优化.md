---
title: JVM参数选项
status: Done
tags:
  - Java
  - JVM
createDate: 2022-01-20T11:58:09+08:00
---

## 1.3 编译器优化

### 1.3.1 字节码是如何运行的？

- 解释执行：由解释器一行一行翻译执行
  - 优势在于没有编译的等待时间
  - 性能相对差一些

- 编译执行：把字节码编译成机器码，直接执行机器码
  - 运行效率会高很多，一般认为比解释执行快一个数量级
  - 带来了额外的开销

那么如何查看自己的java是解释执行还是编译执行呢？

```java
$ java -version
java version "1.8.0_251"
Java(TM) SE Runtime Environment (build 1.8.0_251-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.251-b08, mixed mode)
```

mixed mode 代表混合执行，部分解释执行、部分编译执行。

- -Xint:设置JVM的执行模式为解释执行模式

  ```java
  $ java -Xint -version
  java version "1.8.0_251"
  Java(TM) SE Runtime Environment (build 1.8.0_251-b08)
  Java HotSpot(TM) 64-Bit Server VM (build 25.251-b08, interpreted mode)
  ```

- -Xcomp:JVM优先以编译模式运行，不能编译的，以解释模式运行

  ```java
  $ java -Xcomp -version
  java version "1.8.0_251"
  Java(TM) SE Runtime Environment (build 1.8.0_251-b08)
  Java HotSpot(TM) 64-Bit Server VM (build 25.251-b08, compiled mode)
  ```

- -Xmixed:以混合模式运行

一般情况下，我们的代码一开始一般由解释器解释执行。但是当虚拟机发现某个方法或代码块的运行特别频繁的时候，就会认为这些代码是**热点代码（如何定位？）**。为了提高热点代码的执行效率，会用即使编译器（也就是JIT）把这些热点代码编译城与本地平台相关的机器码，**并进行各层次的优化（操作系统的不同、CPU架构的不同）**

### 1.3.2 Hotspot 的即时编译器 C1

- 是一个简单快速的编译器
- 主要关注局部性的优化
- 适用于执行时间较短或启动性能有要求的程序。例如。GUI应用对界面启动速度就有一定要求。、
- 也被称为 Client Compiler

### 1.3.3 Htospot 的即时编译器 C2

- 是为长期运行的服务器端应用程序做性能调优的编译器
- 适用于执行时间较长或对峰值性能有要求的程序
- 也被称为是 Server Compiler

### 1.3.4 分层编译

从JDK7开始，正式引入了分层编译的概念，可以细分为 5 种编译级别：

- 0. 解释执行
- 1. 简单 C1 编译：会用 C1 编译器进行一些简单的优化，不开启 Profiling（JVM性能监控）
- 2. 受限的 C1 编译：仅执行带**方法调用次数**以及**循环回边执行次数**Profiling的 C1 编译
- 3. 完全C1编译：会执行带有所有Profiling的C1代码
- 4. C2 编译：使用C2编译器进行优化，该级别会启用一些编译耗时较长的优化，一些情况下会根据性能监控信息进行一些非常激进的性能优化

级别越高，应用启动越慢，优化的开销越高，峰值性能也越高。

### 1.3.5 分层编译- JVM参数配置示例

- 只想开启 C2：-XX:-TieredCompilation（禁用中间编译层（123层））
- 只想开启 C1：-XX:+TieredCompilation -XX:TieredStopAtLevel=1

### 1.3.6 如何找到热点代码？思路？

- 基于采样的热点探测

  周期性检查各个线程的栈顶，如果发现某一些方法总是出现在各个栈顶，那就说明是热点代码。

- 基于计数器的热点探测

  大致思路是为每一个方法甚至是代码块建立计数器，然后统计执行的次数，如果超过一定的阈值，那就说明它是热点代码。Hotspot虚拟机采用的就是基于计数器的热点探测。

### 1.3.7 Hotspot 内置的两类计数器

- 方法调用计数器（Invocation Counter）

  用于统计方法被调用的次数，在不开启分层编译的情况下，在 C1 编译器下的默认阈值是 1500 次，在 C2 模式下是 10000次。也可以哦那个 -XX:CompileThreshold=X 指定阈值

- 回边计数器（Back Edge Counter）

  - 用于统计一个方法中循环体代码执行的次数，在字节码中遇到控制流向后跳转的指令称为“回边”（Back Edge）。在不开启分层编译的情况下，C1 编译器心爱的默认阈值 13995，C2 默认为 10700，可使用 -XX:OnStackReplacePercentage=X指定阈值
  - 建立回边计数器的主要目的是为了触发 OSR （OnStackReplacement）编译，参考文档（[https://www.zhihu.com/question/45910849/answer/100636125](https://www.zhihu.com/question/45910849/answer/100636125)）

- 当开启分层编译时，JVM会根据当前编译的方法数以及编译线程数来动态调整阈值，-XX:CompileThreshold、-XX:OnStackReplacePercentage 都会失效。

### 1.3.8 方法调用计数器流程

![image-20210809233815237](https://img-blog.csdnimg.cn/img_convert/8f8ec473655538de9e7290f36b42b95c.png)

如果不做任何设置，方法调用次数统计的并不是方法被调用的绝对次数，而是一个相对的执行频，即一段时间之内方法被调用的次数。当超过一定的时间限度，如果方法的调用次数荏苒不足以让它提交给及时编译器编译，那这个方法的调用计数器就会减少一半，这个过程称为方法调用计数器热度的衰减，而这段时间就称为此方法统计的半衰周期。进行热度衰减的动作是在虚拟机进行垃圾手机是顺便进行的，可以使用虚拟机参数-XX:-UseCounterDecay来关闭热度衰减，让方法计数器统计方法调用的绝对次数，这样，只要系统运行时间足够长，绝大部分方法都会被编译成本地代码。另外，可以使用-XX:CounterHalfLifeTime参数设置半衰周期的时间，单位是秒。

### 1.3.9 回边计数器流程

![image-20210809234920524](https://img-blog.csdnimg.cn/img_convert/f8e7179f6bf5438940b6b779687b382d.png)
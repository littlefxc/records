---
title: JVM类加载过程
status: Done
tags:
  - Java
  - JVM
createDate: 2022-01-20T11:58:09+08:00
---

- [[JVM-08.JVM类加载过程]]
  - [[JVM-08a.类加载器]]

## 1 类加载过程

![https://gitee.com/littlefxc/oss/raw/master/images/类加载过程详解.png](https://img-blog.csdnimg.cn/img_convert/28b9b1dcc28d76e19362eb28cd548e2c.png)

### 1.1 编译

```shell
javac JVMTest1.java
```

结果产生了两个 class 文件，如下所示：

JVMTest1.class

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.fengxuechao.jvm;

public class JVMTest1 {
    public JVMTest1() {
    }

    public static void main(String[] var0) {
        Demo var1 = new Demo("aaa");
        var1.printName();
    }
}
```

Demo.class

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.fengxuechao.jvm;

class Demo {
    private String name;

    public Demo(String var1) {
        this.name = var1;
    }

    public void printName() {
        System.out.println(this.name);
    }
}
```

**反编译**

```java
javap -v -p Demo > a.txt
```

反编译结果如下：

```
// 描述信息
Classfile /Users/fengxuechao/WorkSpace/IdeaProjects/foodie/jvm/src/main/java/com/fengxuechao/jvm/Demo.class
  Last modified 2021-2-24; size 461 bytes
  MD5 checksum 4232b9a7f5bc219b5c632adabd643a3d
  Compiled from "JVMTest1.java"
// 描述信息
class com.fengxuechao.jvm.Demo
  minor version: 0
  major version: 52
  flags: ACC_SUPER
// 常量池
Constant pool:
   #1 = Methodref          #6.#17         // java/lang/Object."<init>":()V
   #2 = Fieldref           #5.#18         // com/fengxuechao/jvm/Demo.name:Ljava/lang/String;
   #3 = Fieldref           #19.#20        // java/lang/System.out:Ljava/io/PrintStream;
   #4 = Methodref          #21.#22        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #23            // com/fengxuechao/jvm/Demo
   #6 = Class              #24            // java/lang/Object
   #7 = Utf8               name
   #8 = Utf8               Ljava/lang/String;
   #9 = Utf8               <init>
  #10 = Utf8               (Ljava/lang/String;)V
  #11 = Utf8               Code
  #12 = Utf8               LineNumberTable
  #13 = Utf8               printName
  #14 = Utf8               ()V
  #15 = Utf8               SourceFile
  #16 = Utf8               JVMTest1.java
  #17 = NameAndType        #9:#14         // "<init>":()V
  #18 = NameAndType        #7:#8          // name:Ljava/lang/String;
  #19 = Class              #25            // java/lang/System
  #20 = NameAndType        #26:#27        // out:Ljava/io/PrintStream;
  #21 = Class              #28            // java/io/PrintStream
  #22 = NameAndType        #29:#10        // println:(Ljava/lang/String;)V
  #23 = Utf8               com/fengxuechao/jvm/Demo
  #24 = Utf8               java/lang/Object
  #25 = Utf8               java/lang/System
  #26 = Utf8               out
  #27 = Utf8               Ljava/io/PrintStream;
  #28 = Utf8               java/io/PrintStream
  #29 = Utf8               println
// 字段信息
{
  private java.lang.String name;
    descriptor: Ljava/lang/String;
    flags: ACC_PRIVATE
// 方法的信息
  public com.fengxuechao.jvm.Demo(java.lang.String);
    descriptor: (Ljava/lang/String;)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: aload_1
         6: putfield      #2                  // Field name:Ljava/lang/String;
         9: return
      LineNumberTable:
        line 16: 0
        line 17: 4
        line 18: 9

  public void printName();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: aload_0
         4: getfield      #2                  // Field name:Ljava/lang/String;
         7: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        10: return
      LineNumberTable:
        line 21: 0
        line 22: 10
}
SourceFile: "JVMTest1.java"
```

指令参考：[https://docs.oracle.com/javase/specs/jvms/se8/html/index.html](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)

### 1.2 加载

- JVM 是如何加载 class 文件的呢？

  当一个类被创建实例或者被引用到的时候, 如果虚拟机发现之前没有加载过这个类，就会通过类加载器（也就是 ClassLoader ）把 class 文件加载到内存。
  
  在加载的过程中主要做了 3 件事：
  
  1. 读取类的二进制流。
  2. 把二进制流转为方法区数据结构，并存放到方法区。
  3. 最后，在 Java 堆中产生 java.lang.Class 对象。
  
  文档参考：[描述一下JVM加载class文件的原理机制](https://www.cnblogs.com/williamjie/p/11167920.html)
  
### 1.3 链接

class 文件加载完成后，会进入“链接”这个步骤，链接这个步骤又可以分为“验证”、“准备”和“解析”。

#### 1.3.1 验证

验证——顾名思义，就是验证 class 文件是不是符合规范，这里面包含了多个层次的验证，包括：

- 文件格式的验证

  - 是否以 `0xCAFEBABE` 开头（可以用 16 进制编辑器打开查看）
  - 版本好是否合理

- 元数据验证

  - 是否有父类
  - 是否继承了 final 类（final 类是不能被继承的，如果继承了，那就是有问题）
  - 非抽象类是否实现了所有抽象方法（没有，那就是有问题）

- 字节码验证

  字节码的验证是非常复杂的，一个 class 文件能够通过字节码验证并不代表它没有问题，但是如果它没有通过字节码验证，那就一定有问题。

  - 运行检查
  - 栈数据类型和操作码操作参数吻合（比如栈空间只有 2 字节，但其实却需要大于 2 字节，此时就认为这个字节码是有问题的）
  - 跳转指令是不是指向合理的位置

- 符号引用验证

  - 常量池中描述类是否存在
  - 访问的方法或字段是否存在且有足够的权限

- 如果事先已经确认代码是安全无误的可以在启动的时候用`-Xverify:none`关闭验证。

#### 1.3.2 准备

如果通过验证发现没有问题的话，就会进入准备环节。

准备环节的作用：

- 为类的静态变量分配内存，初始化为系统的初始值
  - final static 修饰的变量：直接赋值为用户定义的值，比如 `private final static int value =123`，直接赋值 123。
  - 但是 `private static int value = 123`, 该阶段的值依然是 0。

#### 1.3.3 解析

“准备”完成后，就可以进入解析了。

解析的作用：把符号引用转换成直接引用。

- 什么是符号引用？

  符号引用（Symbolic References）：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能够无歧义的定位到目标即可。例如，在Class文件中它以CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info等类型的常量出现。符号引用与虚拟机的内存布局无关，引用的目标并不一定加载到内存中。在Java中，一个java类将会编译成一个class文件。在编译时，java类并不知道所引用的类的实际地址，因此只能使用符号引用来代替。比如org.simple.People类引用了org.simple.Language类，在编译时People类并不知道Language类的实际内存地址，因此只能使用符号org.simple.Language（假设是这个，当然实际中是由类似于CONSTANT_Class_info的常量来表示的）来表示Language类的地址。各种虚拟机实现的内存布局可能有所不同，但是它们能接受的符号引用都是一致的，因为符号引用的字面量形式明确定义在Java虚拟机规范的Class文件格式中。

  ![image-20210224212914755](https://img-blog.csdnimg.cn/img_convert/673d5f543e85b3a6605bf93732636f4d.png)

- 什么是直接引用？

  可以是

  - 直接指向目标的指针（比如，指向“类型”【Class对象】、类变量、类方法的直接引用可能是指向方法区的指针）
  - 相对偏移量（比如，指向实例变量、实例方法的直接引用都是偏移量）
  - 一个能间接定位到目标的句柄
  
  直接引用是和虚拟机的布局相关的，同一个符号引用在不同的虚拟机实例上翻译出来的直接引用一般不会相同。如果有了直接引用，那引用的目标必定已经被加载入内存中了。
  
#### 1.3.4 初始化

解析完成之后，就会进入初始化这个阶段。在这个阶段，JVM首先会执行 <clinit> 方法，clinit 方法由编译器自动收集里面的所有静态变量赋值动作和静态语句块合并而成，也叫类构造器方法

- 初始化的顺序和源文件中的顺序一致
- 子类的 <clinit> 被调用前，会先调用父类的 <clinit>
- JVM 会保证 clinit 方法的线程安全性

示例1：

```java
public class JVMTest2 {

    static {
        System.out.println("JVMTest2 静态块");
    }

    {
        System.out.println("JVMTest2 构造块");
    }

    public JVMTest2() {
        System.out.println("JVMTest2 构造方法");
    }

    public static void main(String[] args) {
        System.out.println("JVMTest2 main() 方法");
        new JVMTest2();
    }
}
```

执行结果：

```
JVMTest2 静态块
JVMTest2 main() 方法
JVMTest2 构造块
JVMTest2 构造方法
```

示例2:

```java
public class JVMTest3 {

    static {
        System.out.println("JVMTest3 静态块");
    }

    {
        System.out.println("JVMTest3 构造块");
    }

    public JVMTest3() {
        System.out.println("JVMTest3 构造方法");
    }

    public static void main(String[] args) {
        System.out.println("JVMTest3 main() 方法");
        new Sub();
    }
}

class Super {
    static {
        System.out.println("Super 静态块");
    }

    {
        System.out.println("Super 构造块");
    }

    public Super() {
        System.out.println("Super 构造方法");
    }
}

class Sub extends Super {
    static {
        System.out.println("Sub 静态块");
    }

    {
        System.out.println("Sub 构造块");
    }

    public Sub() {
        System.out.println("Sub 构造方法");
    }
}
```

执行结果：

```java
JVMTest3 静态块
JVMTest3 main() 方法
Super 静态块
Sub 静态块
Super 构造块
Super 构造方法
Sub 构造块
Sub 构造方法
```

### 1.4 使用

初始化完成之后就可以使用这个类了。

### 1.5 卸载

但不使用这个类的话，可以把它卸载掉。

### 1.6 小节

章节 1 的图表示的是一般的类加载流程，而事实上类加载的时候并不一定完全按照这个流程走。例如，解析不一定在初始化之前，也有可能在初始化之后去做。
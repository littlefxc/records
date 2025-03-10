---
title: JVM之对象的创建
tags:
  - Java
  - JVM
status: Done
createDate: 2023-04-14
---
# JVM之对象的创建

![https://img-blog.csdnimg.cn/img_convert/d7b5e820db571f629b11d1f67d0e4c55.png](https://img-blog.csdnimg.cn/img_convert/d7b5e820db571f629b11d1f67d0e4c55.png)

<!-- more -->

# 1 前言

Java是一门面向对象的编程语言,Java程序运行过程中无时无刻都有对象被创建出来。在语言层面上,创建对象通常(例外:复制、反序列化)仅仅是一个new关键字而已,而在虚拟机中,对象(文中讨论的对象限于普通Java对象,不包括数组和Class对象等)的创建又是怎样一个过程呢?

1. 常量池中定位类的符号引用
2. 检查符号引用所代表的类是否已被加载，解析和初始化过。如果没有,那必须先执行相应的类加载过程
3. 为新生对象分配内存
4. 初始化为零值 - 可以不赋就直接使用
5. 设置对象头信息
6. 执行对象<init>方法 - 构造方法

# 2 常量池中定位类的符号引用

当Java虚拟机遇到一条字节码new指令时,首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用,并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有,那必须先执行相应的类加载过程。

# 3 为新生对象分配内存

在类加载检查通过后,接下来虚拟机将为新生对象分配内存。对象所需内存的大小在类加载完成后便可完全确定，为对象分配空间的任务实际上便等同于把一块确定大小的内存块从Java堆中划分出来。

## 3.1 内存的分配方式

选择哪种分配方式由Java堆是否规整决定,而Java堆是否规整又由所采用的垃圾收集器是否带有空间压缩整理(Compact)的能力决定。

Serial、ParNew等带压缩整理过程的收集器对应的是指针碰撞。

CMS这种基于清除(Sweep)算法的收集器对应的是空闲列表。

### 3.1.1 指针碰撞

假设Java堆中内存是绝对规整的,所有被使用过的内存都被放在一边,空闲的内存被放在另一边,中间放着一个指针作为分界点的指示器,那所分配内存就仅仅是把那个指针向空闲空间方向挪动一段与对象大小相等的距离,这种分配方式称为“指针碰撞”(Bump The Pointer)。

### 3.1.2 空闲列表

如果Java堆中的内存并不是规整的,已被使用的内存和空闲的内存相互交错在一起,那就没有办法简单地进行指针碰撞了,虚拟机就必须维护一个列表,记录上哪些内存块是可用的,在分配的时候从列表中找到一块足够大的空间划分给对象实例,并更新列表上的记录,这种分配方式称为“空闲列表”(Free List)。

使用CMS这种基于清除(Sweep)算法的收集器。

## 3.2 分配内存的并发保证

除划分空间外还需要考虑一个问题：

对象创建在虚拟机中是非常频繁的行为,即使仅仅修改一个指针所指向的位置,在并发情况下也并不是线程安全的,可能出现正在给对象A分配内存,指针还没来得及修改,对象B又同时使用了原来的指针来分配内存的情况。

解决方法

### 3.2.1 CAS + 失败重试

是对分配内存空间的动作进行同步处理——实际上虚拟机是采用CAS配上失败重试的方式保证更新操作的原子性。

### 3.2.2 本地线程分配缓冲

是把内存分配的动作按照线程划分在不同的空间之中进行,即每个线程在Java堆中预先分配一小块内存,称为本地线程分配缓冲(Thread Local Allocation Buffer,TLAB),哪个线程要分配内存,就在哪个线程的本地缓冲区中分配,只有本地缓冲区用完了,分配新的缓存区时才需要同步锁定。

虚拟机是否使用TLAB,可以通过-XX:+/-UseTLAB参数来设定。

# 4 初始化为零值 - 可以不赋就直接使用

内存分配完成之后,虚拟机必须将分配到的内存空间(但不包括对象头)都初始化为零值,如果使用了TLAB的话,这一项工作也可以提前至TLAB分配时顺便进行。

这步操作保证了对象的实例字段在Java代码中可以不赋初始值就直接使用,使程序能访问到这些字段的数据类型所对应的零值。

# 5 设置对象头信息

Java虚拟机还要对对象进行必要的设置,例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码(实际上对象的哈希码会延后到真正调用`Object::hashCode()`方法时才计算)、对象的GC分代年龄等信息。这些信息存放在对象的对象头(Object Header)之中。

根据虚拟机当前运行状态的不同,如是否启用偏向锁等,对象头会有不同的设置方式。

这一步中从虚拟机的角度来说对象已经创建好了，但从程序的角度来说因为还没有执行构造方法，对象的创建才刚刚开始。

# 6 执行对象`<init>`方法 - 构造方法

在上面工作都完成之后,从虚拟机的视角来看,一个新的对象已经产生了。但是从Java程序的视角看来,对象创建才刚刚开始——构造函数,即Class文件中的<init>()方法还没有执行,所有的字段都为默认的零值,对象需要的其他资源和状态信息也还没有按照预定的意图构造好。

一般来说(由字节码流中new指令后面是否跟随`invokespecial`指令所决定,Java编译器会在遇到new关键字的地方同时生成这两条字节码指令,但如果直接通过其他方式产生的则不一定如此),`new`指令之后会接着执行`<init> ()`方法,按照程序员的意愿对对象进行初始化,这样一个真正可用的对象才算完全被构造出来。

# 7 HotSpot 解释器代码片段

下面是HotSpot虚拟机字节码解释器(bytecodeInterpreter.cpp)中的代码片段。这个解释器实现很少有机会实际使用,大部分平台上都使用模板解释器;当代码通过即时编译器执行时差异就更大了。不过这段代码(以及笔者添加的注释)用于了解HotSpot的运作过程是没有什么问题的。

```java
// 确保常量池中存放的是已解释的类
if (!constants->tag_at(index).is_unresolved_klass()) {
    // 断言确保是klassOop和instanceKlassOop(这部分下一节介绍)
    oop entry = (klassOop) *constants->obj_at_addr(index);
    assert(entry->is_klass(), "Should be resolved klass");
    klassOop k_entry = (klassOop) entry;
    assert(k_entry->klass_part()->oop_is_instance(), "Should be instanceKlass");
    instanceKlass* ik = (instanceKlass*) k_entry->klass_part();
    // 确保对象所属类型已经经过初始化阶段
    if ( ik->is_initialized() && ik->can_be_fastpath_allocated() ) {
      // 取对象长度
      size_t obj_size = ik->size_helper();
      oop result = NULL;
      // 记录是否需要将对象所有字段置零值
      bool need_zero = !ZeroTLAB;
      // 是否在TLAB中分配对象
      if (UseTLAB) {
        result = (oop) THREAD->tlab().allocate(obj_size);
      }
      if (result == NULL) {
        need_zero = true;
        // 直接在eden中分配对象retry:
        HeapWord* compare_to = *Universe::heap()->top_addr();
        HeapWord* new_top = compare_to + obj_size;
        // cmpxchg是x86中的CAS指令,这里是一个C++方法,通过CAS方式分配空间,并发失败的               话,转到retry中重试直至成功分配为止
        if (new_top <= *Universe::heap()->end_addr()) {
          if (Atomic::cmpxchg_ptr(new_top, Universe::heap()->top_addr(), compare_to) != compare_to) {
            goto retry;
          }
          result = (oop) compare_to;
        }
      }
      if (result != NULL) {
        // 如果需要,为对象初始化零值
        if (need_zero ) {
          HeapWord* to_zero = (HeapWord*) result + sizeof(oopDesc) / oopSize;
          obj_size -= sizeof(oopDesc) / oopSize;
          if (obj_size > 0 ) {
            memset(to_zero, 0, obj_size * HeapWordSize);
          }
        }
        // 根据是否启用偏向锁,设置对象头信息
        if (UseBiasedLocking) {
          result->set_mark(ik->prototype_header());
        } else {
          result->set_mark(markOopDesc::prototype());
        }
        result->set_klass_gap(0);
        result->set_klass(k_entry);
        // 将对象引用入栈,继续执行下一条指令
        SET_STACK_OBJECT(result, 0);
        UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
      }
    }
}
```

# 8 参考资源

《深入理解Java虚拟机：JVM高级特性与最佳特性（第三版）》
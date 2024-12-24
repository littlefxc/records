---
title: IO的五种模型
status: Done
Tags:
  - netty
  - netty基础知识
---

## Channel

![Channel是什么.png (1030×425) (gitee.com)](https://gitee.com/littlefxc/records/raw/dev/attachments/Netty-04.Java%20NIO%E7%9A%84%E6%A0%B8%E5%BF%83%E7%BB%84%E4%BB%B6/Channel%E6%98%AF%E4%BB%80%E4%B9%88.png)

Channel 是一种文件的链接，它代表的是到实体的开放连接，这个实体可以是硬件设备、文件、网络套接字或者可执行文件 IO 操作（比如读、写）的程序组件。

Channel 就是到文件的连接，并可以通过 IO 操作这些文件。

根据不同的文件类型，又衍生出了不同类型的 Channel：
- FileChannel：操作普通文件
- DatagramChannel：用于 UDP 协议
- SocketChannel：用于 TCP 协议，客户端与服务端之间的 Channel
- ServerSocketChannel：用于 TCP 协议，仅用于服务端的 Channel

示例：

```java
public class FileChannelTest {
    public static void main(String[] args) throws IOException {
        // 从文件获取一个FileChannel
        FileChannel fileChannel = new RandomAccessFile("D:\\object.txt", "rw").getChannel();
        // 声明一个Byte类型的Buffer
        ByteBuffer buffer = ByteBuffer.allocate(10);
        // 将FileChannel中的数据读出到buffer中，-1表示读取完毕
        // buffer默认为写模式
        // read()方法是相对channel而言的，相对buffer就是写
        while ((fileChannel.read(buffer)) != -1) {
            // buffer切换为读模式
            buffer.flip();
            // buffer中是否有未读数据
            while (buffer.hasRemaining()) {
                // 未读数据的长度
                int remain = buffer.remaining();
                // 声明一个字节数组
                byte[] bytes = new byte[remain];
                // 将buffer中数据读出到字节数组中
                buffer.get(bytes);
                // 打印出来
                System.out.println(new String(bytes, StandardCharsets.UTF_8));
            }
            // 清空buffer，为下一次写入数据做准备
            // clear()会将buffer再次切换为写模式
            buffer.clear();
        }
    }
}
```

## Buffer

Buffer 是存放数据的容器。

按照基本类型的角度划分：ByteBuffer、CharBuffer、ShortBuffer、IntBuffer、LongBuffer、FloatBuffer、DoubleBuffer

按照不同的内存实现划分：堆内存实现 HeapByteBuffer、直接内存实现 DirectByteBuffer

### Buffer 的结构

3个属性：

- capacity：Buffer 的容量，即能够容纳的数量
- limit：表示最大可写或者最大可读的数据
- position：表示下一次可使用的位置，针对读模式表示下一个可读的位置，针对写模式表示下一个可写的位置
![Buffer的结构.png (416×276) (gitee.com)](https://gitee.com/littlefxc/records/raw/dev/attachments/Netty-04.Java%20NIO%E7%9A%84%E6%A0%B8%E5%BF%83%E7%BB%84%E4%BB%B6/Buffer%E7%9A%84%E7%BB%93%E6%9E%84.png)

几个重要的方法：
- 分配一个 Buffer：allocate ()
- 写入数据：buf.put () 或者 channel.read (buf)，read 为 read to 的意思，从 channel 读出并写入 buffer
- 切换为读模式：buf.flip ()
- 读取数据：buf.read () 或者 channel.write (buf)，write 为 write from 的意思，从 buffer 读出并写入 channel
- 重新读取或重新写入：rewind ()，重置 position 为 0，limit 和 capacity 保持不变，可以重新读取或重新写入数据
- 清空数据：buf.clear ()，清空所有数据
- 压缩数据：buf.compact ()，清除已读取的数据，并将未读取的数据往前移

示例：

```java
public class FileChannelTest {
    public static void main(String[] args) throws IOException {
        // 从文件获取一个FileChannel
        FileChannel fileChannel = new RandomAccessFile("D:\\object.txt", "rw").getChannel();
        // 分配一个Byte类型的Buffer
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        // 将FileChannel中的数据读出到buffer中，-1表示读取完毕
        // buffer默认为写模式
        // read()方法是相对channel而言的，相对buffer就是写
        while ((fileChannel.read(buffer)) != -1) {
            // buffer切换为读模式
            buffer.flip();
            // buffer中是否有未读数据
            while (buffer.hasRemaining()) {
                // 读取数据
                System.out.print((char)buffer.get());
            }
            // 清空buffer，为下一次写入数据做准备
            // clear()会将buffer再次切换为写模式
            buffer.clear();
        }
    }
}
```

## Selector

Selector 是一个（SelectableChannel的）多路复用器

![Selector和Channel的关系.png (395×221) (gitee.com)](https://gitee.com/littlefxc/records/raw/dev/attachments/Netty-04.Java%20NIO%E7%9A%84%E6%A0%B8%E5%BF%83%E7%BB%84%E4%BB%B6/Selector%E5%92%8CChannel%E7%9A%84%E5%85%B3%E7%B3%BB.png)

Selector 和 Channel 是什么关系？Selector 和 Channel 是一对多关系，一个Selector 可以为多个 Channel 服务，监听它们准备好的事件。

### Selector 的用法

```java
// 创建一个Selector
Selector selector = Selector.open();

// 注册事件到Selector上
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```

Channel 注册到 Selector 上之后，返回一个叫做 SelectionKey 的对象。

### SelectionKey 是什么？

一般地，我们在饭店点完菜之后，都会给一个牌子到你手上，服务员通过这个牌子可以找到你，你通过这个牌子可以去拿饭。SelectionKey 就相当于是这个牌子，它将 Channel 和 Selector 的牢牢地绑定在一起，并保存着你感兴趣的事件。

### 事件是什么？

事件是 Channel 感兴趣的事情，比如读事件、写事件等等，在 Java 中，定义了四种事件，位于 SelectionKey 这个类中：

- 读事件：SelectionKey.OP_READ = 1 << 0 = 0000 0001
- 写事件：SelectionKey.OP_WRITE = 1 << 2 = 0000 0100
- 连接事件：SelectionKey.OP_CONNECT = 1 << 3 = 0000 1000
- 接受连接事件：SelectionKey.OP_ACCEPT = 1 << 4 = 0001 0000

这四种事件的二进制都是错开的，故此我们可以使用“位或”操作监听多种感兴趣的事件：

```java
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```

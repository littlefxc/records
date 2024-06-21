---
{}
---



## 简介

[官网文档]([Protocol Buffers Documentation (protobuf.dev)](https://protobuf.dev/))

protobuf是一种用于**序列化结构数据**的工具，实现**数据的存储与交换**，与编程语言和开发平台无关。

定义数据的结构，然后使用protoc编译器生成源代码，在各种数据流中使用各种语言进行编写和读取结构数据。甚至可以更新数据结构，而不破坏由旧数据结构编译的已部署程序。

## 优点

- **性能高效**: 与XML相比，protobuf 更小（3 ~ 10倍）、更快（20 ~ 100倍）、更为简单。
- **语言无关、平台无关**：protobuf 支持 Java、C++、Python 等多种语言，支持多个平台。
- **扩展性、兼容性强**：只需要使用protobuf对结构数据进行一次描述，即可从各种数据流中读取结构数据，更新数据结构时不会破坏原有的程序。

## 缺点

- 不适合用来对**基于文本的标记文档**（如 HTML）建模。
- **自解释性较差**，数据存储格式为二进制，需要通过proto文件才能了解到内部的数据结构。

## 应用场景

- **压缩效率高**：服务器间的海量数据传输与通信，可以节省磁盘和带宽，protobuf适合处理大数据集中的单个小消息，但并不适合处理单个的大消息。
- **解析速度快**：可以提高服务器的吞吐能力。

## 安装

[protoc的源码和各个系统的预编译包](https://github.com/protocolbuffers/protobuf/releases)

### windows

1. 选择对应的安装文件下载 
2. 添加到环境变量中

```sh
$ protoc --version
libprotoc 3.19.1
```

出现这个表示安装完成

### linux

```sh
$ sudo apt-get install autoconf automake libtool curl make g++ unzip
$ git clone https://github.com/google/protobuf.git
$ cd protobuf
$ git submodule update --init --recursive
$ ./autogen.sh
$ ./configure
$ make
$ make check
$ sudo make install
$ sudo ldconfig
```

## proto3 数据类型和其它语言的类型对应关系

A scalar message field can have one of the following types – the table shows the type specified in the file, and the corresponding type in the automatically generated class:`.proto`

| .proto Type | Notes                                                                                                                                           | C++ Type | Java/Kotlin Type[1] | Python Type[3]                       | Go Type | Ruby Type                      | C# Type    | PHP Type          | Dart Type |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ------------------- | ------------------------------------ | ------- | ------------------------------ | ---------- | ----------------- | --------- |
| double      |                                                                                                                                                 | double   | double              | float                                | float64 | Float                          | double     | float             | double    |
| float       |                                                                                                                                                 | float    | float               | float                                | float32 | Float                          | float      | float             | double    |
| int32       | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead. | int32    | int                 | int                                  | int32   | Fixnum or Bignum (as required) | int        | integer           | int       |
| int64       | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead. | int64    | long                | int/long[4]                          | int64   | Bignum                         | long       | integer/string[6] | Int64     |
| uint32      | Uses variable-length encoding.                                                                                                                  | uint32   | int[2]              | int/long[4]                          | uint32  | Fixnum or Bignum (as required) | uint       | integer           | int       |
| uint64      | Uses variable-length encoding.                                                                                                                  | uint64   | long[2]             | int/long[4]                          | uint64  | Bignum                         | ulong      | integer/string[6] | Int64     |
| sint32      | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s.                            | int32    | int                 | int                                  | int32   | Fixnum or Bignum (as required) | int        | integer           | int       |
| sint64      | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s.                            | int64    | long                | int/long[4]                          | int64   | Bignum                         | long       | integer/string[6] | Int64     |
| fixed32     | Always four bytes. More efficient than uint32 if values are often greater than 228.                                                             | uint32   | int[2]              | int/long[4]                          | uint32  | Fixnum or Bignum (as required) | uint       | integer           | int       |
| fixed64     | Always eight bytes. More efficient than uint64 if values are often greater than 256.                                                            | uint64   | long[2]             | int/long[4]                          | uint64  | Bignum                         | ulong      | integer/string[6] | Int64     |
| sfixed32    | Always four bytes.                                                                                                                              | int32    | int                 | int                                  | int32   | Fixnum or Bignum (as required) | int        | integer           | int       |
| sfixed64    | Always eight bytes.                                                                                                                             | int64    | long                | int/long[4]                          | int64   | Bignum                         | long       | integer/string[6] | Int64     |
| bool        |                                                                                                                                                 | bool     | boolean             | bool                                 | bool    | TrueClass/FalseClass           | bool       | boolean           | bool      |
| string      | A string must always contain UTF-8 encoded or 7-bit ASCII text, and cannot be longer than 232.                                                  | string   | String              | str/unicode[5]                       | string  | String (UTF-8)                 | string     | string            | String    |
| bytes       | May contain any arbitrary sequence of bytes no longer than 232.                                                                                 | string   | ByteString          | str (Python 2)  <br>bytes (Python 3) | []byte  | String (ASCII-8BIT)            | ByteString | string            | List      |

## 默认值

解析消息时，如果编码的消息不包含特定的 隐式存在元素，访问解析后的相应字段 object 返回该字段的默认值。这些默认值为 特定于类型：

- 对于字符串，默认值为空字符串。
- 对于字节，默认值为空字节。
- 对于 bools，默认值为 false。
- 对于数值类型，默认值为零。
- 对于枚举，默认值**是第一个定义的枚举值**，该值必须 为 0。
- 对于消息字段，未设置该字段。它的确切值是 依赖于语言。有关详细信息，请参阅[生成的代码指南](https://protobuf.dev/reference/)。

重复字段的默认值为空（通常为 适当的语言）。

请注意，对于标量消息字段，一旦解析了消息，就无法 告知字段是否显式设置为默认值（例如 无论布尔值是设置为 ） 还是根本不设置：您应该承担 在定义消息类型时要牢记这一点。例如，不要有布尔值 当您不想要时，它会打开某些行为 默认情况下也会发生的行为。另请注意，如果标量**消息字段设置为**默认值，则不会在网络上序列化该值。如果 float 或 double 值设置为 +0 它不会被序列化，但 -0 是 被认为是不同的，将被序列化。`false` `false`

## 基本语法

message 就类似于 java 中的 对象

```proto3
message SearchInfo{
  int32 num = 1; // 数字
  string uuid = 2; // 字符串
  Sex sex = 3; // 枚举
  PosInfo baseInfo = 4; // 对象
  repeated Hobby hobbies = 5; // 集合
}

message PosInfo {
  string address = 1;
  string desc = 2;
}

message Hobby {
  string name = 1;
  string desc = 2;
}

enum Sex {
  BOY = 0; // 第一个枚举必须为 0
  GIRL = 1;
}
```

## proto 文件的编译

Consumer和provider之间是通过网络进行通信的，两者有可能不同的语言编写的。那两者通信的时候都得将数据转换为ProtoBuf这种中间格式来进行通讯。这样就做到了跨语言的数据传输。一旦应用了ProtoBuf这种序列化方式。

ProtoBuf作为数据传输的中间格式，一旦应用了ProtoBuf作为序列化方式的。IDL是ProtoBuf作为中间的格式，是ProtoBuf的特有的预发，里边都称之为Message。这里边涉及到一个编译的问题，需要使用ProtoC进行编译，使用ProtoC进行编译的话，有两种使用方式。1是直接本地安装protoC，另外一种是使用maven插件的方式。我们maven插件即可。

### *.proto 文件在项目中的位置

![[dubbo3-02a.proto 文件在Java项目中的位置.png]]

### pom 文件

```
<dubbo.version>3.2.11</dubbo.version>  
<grpc.version>1.58.0</grpc.version>  
<protoc.version>3.19.1</protoc.version>  
<maven.compiler.version>3.8.1</maven.compiler.version>

<dependencies>
  <dependency>  
    <groupId>com.google.protobuf</groupId>  
    <artifactId>protobuf-java</artifactId>
    <version>${protoc.version}</version>
  </dependency>  
  <dependency>  
    <groupId>com.google.protobuf</groupId>  
    <artifactId>protobuf-java-util</artifactId>
    <version>${protoc.version}</version>
  </dependency>  
</dependencies>

<build>  
  <extensions>  
    <extension>  
      <groupId>kr.motd.maven</groupId>  
      <artifactId>os-maven-plugin</artifactId>  
      <version>1.6.1</version>  
    </extension>  
  </extensions>  
  <plugins>  
    <plugin>  
      <groupId>org.apache.maven.plugins</groupId>  
      <artifactId>maven-compiler-plugin</artifactId>  
      <version>${maven.compiler.version}</version>  
      <configuration>  
        <source>1.8</source>  
        <target>1.8</target>  
        <encoding>UTF-8</encoding>  
      </configuration>  
    </plugin>  
    <plugin>  
      <groupId>org.xolstice.maven.plugins</groupId>  
      <artifactId>protobuf-maven-plugin</artifactId>  
      <version>0.6.1</version>  
      <configuration>  
        <protocArtifact>com.google.protobuf:protoc:${protoc.version}:exe:${os.detected.classifier}  
        </protocArtifact>  
        <pluginId>grpc-java</pluginId>  
        <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}  
        </pluginArtifact>  
        <protocPlugins>  
          <protocPlugin>  
            <id>dubbo</id>  
            <groupId>org.apache.dubbo</groupId>  
            <artifactId>dubbo-<compiler/artifactId>  
            <version>${dubbo.version}</version>  
            <mainClass>org.apache.dubbo.gen.tri.Dubbo3TripleGenerator</mainClass>  
          </protocPlugin>  
        </protocPlugins>  
      </configuration>  
      <executions>  
        <execution>  
          <goals>  
            <goal>compile</goal>  
            <goal>test-compile</goal>  
            <goal>compile-custom</goal>  
            <goal>test-compile-custom</goal>  
          </goals>  
        </execution>  
      </executions>  
    </plugin>  
  </plugins>  
</build>
```

### 编译

```
mvn compile
```

编译成果

![[dubbo3-02a.proto maven 编译成功.png]]

### 测试

```java
package org.example.dubbo.proto;  
  
import com.google.protobuf.InvalidProtocolBufferException;  
import java.util.UUID;  
import org.assertj.core.util.Lists;  
import org.junit.jupiter.api.Test;  
  
public class SearchInfoTest {  
  
    @Test  
    public void proto() throws InvalidProtocolBufferException {  
  
       SearchInfo clientSearchInfo = SearchInfo  
          .newBuilder()  
          .setNum(100)  
          .setUuid(UUID.randomUUID().toString())  
          .setSex(Sex.BOY)  
          .setBaseInfo(PosInfo.newBuilder().setAddress("address").setDesc("desc").build())  
          .addAllHobbies(Lists.newArrayList(  
             Hobby.newBuilder().setName("hobby.name1").setDesc("hobby.desc1").build(),  
             Hobby.newBuilder().setName("hobby.name2").build()  
          )).build();  
  
       System.out.println("模拟客户端构造数据:\n" + clientSearchInfo);  
  
       byte[] bytes = clientSearchInfo.toByteArray();  
  
       SearchInfo serverSearchInfo = SearchInfo.parseFrom(bytes);  
  
       System.out.println("---客户端数据发送数据，发送数据的形式不限于是tcp，udp，mq等传输字节组的形式---\n");  
  
       System.out.println("模拟服务端解析数据:\n" + serverSearchInfo);  
    }  
  
}
```

测试结果

```
模拟客户端构造数据:
num: 100
uuid: "b7a753de-0f89-462c-99d3-1709f4859cb9"
baseInfo {
  address: "address"
  desc: "desc"
}
hobbies {
  name: "hobby.name1"
  desc: "hobby.desc1"
}
hobbies {
  name: "hobby.name2"
}

--- 客户端数据发送数据，发送数据的形式不限于是tcp，udp，mq等传输字节组的形式 ---

模拟服务端解析数据:
num: 100
uuid: "b7a753de-0f89-462c-99d3-1709f4859cb9"
baseInfo {
  address: "address"
  desc: "desc"
}
hobbies {
  name: "hobby.name1"
  desc: "hobby.desc1"
}
hobbies {
  name: "hobby.name2"
}
```
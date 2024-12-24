---
{}
---


## 项目结构

```
- dubbo-examples
  - consumer-service
  - provider-interface
  - provider-service
```

## pom 文件

### dubbo-examples 的 pom.xml

```
<dependencyManagement>  
  <dependencies>  
    <dependency>  
      <groupId>org.springframework.boot</groupId>  
      <artifactId>spring-boot-dependencies</artifactId>  
      <version>${spring.boot.version}</version>  
      <type>pom</type>  
      <scope>import</scope>  
    </dependency>  
    <dependency>  
      <groupId>org.springframework.cloud</groupId>  
      <artifactId>spring-cloud-dependencies</artifactId>  
      <version>${spring.cloud.version}</version>  
      <type>pom</type>  
      <scope>import</scope>  
    </dependency>  
    <dependency>  
      <groupId>com.alibaba.cloud</groupId>  
      <artifactId>spring-cloud-alibaba-dependencies</artifactId>  
      <version>${spring.cloud.alibaba.version}</version>  
      <type>pom</type>  
      <scope>import</scope>  
    </dependency>  
    <dependency>  
      <groupId>org.apache.dubbo</groupId>  
      <artifactId>dubbo-bom</artifactId>  
      <version>${dubbo.version}</version>  
      <type>pom</type>  
      <scope>import</scope>  
    </dependency>  
    <dependency>  
      <groupId>org.example</groupId>  
      <artifactId>provider-interface</artifactId>  
      <version>${project.version}</version>  
    </dependency>  
    <dependency>  
      <groupId>com.google.guava</groupId>  
      <artifactId>guava</artifactId>  
      <version>${guava.version}</version>  
    </dependency>  
    <dependency>  
      <groupId>org.apache.commons</groupId>  
      <artifactId>commons-lang3</artifactId>  
      <version>${commons-lang3.version}</version>  
    </dependency>  
  </dependencies>  
</dependencyManagement>
```

### consumer 和 provider 的 pom.xml

两者的 pom.xml 是一样的

```
<dependencies>  
  <dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-web</artifactId>  
  </dependency>  
  <!-- Registry 注册中心相关 -->  
  <dependency>  
    <groupId>com.alibaba.cloud</groupId>  
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>  
  </dependency>  
  <!-- Dubbo -->  
  <dependency>  
    <groupId>org.apache.dubbo</groupId>  
    <artifactId>dubbo-spring-boot-starter</artifactId>  
  </dependency>  
  <dependency>  
    <groupId>org.apache.dubbo</groupId>  
    <artifactId>dubbo-registry-nacos</artifactId>  
  </dependency>  
  <dependency>  
    <groupId>org.example</groupId>  
    <artifactId>provider-interface</artifactId>  
  </dependency>  
  <dependency>  
    <groupId>com.google.guava</groupId>  
    <artifactId>guava</artifactId>  
  </dependency>  
  <dependency>  
    <groupId>org.apache.commons</groupId>  
    <artifactId>commons-lang3</artifactId>  
  </dependency>  
</dependencies>
```

## 代码

### provider-interface

provider-service 暴露的 API 接口定义

```java
package org.example.dubbo.service;  
  
public interface ProvideService {  
  
    String hello(String name);  
}
```

### provider-service

#### ProviderApp

```java
@EnableDiscoveryClient  
@EnableDubbo  
@SpringBootApplication  
public class ProviderApp {  
  
    public static void main(String[] args) {  
       SpringApplication.run(ProviderApp.class, args);  
    }  
  
}
```

#### ProviderServiceImpl - dubbo 接口实现

```java
@Component  
@DubboService  // dubbo 接口实现
public class ProviderServiceImpl implements ProvideService {  
  
    @Override  
    public String hello(String name) {  
       return "hello, " + name + "!";  
    }  
  
}
```

#### 配置文件

```yaml
server:  
  port: 8080  
spring:  
  application:  
    name: dubbo-provider  
  cloud:  
    nacos:  
      server-addr: localhost:8848  
      discovery:  
        namespace: local # 命名空间  
dubbo:  
  application:  
    id: ${spring.application.name}  
    name: ${spring.application.name}  
  protocol:  
    name: dubbo  
    port: -1  
  registry:  
    # 使用了 nacos 作为注册中心
    address: nacos://localhost:8848  
    check: false  
  scan:  
    base-packages: org.example.dubbo.service
```

### consumer -service

#### web 

```java
package org.example.dubbo.web;  
  
import org.apache.dubbo.config.annotation.DubboReference;  
import org.example.dubbo.service.ProvideService;  
import org.springframework.web.bind.annotation.GetMapping;  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.RequestParam;  
import org.springframework.web.bind.annotation.RestController;  
  
@RestController  
@RequestMapping("/consumer")  
public class ConsumerApi {  
  
    @DubboReference(check = false)  
    private ProvideService providerService;  
  
    @GetMapping("/hello")  
    public String hello(@RequestParam String name) {  
       return providerService.hello(name);  
    }  
  
}
```

#### 配置文件

```yaml
server:  
  port: 8081  
spring:  
  application:  
    name: dubbo-consumer  
  cloud:  
    nacos:  
      server-addr: localhost:8848  
      discovery:  
        namespace: local  
dubbo:  
  application:  
    name: ${spring.application.name}  
    logger: slf4j  
  registry:  
    # 使用了 nacos 作为注册中心
    address: nacos://localhost:8848  
    port: -1  
    check: false
```

### 启动 和 结果演示

启动顺序: nacos, provider-service, consumer-service

```
GET http://localhost:8081/consumer/hello?name=dubbo

HTTP/1.1 200 
Content-Type: text/plain;charset=UTF-8
Content-Length: 13
Date: Mon, 22 Apr 2024 07:48:06 GMT
Keep-Alive: timeout=60
Connection: keep-alive

hello, dubbo!

Response code: 200; Time: 188ms (188 ms); Content length: 13 bytes (13 B)
```


## 总结

总的来说还是很简单的,主要也是参考的[官网]([3 - 基于 Spring Boot Starter 开发微服务应用 | Apache Dubbo](https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/quick-start/spring-boot/));
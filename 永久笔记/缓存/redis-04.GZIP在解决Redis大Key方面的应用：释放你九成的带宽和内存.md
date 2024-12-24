---
title: GZIP在解决Redis大Key方面的应用：释放你九成的带宽和内存
status: Done
转载: https://mp.weixin.qq.com/s/SydFf5KmcqOrYIX7gTzB2g
Tags:
  - redis
---

# 引言

目前主流HTTP协议接口都是使用JSON格式做数据交换的，JSON数据格式有着结构简单、可读性高、跨平台，易解析等优点，同时也存在着冗余数据会占用非常多的储存空间的问题，这大大增加了JSON格式数据在存储、传输过程中的性能消耗。所以对JSON格式数据压缩后再传输、存储就变的非常的有价值，如对JSON格式数据使用GZIP压缩算法可以实现90%左右的压缩率，更小的空间可以节省存储成本和降低传输带宽成本，本文介绍GZIP压缩算法在优化Redis使用大KEY字段中的应用，通过简单压缩可以节省88%的内存空间和带宽资源。

## HTTP协议开启GZIP

HTTP协议标准中是直接支持GZIP压缩算法的，通过响应头`Content-Encoding: gzip`来表明响应内容使用了GZIP压缩，当客户端收到数据后会使用GZIP算法对Body内容进行解压。

> ★
> 
> RFC 1952 - IETF（互联网工程任务组）标准化的Gzip文件格式规范,

> ★
> 
> RFC 2616 - HTTP 1.1 协议规范，其中包括对 Content-Encoding 头的定义

在Nginx中可以通过 `gzip on`开启GZIP压缩功能：

```
gzip on;   gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;   
```

在Springboot中可以通过`server.compression.enabled`开启GZIP压缩功能：

```
server:     
  port: 80     
    compression:       
      enabled: true       
      mime-types:  application/javascript,text/css,application/json,application/xml,text/html,text/xml,text/plain       
      min-response-size: 2KB   
```

- enabled，开启或关闭
- mime-types，压缩的数据类型
- min-response-size，最小压缩大小

## 测试GZIP

为了测试开启GZIP前后的对比效果我们写一个简单的接口：

```java
@GetMapping("/list")  
public ResponseEntity<ApiResult> list() {  
    return renderOk(getData());  
}
```

我们返回1000条JSON格式的用户信息：

```java
private List<UserVo> getData() {  
    return IntStream.range(1, 1000).mapToObj(x -> new UserVo(x,x+"+email@q63.com",x+"_公众号",x+"_赵侠客")).collect(Collectors.toList());  
}  
@Data  
@AllArgsConstructor  
public class UserVo {  
    private Integer id;  
    private String username;  
    private String email;  
    private String trueName;  
}
```

在未开启GZIP前接口返回数据的大小是92.8KB， `Content-Encoding`为空，在开启GZIP后接口返回的数据大小为11.5KB，`Content-Encoding`为gzip，接口返回数量降低了88%。![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RBARszDJ69jmciapK1Jmoos5UsS8IicKPBb1E3ia0bxgXiaqpfc3eJiau5x9qk8PAblKnmjlspOowX6zOEUuYMCNICQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当然我们也可以在接口中通过手动添加`content-encoding`响应头,然后通过手动调用`GZIPOutputStream`对返回数据进行GZIP压缩：

```java
@GetMapping("/gzip")  
public void gzip(HttpServletResponse response) throws IOException {  
    response.setContentType("application/json;charset=utf-8");  
    response.setHeader("content-encoding", "gzip");  
    try (GZIPOutputStream gzipOutputStream = new GZIPOutputStream(response.getOutputStream())) {  
        IOUtils.write(JsonUtils.toJson(getData()), gzipOutputStream);  
    }  
}
```

## Redis缓存压缩

为了增加接口的响应速度我们通常会使用Redis当缓存，基本逻辑是先查Redis有没有数据如果有直接返回，如果没有会查数据库，然后再存入Redis，以下是一个简单的使用Redis当缓存的接口：

```java
@Resource  
private RedissonClient redissonClient;  
public static final String REDIS_KEY = "REDIS_KEY";  
  
@GetMapping("/redis")  
public void redis(HttpServletResponse response) throws IOException {  
    RBucket<String> bucket = redissonClient.getBucket(REDIS_KEY);  
    String data = bucket.get();  
    if (data == null) {  
         data=JsonUtils.toJson(getData());  
        redissonClient.getBucket(REDIS_KEY).set(data,100L, TimeUnit.SECONDS);  
    }  
    response.setContentType("application/json");  
    IOUtils.write(data, response.getOutputStream());  
}
```

我们分析一下这样个接口的基本数据流：

- 第一次从数据库服务器查出92.8KB的数据传输到WEB服务器中
- 将92.8KB的数据从WEB服务器传输到Redis服务器中
- 后面如果命中缓存将92.8KB数据从Redis服务器传输到WEB服务器
- 最后将92.8KB数据从WEB服务器返回给用户浏览器

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RBARszDJ69jmciapK1Jmoos5UsS8IicKPB43FqPop6YXsqRBuLzSeW4lF6QzeibAozjnVpt1G4GbT1JU6xBFJ8kDw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

使用Redis当缓存加速接口

使用ZIP优化Redis缓存：

```java
public static final String GZIP_REDIS_KEY = "GZIP_REDIS_KEY";  
  
@GetMapping("/gzipRedis")  
public void gzipRedis(HttpServletResponse response) throws IOException {  
    RBucket<byte[]> bucket = redissonClient.getBucket(GZIP_REDIS_KEY);  
    byte[] data = bucket.get();  
    if (data == null) {  
        String json=JsonUtils.toJson(getData());  
        try (ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();  
             GZIPOutputStream gzipOutputStream = new GZIPOutputStream(byteArrayOutputStream)) {  
            IOUtils.write(json, gzipOutputStream, String.valueOf(StandardCharsets.UTF_8));  
            gzipOutputStream.finish();  
            data= byteArrayOutputStream.toByteArray();  
            redissonClient.getBucket(GZIP_REDIS_KEY).set(data,100L, TimeUnit.SECONDS);  
        }  
    }  
    response.setContentType("application/json");  
    response.setHeader("content-encoding", "gzip");  
    IOUtils.write(data, response.getOutputStream());  
}
```

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RBARszDJ69jmciapK1Jmoos5UsS8IicKPBgzhecVhKqW3H6HjReYBjtNXkrDliciaicjmLnnEEfECCdCwZibVy3bPiccw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

使用GZIP压缩后的缓存接口

我们再分析一下以上使用GZIP压缩后的数据传输：

- 第一次从数据库服务器查出92.8KB的数据传输到WEB服务器中
- 将11.5KB的GZIP数据从WEB服务器传输到Redis服务器中
- 后面命中缓存将11.5KB数据从Redis服务器传输到WEB服务器
- 最后将11.KB数据从WEB服务器返回给用户浏览器

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RBARszDJ69jmciapK1Jmoos5UsS8IicKPBfaibjibiculTDjcbFU64iaV1c2q10paX4dibUPktehLLWZu8LYBJc904s2Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

GZIP压缩后的Redis缓存

单次接口请求好像感觉不到这个 GZIP压缩带来的好处，接下来我们压测一下看看会不会有差距。

## 压力测试

压测可以使用ab (Apache Benchmark) 工具，ab工具是 Apache HTTP server 的一部分，在 macOS使用Homebrew包管理器可以快速安装上ab :

`brew install httpd   ab -V   ab -n 100 -c 10 http://localhost/list   `

其中：
- -n 100 表示总共请求 100 次。
- -c 10  表示并发 10 个请求。

**未压缩走Redis压缩结果:**

```
ab -n 100000 -c 10 http://localhost/redis  
  
Finished 100000 requests  
Document Length:        92476 bytes  
Concurrency Level:      10  
Time taken for tests:   194.917 seconds  
Complete requests:      100000  
Failed requests:        0  
Total transferred:      9258100000 bytes  
HTML transferred:       9247600000 bytes  
Requests per second:    513.04 [#/sec] (mean)  
Time per request:       19.492 [ms] (mean)  
Time per request:       1.949 [ms] (mean, across all concurrent requests)  
Transfer rate:          46384.34 [Kbytes/sec] received  
  
Connection Times (ms)  
              min  mean[+/-sd] median   max  
Connect:        0    8 249.5      0   19514  
Processing:     4   12  19.8     10     754  
Waiting:        4   11  19.8     10     754  
Total:          4   19 250.4     10   19525  
Percentage of the requests served within a certain time (ms)  
  50%     10  
  66%     11  
  75%     11  
  80%     12  
  90%     12  
  95%     15  
  98%     27  
  99%    134  
 100%  19525 (longest request)
```

**使用GZIP压缩后走Redis缓存压测结果：**

```
ab -n 100000 -c 10 http://localhost/gzipRedis  
  
Finished 100000 requests  
Document Length:        11091 bytes  
Concurrency Level:      10  
Time taken for tests:   194.927 seconds  
Complete requests:      100000  
Failed requests:        0  
Total transferred:      1122000000 bytes  
HTML transferred:       1109100000 bytes  
Requests per second:    513.01 [#/sec] (mean)  
Time per request:       19.493 [ms] (mean)  
Time per request:       1.949 [ms] (mean, across all concurrent requests)  
Transfer rate:          5621.09 [Kbytes/sec] received  
  
Connection Times (ms)  
              min  mean[+/-sd] median   max  
Connect:        0   12 410.4      0   19608  
Processing:     3    7  20.0      4     802  
Waiting:        3    7  19.9      4     801  
Total:          3   19 410.9      4   19613  
  
Percentage of the requests served within a certain time (ms)  
  50%      4  
  66%      9  
  75%      9  
  80%      9  
  90%     10  
  95%     10  
  98%     11  
  99%     19  
 100%  19613 (longest request)
```

# 总结

对比使用GZIP压缩我们可以得出以下几点：

- 测试中10万请求在194S完成，缓存时间是100S，服务器端只做了二次查数据库和GZIP压缩然后存数Redis
- 两次GZIP和之后的数据传输消耗资源可以忽略不计
- 未压缩10万请求从Redis传输了8.6GB数据到WEB服务器，又从WEB服务器传输8.6GB给用户浏览器，
- 压缩10万请求从Redis传输了1GB数据到WEB服务器，又从WEB服务器传输1GB给用户浏览器，节省数据传输15.2GB，节省率88%
- 未压缩数据传输速度达到45M/S，压缩后5.4M/S，节省带宽88%
- 如果Redis中大JSON都使用GZIP压缩理论上可以节省Redis内存达到88%
- 因为直接使用gzip返回，所有解压计算在用户浏览器端完成，不消耗服务器CPU资源

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RBARszDJ69jmciapK1Jmoos5UsS8IicKPBfBWJqbUarRT53Uriaj4ibfVNE1kLu7WJdW1MWU93yDQhJZUErY7frONQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

请求10万次数据传输流程

综合上所述如里你的Redis缓存中存在大量的大Key，可能先达到瓶颈的不是Redis的读写性能，很可能是你的带宽，此时只需要简单的使用GZIP压缩就能你给不仅节省88%的Redis内存空间还大大减少了数据的传输量和节省了带宽资源，而且还能使用的C端用户的资源来解压，这个ROI是非常高的。
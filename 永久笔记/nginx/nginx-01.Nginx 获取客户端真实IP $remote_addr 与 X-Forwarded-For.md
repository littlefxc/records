---
modified: 2024-11-22 15:41:02
created: 2024-11-22 15:33:44
---
> 转载自 [Nginx 获取客户端真实IP $remote_addr与X-Forwarded-For_nginx x-real-ip-CSDN博客](https://blog.csdn.net/qq_34556414/article/details/106634895)

## nginx 配置

首先，一个请求肯定是可以分为请求头和请求体的，而我们客户端的IP地址信息一般都是存储在请求头里的。如果你的服务器有用Nginx做负载均衡的话，你需要在你的location里面配置`X-Real-IP`和`X-Forwarded-For`请求头：

```
location ^~ /your-service/ {
    proxy_set_header        X-Real-IP       $remote_addr;
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://localhost:60000/your-service/;
}
```

## X-Real-IP

在《实战nginx》中，有这么一句话：

> 经过反向代理后，由于在客户端和web服务器之间增加了中间层，因此web服务器无法直接拿到客户端的ip，通过$remote_addr变量拿到的将是反向代理服务器的ip地址。

这句话的意思是说，当你使用了nginx反向服务器后，在web端使用request.getRemoteAddr()（本质上就是获取$remote_addr），取得的是nginx的地址，即$remote_addr变量中封装的是nginx的地址，当然是没法获得用户的真实ip的。但是，nginx是可以获得用户的真实ip的，也就是说nginx使用$remote_addr变量时获得的是用户的真实ip，如果我们想要在web端获得用户的真实ip，就必须在nginx里作一个赋值操作，即我在上面的配置：

> proxy_set_header X-Real-IP $remote_addr;

$remote_addr 只能获取到与服务器本身直连的上层请求ip，所以设置 $remote_addr一般都是设置第一个代理上面;但是问题是，**有时候是通过cdn访问过来的，那么后面web服务器获取到的，永远都是cdn 的ip 而非真是用户ip,那么这个时候就要用到X-Forwarded-For 了**，这个变量的意思，其实就像是链路反追踪，从客户的真实ip为起点，穿过多层级的proxy ，最终到达web 服务器，都会记录下来，所以在获取用户真实ip的时候，一般就可以设置成，proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 这样就能获取所有的代理ip 客户ip。 

## X-Forwarded-For

X-Forwarded-For变量，这是一个squid开发的，用于识别通过HTTP代理或负载平衡器原始IP一个连接到Web服务器的客户机地址的非rfc标准，如果有做X-Forwarded-For设置的话,每次经过proxy转发都会有记录,格式就是client1,proxy1,proxy2以逗号隔开各个地址，由于它是非rfc标准，所以默认是没有的，需要强制添加。在默认情况下经过proxy转发的请求，在后端看来远程地址都是proxy端的ip 。也就是说在默认情况下我们使用request.getAttribute("X-Forwarded-For")获取不到用户的ip，如果我们想要通过这个变量获得用户的ip，我们需要自己在nginx添加配置：

> proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

意思是增加一个 $proxy_add_x_forwarded_for 到 X-Forwarded-For 里去，注意是增加，而不是覆盖，当然由于默认的X-Forwarded-For值是空的，所以我们总感觉X-Forwarded-For的值就等于 $proxy_add_x_forwarded_for的值，实际上当你搭建两台nginx在不同的ip上，并且都使用了这段配置，那你会发现在web服务器端通过request.getAttribute("X-Forwarded-For")获得的将会是客户端ip和第一台nginx的ip。

那么`$proxy_add_x_forwarded_for`又是什么？
**`$proxy_add_x_forwarded_for`变量包含客户端请求头中的`X-Forwarded-For`与`$remote_addr`两部分，他们之间用逗号分开。**

举个例子，有一个web应用，在它之前通过了两个nginx转发，`www.linuxidc.com`即用户访问该web通过两台nginx。

在第一台nginx中,使用：

```
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

现在的 $proxy_add_x_forwarded_for变量，X-Forwarded-For部分包含的是用户的真实ip， $remote_addr 部分的值是上一台nginx的ip地址，于是通过这个赋值以后现在的X-Forwarded-For的值就变成了“用户的真实ip，第一台nginx的ip”，这样就清楚了吧。

## 举个例子说明 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

```
192.168.179.99->192.168.179.100->192.168.179.101->192.168.179.102（102为最后的服务端）
 
192.168.179.99配置
      location /{
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_pass http://192.168.179.100;
     }
 
 
192.168.179.100配置
      location /{
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_pass http://192.168.179.101;
     }
 
 
192.168.179.101配置
      location /{
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_pass http://192.168.179.102;
     }
 
 
192.168.179.102配置
在日志中设置打印$http_x_forwarded_for，进行观察
 log_format  main  '$http_x_forwarded_for|$http_x_real_ip|$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
```

```
(1)客户端使用浏览器去访问192.168.179.99服务端192.168.179.102日志如下，可以看到获取到客户端的IP为192.168.179.4
192.168.179.4, 192.168.179.99, 192.168.179.100|-|192.168.179.101 - - [26/Apr/2020:10:57:22 +0800] "GET / HTTP/1.0" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.122 Safari/537.36" "192.168.179.4, 192.168.179.99, 192.168.179.100"
 
（2）客户端192.168.179.103去访问192.168.179.99服务端192.168.179.102日志如下，可以看到获取到客户端的IP为192.168.179.103
192.168.179.103, 192.168.179.99, 192.168.179.100|-|192.168.179.101 - - [26/Apr/2020:10:57:32 +0800] "GET / HTTP/1.0" 200 4833 "-" "curl/7.29.0" "192.168.179.103, 192.168.179.99, 192.168.179.100"
```
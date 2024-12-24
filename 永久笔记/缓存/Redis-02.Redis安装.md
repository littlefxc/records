---
title: Redis安装
status: Done
Tags:
  - redis
---

### **下载**

官网：[https://redis.io/download](https://redis.io/download)

选择下载稳定版本，不稳定版本可以尝鲜，但是不推荐在生产使用。

### **上传至linux**

![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled.png](../../attachments/Redis-02.Redis安装/Untitled.png)

### **安装 Redis**

1. 解压redis：
    
    **tar -zxvf redis-5.0.5.tar.gz**
    
    得到：
    
    ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%201.png](../../attachments/Redis-02.Redis安装/Untitled%201.png)
    
2. 安装gcc编译环境，如果已经安装过了，那么就是 nothing to do
    
    **yum install gcc-c++**
    
3. 进入到 **redis-5.0.5** 目录，进行安装：
    
    **make && make install**
    
    执行完毕后安装成功
    
4. 配置redis，在utils下，拷贝**redis_init_script**到**/etc/init.d**目录，目的要把redis作为开机自启动创建 /usr/local/redis，用于存放配置文件
    
    ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%202.png](../../attachments/Redis-02.Redis安装/Untitled%202.png)
    
5. 创建 /usr/local/redis，用于存放配置文件
    
    ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%203.png](../../attachments/Redis-02.Redis安装/Untitled%203.png)
    
6. 拷贝redis配置文件：
    
    ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%204.png](../../attachments/Redis-02.Redis安装/Untitled%204.png)
    
7. 拷贝到 /usr/local/redis 下
    
    ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%205.png](../../attachments/Redis-02.Redis安装/Untitled%205.png)
    
8. 修改redis.conf这个核心配置文件
    1. 修改 daemonize no -> daemonize yes，目的是为了让redis启动在linux后台运行
        
        ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%206.png](../../attachments/Redis-02.Redis安装/Untitled%206.png)
        
    2. 修改redis的工作目录：
        
        ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%207.png](../../attachments/Redis-02.Redis安装/Untitled%207.png)
        
    3. 建议修改为： /usr/local/redis/working，名称随意
    4. 修改如下内容，绑定IP改为 0.0.0.0 ，代表可以让远程连接，不收ip限制
        
        ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%208.png](../../attachments/Redis-02.Redis安装/Untitled%208.png)
        
    5. 最关键的是密码，默认是没有的，一定要设置
        
        ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%209.png](../../attachments/Redis-02.Redis安装/Untitled%209.png)
        
    6. 并且修改redis核心配置文件名称为：6379.conf
    7. 为redis启动脚本添加执行权限，随后运行启动redis:
        
        ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%2010.png](../../attachments/Redis-02.Redis安装/Untitled%2010.png)
        
    8. 检查redis进程：
        
        ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%2011.png](../../attachments/Redis-02.Redis安装/Untitled%2011.png)
        
        到此redis安装并且启动成功！
        
    9. 设置redis开机自启动，修改 redis_init_script，添加如下内容
        
        ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%2012.png](../../attachments/Redis-02.Redis安装/Untitled%2012.png)
        
        随后执行如下操作：
        
        **chkconfig redis_init_script on**
        
        重启服务器(虚拟机)后，再看进程：
        
        ![%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E9%AB%98%E5%8F%AF%E7%94%A8Redis%E9%9B%86%E7%BE%A4%E4%B9%8B%E5%AE%89%E8%A3%85%20Redis%EF%BC%88%E5%85%AD%EF%BC%89%20f6f8bd84e6f94e9d9653ee1f7e55a1c6/Untitled%2013.png](../../attachments/Redis-02.Redis安装/Untitled%2013.png)
        
        OK，没毛病
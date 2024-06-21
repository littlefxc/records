
---
title: G1垃圾收集器jvm参数设置
status: Done
tags:
  - Java
  - JVM
createDate: 2023-06-17T11:58:09+08:00
---

---

```
#堆内存最大最小值为4g，新生代内存2g
-Xms4g -Xmx4g -Xmn2g 
#元空间128m，最大320m
-XX:MetaspaceSize=128m 
-XX:MaxMetaspaceSize=320m 
#开启远程debug
-Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n 

#使用G1垃圾收集器，在低延迟和高吞吐间寻找平衡，可以调整最大停止时间，设置新生代大小来提高吞吐量，让出cpu资源
-XX:+UseG1GC
#设置最大暂停时间，默认200ms
-XX:MaxGCPauseMillis=200
#指定Region大小，必须是2次幂
-XX:G1HeapRegionSize=2m
#反复执行混合回收8次，每次回收受MaxGCPauseMillis的影响可能一次性回收不了所有垃圾，增加次数才能回收的更彻底
-XX:G1MixedGCCountTarget=8
# 混合回收整理出来的空闲空间占heap的10时，结果老年代的回收，默认5
-XX:G1HeapWastePercent=10
#设置新生代大小，最大60%，默认5%
-XX:G1NewSizePercent=10 -XX:G1MaxNewSizePercent=50

-XX:SurvivorRatio=8 
#在控制台输出GC情况
-verbose:gc 
#gc日志打印到执行日志文件
-Xloggc:./logs/job_execute_gc.log
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime 
#可以生成更详细的Survivor空间占用日志
-XX:+PrintAdaptiveSizePolicy 
#jdk 1.6开始，默认server模式下开启了这个参数，意为当jvm检测到程序在重复抛一个异常，在执行若干次后会将异常吞掉
-XX:-OmitStackTraceInFastThrow 
-XX:-UseLargePages
#指定加载配置文件
--spring.config.location=classpath:/,classpath:/config/,file:./,file:./config/,file:/home/mall-job/conf/


#---当前分布式任务调度采用jvm参数，-Xmn2g，-XX:MaxGCPauseMillis=400调整新生代内存大小，增大暂停时间提高吞吐量---------------------------
-Xms4g -Xmx4g -Xmn2g -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n -XX:+UseG1GC -XX:MaxGCPauseMillis=400 -XX:G1HeapRegionSize=2m -XX:G1MixedGCCountTarget=8 -XX:G1MixedGCCountTarget=8 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime -Xloggc:/logs/execute/mall-job-execute-gc.log
```
---
title: Oracle查看当前连接数
tags:
  - Oracle
status: Done
createDate: 2023-04-14
---

**1、查看当前的数据库连接数**

```sql
select count(*) from v$process ;
```

**2、数据库允许的最大连接数**

```sql
select value from v$parameter where name ='processes';
```

**3、修改数据库最大连接数**

```sql
alter system set processes = 300 scope = spfile;
```

**4、关闭/重启数据库**

```sql
shutdown immediate; --关闭数据库 
startup; --重启数据库
```

**5、查看当前有哪些用户正在使用数据**

```sql
select osuser, a.username, cpu_time/executions/1000000||'s', b.sql_text, machine
from v$session a, v$sqlarea b
where a.sql_address =b.address 
order by cpu_time/executions desc;  
```

**6、 --当前的session连接数**

```sql
select count(*) from v$session
```

**7、当前并发连接数**

```sql
 select count(*) from v$session where status='ACTIVE';　
```

**8、** **v$process**：这个视图提供的信息，都是oracle服务进程的信息，没有客户端程序相关的信息 服务进程分两类，一是后台的，一是dedicate/shared erver

```sql
pid, serial#       这是oracle分配的PID   
spid               这才是操作系统的
pid program        这是服务进程对应的操作系统进程名
```

**9、**v$session：****

> 这个视图主要提供的是一个数据库connect的信息， 主要是client端的信息，比如以下字段：

```sql
machine   在哪台机器上
terminal  使用什么终端
osuser    操作系统用户是谁
program   通过什么客户端程序，比如TOAD
process   操作系统分配给TOAD的进程号
logon_time  在什么时间
username    以什么oracle的帐号登录
command     执行了什么类型的SQL命令
sql_hash_value  SQL语句信息

有一些是server端的信息：
paddr   即v$process中的server进程的addr
server  服务器是dedicate/shared
```
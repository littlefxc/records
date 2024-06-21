---
title: logstash同步数据库配置
tags:
  - elk
  - logstash
status: Done
createDate: 2023-04-14
---

[上一篇文档](https://blog.csdn.net/Little_fxc/article/details/112644481)
[下一篇文档](https://blog.csdn.net/Little_fxc/article/details/112644609)

# logstash同步数据库配置

1. 上传并解压logstash，路径为 `/usr/local/logstash`
2. 创建文件名：logstash-db-sync.conf，后缀为`conf`，文件名随意，位置也随意，为方便起见，路径为`/usr/local/logstash/sync`
3. 把数据库驱动拷贝至`/usr/local/logstash/sync`下
4. 配置内容如下：

	  ```yaml
	  input {
	      jdbc {
	          # 设置 MySql/MariaDB 数据库url以及数据库名称
	          jdbc_connection_string => "jdbc:mysql://localhost:3306/foodie-shop-dev?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true"
	          # 用户名和密码
	          jdbc_user => "root"
	          jdbc_password => "root"
	          # 数据库驱动所在位置，可以是绝对路径或者相对路径
	          jdbc_driver_library => "/usr/local/logstash/sync/mysql-connector-java-5.1.41.jar"
	          # 驱动类名
	          jdbc_driver_class => "com.mysql.jdbc.Driver"
	          # 开启分页
	          jdbc_paging_enabled => "true"
	          # 分页每页数量，可以自定义
	          jdbc_page_size => "10000"
	          # 执行的sql文件路径
	          statement_filepath => "/usr/local/logstash/sync/foodie-items.sql"
	          # 设置定时任务间隔  含义：分、时、天、月、年，全部为*默认含义为每分钟跑一次任务
	          schedule => "* * * * *"
	          # 索引类型
	          type => "_doc"
	          # 是否开启记录上次追踪的结果，也就是上次更新的时间，这个会记录到 last_run_metadata_path 的文件
	          use_column_value => true
	          # 记录上一次追踪的结果值
	          last_run_metadata_path => "/usr/local/logstash/sync/track_time"
	          # 如果 use_column_value 为true， 配置本参数，追踪的 column 名，可以是自增id或者时间
	          tracking_column => "updated_time"
	          # tracking_column 对应字段的类型
	          tracking_column_type => "timestamp"
	          # 是否清除 last_run_metadata_path 的记录，true则每次都从头开始查询所有的数据库记录
	          clean_run => false
	          # 数据库字段名称大写转小写
	          lowercase_column_names => false
	      }
	  }
	  output {
	      elasticsearch {
	          # es地址
	          hosts => ["localhost:9200"]
	          # 同步的索引名
	          index => "foodie-items"
	          # 设置_docID和数据相同。itemId与sql同步脚本中的itemId保持一致
	          document_id => "%{itemId}"
	          #document_id => "%{id}"
	      }
	      # 日志输出
	      stdout {
	          codec => json_lines
	      }
	  }
	  ```

5. sql同步脚本

	  ```sql
	  SELECT
	      i.id as itemId,
	      i.item_name as itemName,
	      i.sell_counts as sellCounts,
	      ii.url as imgUrl,
	      tempSpec.price_discount as price,
	      i.updated_time as updated_time
	  FROM
	      items i
	  LEFT JOIN
	      items_img ii
	  on
	      i.id = ii.item_id
	  LEFT JOIN
	      (SELECT item_id,MIN(price_discount) as price_discount from items_spec GROUP BY item_id) tempSpec
	  on
	      i.id = tempSpec.item_id
	  WHERE
	      ii.is_main = 1
	      and
	      i.updated_time >= :sql_last_value
	  --:sql_last_value是记录的最后的一个值
	  ```

# 启动logstatsh

```bash
./logstash -f /usr/local/logstash/sync/logstash-db-sync.conf
```

# 参考资源

[Logstash使用介绍](https://www.cnblogs.com/huhangfei/p/7605511.html)

[Logstash 参考指南（logstash.yml）](https://segmentfault.com/a/1190000016591476)
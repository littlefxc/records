---
title: logstash将mysql经纬度数据转为geo_point
status: Done
Tags:
  - elk
  - logstash
---

## 前言

目标：通过logstash同步，将mysql 中的经纬度字符串转为 es 中的 `geo_point` 类型

这种字段类型有 4 中格式（官方示例）：

第一种：

```json
{
  "name":     "Chipotle Mexican Grill",
  "location": "40.715, -74.011" 
}
```

第二种：

```json
{
  "name":     "Pala Pizza",
  "location": { 
    "lat":     40.722,
    "lon":    -73.989
  }
}
```

第三种：

```json
{
  "name":     "Mini Munchies Pizza",
  "location": [ -73.983, 40.719 ] 
}
```

以第一种 `geo_point` 为例

## 新建索引

首先创建索引

```http
PUT monitor_point_position
{
  "mappings": {
    "properties": {
      "location": {  
        "type": "geo_point"  
      }
    }
  }
}
```

## logstash 脚本

```yaml
input {  
  jdbc {  
    jdbc_driver_library => "/usr/share/logstash/output/mysql-connector-java-8.0.21.jar"  
    jdbc_driver_class => "com.mysql.jdbc.Driver"  
    jdbc_connection_string => "jdbc:mysql://localhost:3306/monitor"  
    jdbc_user => "root"  
    jdbc_password => "12345678"  
    statement => "select id,name,group_id,device_id,latitude,longitude,concat_ws(',',`latitude`, `longitude`) as location from monitor_point_position"  
    jdbc_paging_enabled => "true"  
    jdbc_page_size => "50000"  
    schedule => "*/5 * * * *"  
  }  
}  
filter {  
        # 转换经纬度坐标的字段类型  
        convert => ["latitude", "float"]  
        convert => ["longitude", "float"]  
    }  
    mutate {  
        # 这里要注意一下 一定要lat在前 lon在后，因为es geo_point的格式就是[lat, lon]  
        rename => {  
            "lat" => "[location][latitude]"  
            "lon" => "[location][longitude]"  
        }  
    }  
}  
output {  
  stdout {  
    codec => json_lines  
  }  
  elasticsearch {  
    hosts => ["172.17.0.3:9200"]  
    index => "monitor_point_position"  
    document_id => "%{id}"  
    user => "elastic"  
    password => "12345678"  
  }  
}
```

>Tips: rename 插件中，一定要lat在前，lon在后， 否则es解析经纬度坐标会报错：faild to parse field gis of type `[geo_point]`

## 启动

```sh
./logstash -f logstash-sync-mysql.conf
```

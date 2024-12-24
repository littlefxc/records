---
title: Logstash自定义模板配置中文分词
status: Done
Tags:
  - elk
  - logstash
---

# 前言

[上篇文档](https://blog.csdn.net/Little_fxc/article/details/112644581)

目前的数据同步，mappings映射会自动创建，但是分词不会，还是会使用默认的，而我们需要中文分词，这个时候就需要自定义模板功能来设置分词了。

# 查看Logstash默认模板

```jsx
GET /_template/logstash
```

# 修改模板如下

```jsx
{
    "order": 0,
    "version": 1,
    "index_patterns": ["*"],
    "settings": {
        "index": {
            "refresh_interval": "5s"
        }
    },
    "mappings": {
        "_default_": {
            "dynamic_templates": [
                {
                    "message_field": {
                        "path_match": "message",
                        "match_mapping_type": "string",
                        "mapping": {
                            "type": "text",
                            "norms": false
                        }
                    }
                },
                {
                    "string_fields": {
                        "match": "*",
                        "match_mapping_type": "string",
                        "mapping": {
                            "type": "text",
                            "norms": false,
                            "analyzer": "ik_max_word",
                            "fields": {
                                "keyword": {
                                    "type": "keyword",
                                    "ignore_above": 256
                                }
                            }
                        }
                    }
                }
            ],
            "properties": {
                "@timestamp": {
                    "type": "date"
                },
                "@version": {
                    "type": "keyword"
                },
                "geoip": {
                    "dynamic": true,
                    "properties": {
                        "ip": {
                            "type": "ip"
                        },
                        "location": {
                            "type": "geo_point"
                        },
                        "latitude": {
                            "type": "half_float"
                        },
                        "longitude": {
                            "type": "half_float"
                        }
                    }
                }
            }
        }
    },
    "aliases": {}
}
```

# 新增如下配置，用于更新模板，设置中文分词

将下列配置放到 [logstash-db-sync.conf](https://blog.csdn.net/Little_fxc/article/details/112644581) 文件 output 下的 elasticsearch 中

```jsx
# 定义模板名称
template_name => "myik"
# 模板所在位置
template => "/usr/local/logstash/sync/logstash-ik.json"
# 重写模板
template_overwrite => true
# 默认为true，false关闭logstash自动管理模板功能，如果自定义模板，则设置为false
manage_template => false
```

# 重新运行Logstash进行同步

```jsx
./logstash -f /usr/local/logstash/sync/logstash-db-sync.conf
```


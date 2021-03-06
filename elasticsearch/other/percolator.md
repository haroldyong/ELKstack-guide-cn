# percolator 接口

在运维体系中，监控和报警总是成双成对的出现。Elastic Stack 在时序统计方面的便捷，在很多时候被作为监控的一种方式在使用。那么，自然就引申出一个问题：Elastic Stack 如何做报警？

对于简单而且固定需求的模式，我们可以在 Logstash 中利用 `filter/metric` 和 `filter/ruby` 等插件做预处理，直接 `output/nagios` 或 `output/zabbix` 来报警；但是对于针对全局的、更复杂的情况，Logstash 就无能为力了。

目前比较通行的办法。有两种：

1. 对于匹配报警，采用 ES 的 Percolator 接口做响应报警；
2. 对于时序统计，采用定时任务方式，发送 ES aggs 请求，分析响应体报警。

针对报警的需求，ES 官方也在最近开发了 Watcher 商业产品，和 Shield 一样以 ES 插件形式存在。本节即稍微描述一下 Percolator 接口的用法和 Watcher 产品的思路。相信稍有编程能力的读者都可以根据自己的需求写出来类似的程序。

## Percolator 接口

Percolator 接口和我们习惯的搜索接口正好相反，它要求预先定义好 query，然后通过接口提交文档看能匹配上哪个 query。也就是说，这是一个实时的模式过滤接口。

5.0 版中，对 Percolator 功能做了大幅度改造，现在已经没有单独的接口，而是作为一种 mapping 类型存在。也就是说，我们在创建索引的时候需要预先定义。

比如我们通过 syslog 来发现硬件报错的时候，需要预先定义 mapping：

```
# curl -XPUT http://127.0.0.1:9200/syslog -d '{
    "mappings" : {
        "syslog" : {
            "properties" : {
                "message" : {
                    "type" : "text"
                },
                "severity" : {
                    "type" : "long"
                },
                "program" : {
                    "type" : "keyword"
                }
            }
        },
        "queries" : {
            "properties" : {
                "query" : {
                    "type" : "percolator"
                }
            }
        }
    }
}'
```

然后我们往 `syslog/queries` 里注册 2 条 percolator 请求规则：

```
# curl -XPUT http://127.0.0.1:9200/syslog/queries/memory -d '{
    "query" : {
        "query_string" : {
            "default_field" : "message",
            "default_operator" : "OR",
            "query" : "mem DMA segfault page allocation AND severity:>2 AND program:kernel"
        }
    }
}'
# curl -XPUT http://127.0.0.1:9200/syslog/queries/disk -d '{
    "query" : {
        "query_string" : {
            "default_field" : "message",
            "default_operator" : "OR",
            "query" : "scsi sata hdd sda AND severity:>2 AND program:kernel"
        }
    }
}'
```

然后，将标准的数据写入请求做一点改动，通过搜索接口进行：

```
# curl -XPOST http://127.0.0.1:9200/syslog/_search -d '{
    "query" : {
        "percolate" : {
            "field" : "query",
            "document_type" : "syslog",
            "document" : {
                "program" : "kernel",
                "severity" : 3,
                "message" : "swapper/0: page allocation failure: order:4, mode:0x4020"
            }
        }
    }
}'
```

得到如下结果：

```
{
  ...,
  "hits": [
     {
        "_index": "syslog",
        "_type": "queries",
        "_id": "memory",
        ...
     }
  ]
}
```

从结果可以看出来，这条 syslog 日志匹配上了 memory 异常。接下来就可以发送给报警系统了。

如果是 syslog 索引中已经有的数据，也可以重新过一遍 Percolator 查询。比如我们有一条之前已经写入到 `http://127.0.0.1:9200/syslog/cisco/1234567` 的数据，如下命令就可以把这条数据再过一次 percolate：

```
# curl -XPOST http://127.0.0.1:9200/syslog/_search -d '{
    "query" : {
        "percolate" : {
            "field" : "query",
            "document_type" : "syslog",
            "index" : "syslog",
            "type" : "cisco",
            "id" : "1234567",
        }
    }
}'
```

利用更复杂的 query DSL 做 Percolator 请求的示例，推荐阅读官网这篇 geo 定位的文章：<https://www.elastic.co/blog/using-percolator-geo-tagging>


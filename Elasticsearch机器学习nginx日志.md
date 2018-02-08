## Elasticsearch机器学习nginx日志

这篇文章介绍如何使用Elasticsearch X-Pack中的机器学习深入分析nginx日志。

### 介绍

日志分析是一个复杂的过程，分析人员很容易在大量日志中迷失。因此Elastic公司机器学习团队的目标是开发一套工具能够帮助用户深入观察分析存储在Elasticsearch中的数据。

### nginx日志

nginx日志示例

```
 "2021:0eb8:86a3:1000:0000:9b3e:0370:7334 10.225.192.17 10.2.2.121" - - [30/Dec/2016:06:47:09 +0000] "GET /test.html HTTP/1.1" 404 8571 "-" "Mozilla/5.0 (compatible; Facebot 1.0; https://developers.facebook.com/docs/sharing/webmasters/crawler)"
```

nginx日志结构

```
"$http_x_forwarded_for" $remote_addr - [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"
```

> logstash提供的grok filter可以将nginx日志解析为能够存储到Elasticsearch的json对象。

### 分析变化：网站请求数量

网站出现的系统问题常常可以通过观察请求数量的变化趋势发现。例如，网站请求数量突然极速下降，预示着网站系统出现问题。

- 登录Kibana
- 在Machine Learning中创建任务
- 选择存储nginx日志的索引
- 使用向导创建单指标分析任务
- Aggregation选择count
- Field空
- Bucket span选择10m
- 选择时间范围
- 点击箭头按钮
- 输入任务名字、任务描述
- 创建任务

> 通过以上步骤就可以使用Elasticsearch的机器学习分析网站请求数量的变化，找到异常情况。

### 截图

![image](https://raw.githubusercontent.com/syhc006/documents/elastic/images/es-nginx-ml-1.jpg)
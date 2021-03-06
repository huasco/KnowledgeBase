# 第一部分 整体搭建架构说明    

![img](EFK.png)

如上图所示，这次采用了官方最新推荐的 Filebeat + ElasticSearch + Kibana 的架构，放弃之前的ELK组合，一来这是官方首先推荐的组合，二来从我们的需求上来说EFK更加简洁和高效。

# 第二部分 程序安装    

程序安装一共就两个步骤：

**第一步**，在一台服务器上（对应我们线上的运维服务器）部署 ElasticSearch + Kibana，通过 docker-compose 和官方的 dockerfile 构建镜像并启动容器；

**第二步**，在日志所在服务器（对应我们的测试应用服务器和生产应用服务器）上，安装Filebeat。（实际上只是下载压缩包并解压到制定目录）


# 第三部分 EFK配置说明    

### 3.1 filebeat 配置

**filebeat.yml**

```Yaml

# 加载config目录的input配置
filebeat.config.prospectors:
  path: config/*.yml
  reload.enabled: true
  reload.period: 10s

filebeat.prospectors:

processors:
  - drop_fields:
      fields: ["beat", "input_type", "type", "offset"]

# 文件输出，用于测试
output.file:
  path: "./tmp"
  filename: output
  
# 输出到es，注意pipeline和indices配置，indices对应input的配置
output.elasticsearch:
  hosts: ["localhost:9200"]
  template.enabled: false
  pipeline: "testpipline"
  indices:
    - index: "etbc-callcenter"
      when.contains:
        fields.app_id: "etbc-callcenter"

# 日志级别和日志目录
logging.level: error
logging.files:
  path: ./log/filebeat
  
```

**config/input_config.yml**

```Yaml

# 有多个目录可以配置多个input段，注意multiline的配置
- input_type: log
  paths:
    - /mnt/logs/etbc-callcenter/biz/*.log
  fields:
    app_id: etbc-callcenter
  multiline.pattern: '^\[[FATAL|ERROR|WARN|INFO|DEBUG|TRACE]'
  multiline.negate: true
  multiline.match: after
  scan_frequency: 5s
  clean_removed: true

```

**filebeat.sh (来自网上的一段启动脚本, 用法： ./filebeat.sh test|start|stop|restart|status)**

```Bash

#!/bin/bash
PATH=/usr/bin:/sbin:/bin:/usr/sbin
export PATH

# 注意修改这里的路径
agent="/usr/filebeat/filebeat"
args="-c /usr/filebeat/filebeat.yml -path.home /usr/filebeat -path.config /usr/filebeat -path.data /usr/filebeat/data -path.logs /var/log/filebeat"
test_args="-e -configtest"

test() {
$agent $args $test_args
}
start() {
    pid=`ps -ef |grep /usr/filebeat/data |grep -v grep |awk '{print $2}'`
    if [ ! "$pid" ];then
        echo "Starting filebeat: "
        test
        if [ $? -ne 0 ]; then
            echo
            exit 1
        fi
        $agent $args &
        if [ $? == '0' ];then
            echo "start filebeat ok"
        else
            echo "start filebeat failed"
        fi
    else
        echo "filebeat is still running!"
        exit
    fi
}
stop() {
    echo -n $"Stopping filebeat: "
    pid=`ps -ef |grep /usr/filebeat/data |grep -v grep |awk '{print $2}'`
    if [ ! "$pid" ];then
echo "filebeat is not running"
    else
        kill $pid
echo "stop filebeat ok"
    fi
}
restart() {
    stop
    start
}
status(){
    pid=`ps -ef |grep /usr/filebeat/data |grep -v grep |awk '{print $2}'`
    if [ ! "$pid" ];then
        echo "filebeat is not running"
    else
        echo "filebeat is running"
    fi
}
case "$1" in
    start)
        start
    ;;
    stop)
        stop
    ;;
    restart)
        restart
    ;;
    status)
        status
    ;;
    *)
        echo $"Usage: $0 {start|stop|restart|status}"
        exit 1
esac

```

### 3.2 elasticsearch 和 kibana 配置

es没有配置文件，所有的配置都是通过 http rest 接口去实现的


**ingest pipeline配置（注意 date 里面要配置一个时区，否则 kibana 解析日期会差8小时）**

```
PUT /_ingest/pipeline/huasco_pipline
{
    "description": "huasco_pipline",
    "processors": [
        {
            "grok": {
                "field": "message",
                "patterns": [
                    "\\[%{GREEDYDATA:level}\\]\\[%{TIMESTAMP_ISO8601:timestamp}\\]\\[%{GREEDYDATA:logger}\\]\\[%{GREEDYDATA:pid_pname}\\]\\[%{GREEDYDATA:class}\\]\\[%{GREEDYDATA:method}\\]\\[%{INT:line}\\]%{GREEDYDATA:msg}"
                ],
                "ignore_failure": true
            }
        },
        {
            "date": {
                "field": "timestamp",
                "formats": [
                    "YYYY-MM-dd HH:mm:ss,SSS"
                ],
                "timezone": "Asia/Shanghai"
            }
        },
        {
            "trim": {
                "field": "msg"
        	}
        },
        {
            "remove": {
                "field": "source"
            }
        },
        {
            "remove": {
                "field": "fields.app_id"
            }
        },
        {
            "remove": {
                "field": "message"
            }
        },
        {
            "remove": {
                "field": "timestamp"
            }
        }
    ]
}

```

**修改默认的 ingest pattern**

打开\modules\ingest-common\ingest-common-5.4.0.jar，修改里面patterns目录下的grok-patterns，修改 GREEDYDATA（因为默认的不能读取多行，这个和logstash不一致）

```
GREEDYDATA ([\s\S]*)
```

**kibana 配置**

kibana.yml

```Yaml
---
## Default Kibana configuration from kibana-docker.
## from https://github.com/elastic/kibana-docker/blob/master/build/kibana/config/kibana.yml
#
server.name: kibana
server.host: "0"
elasticsearch.url: http://elasticsearch:9200

```


# 第四部分 使用注意事项    

### 4.1 kibana日志时间格式和默认日志字段的配置
kibana的默认时间格式比较复杂，展示效果不好，我们可以在 Management/Advanced Settings 中修改 dateFormat 格式为：** YYYY-MM-DD HH:mm:ss.SSS **，还有它默认展示的字段是 Date 和 \_source，我们可以修改为想要的字段，即修改 defaultColumns 为： msg（这是我们日志中的详情字段）

### 4.2 kibana使用请参考[官网文档](https://www.elastic.co/guide/en/kibana/current/discover.html)
有一点需要注意，kibana discover 界面上的带加减号的放大镜，不是放大和缩小的功能，而是添加当前文本为搜索条件的意思。

### 4.3 filebeat日志解析优化
由于filebeat定时去监测文件是否有变动，如果日志文件积累的比较多，那么对于性能是有影响的。这里我们采用的做法是每天定时任务将历史日志文件转移到备份文件夹，让filebeat不在扫描历史文件。



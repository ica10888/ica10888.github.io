---
layout:     post
title:      EFK方案在kubernetes上收集日志
subtitle:   EFK方案在kubernetes上收集日志
date:       2019-07-11
author:     ica10888
catalog: true
tags:
    - kubernetes
    - 日志收集
    - 日志清理
    - efk
    - logtrail
    - fluentd
---


# EFK 方案在kubernetes上收集日志

### fluentd

在之前的博客中，我提到过使用Elasticsearch , Logstash, Kibana + FileBeat 实现 kubernetes 上的日志收集。

就先在来说，修改了日志收集的方案，使用 Elasticsearch Kibana + fluentd 对日志进行收集，也就是 EFK方案。

##### fluentd的优势

![avatar](http://liangiter.top/2018/10/18/Fluentd%E7%AE%80%E4%BB%8B/fluentd_1.jpg)

- 基于Rust编写的程序，有更好的性能。去除了 java 的 Logstash 中间层，减少了内存使用。使用很少的系统资源，基于内存和文件的缓冲，具有可靠性和高可用性。

- Fluentd社区活跃，Fluentd 本身就是 CNCF 基金会的顶级项目，可以说是有一个好爹。同时基于 kubernetes 生态，使用起来更加方便。

- 输入有多种格式，除了tail之外，还可以以 tcp 协议，或者 syslog 等形式收集日志，使用灵活，扩展性强。

- 输出同样有多种格式，除了常用的 Elasticsearch 作为数据库外，还可以存储日志到其他类型的数据库，甚至是云厂商的支持，这些数据还可以用于训练和分析。

  


### fluentd 的使用

charts仓库：https://github.com/kiwigrid/helm-charts/tree/master/charts/fluentd-elasticsearch

官方已经将仓库移动到这里进行维护了，helm/charts 有一段时间没有维护了

基本上配置文件中修改 Elasticsearch 的配置就可以使用了，会在每一个node启动一个 fluentd 的 daemonsets 收集 /var/log 的日志

##### 收集master节点的日志

容忍污点，让 damonset 能够部署 pod 到 master 节点上

``` yaml
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
        operator: Equal
```

### fluentd的configmap

日志收集分成2个部分。

一个收集容器的日志，也就是`kubernetes-%Y-%m-%d` 的收集的是/var/log/containers/*.log 的日志，将符合正则条件的日志 **匹配这个子表达式的文本**  分组  **捕获**  到 json 节点里面，最后收集到  Elasticsearch 里面。

另一个是收集 kubernetes 日志，也就是 kubelet、kube-proxy、kube-apiserver、kube-controller-manager、kube-scheduler 等进程的日志。由于使用的是 kubespray 安装的 kubernetes 集群。所有的日志都会收集到 /var/log/messages 里面，收集日志的过程中加入 @timestamp 进行排序，最后收集到  Elasticsearch 里面，格式是`messages-%Y-%m-%d`。

##### 开发测试环境

``` xml
apiVersion: v1
kind: ConfigMap
data:
  containers.input.conf: |-
    <source>
      @id fluentd-containers.log
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/containers.log.pos
      tag raw.kubernetes.*
      read_from_head true
      <parse>
        @type multi_format
        <pattern>
          format json
          time_key time
          time_format %Y-%m-%dT%H:%M:%S.%NZ
        </pattern>
        <pattern>
          format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          time_format %Y-%m-%dT%H:%M:%S.%N%:z
        </pattern>
      </parse>
    </source>

    <match raw.kubernetes.**>
      @id raw.kubernetes
      @type detect_exceptions
      remove_tag_prefix raw
      message log
      stream stream
      multiline_flush_interval 5
      max_bytes 500000
      max_lines 1000
    </match>


    <filter kubernetes.**>
      @id filter_concat
      @type concat
      key message
      multiline_end_regexp /\n$/
      separator ""
    </filter>

    <filter kubernetes.**>
      @id filter_kubernetes_metadata
      @type kubernetes_metadata
    </filter>


    <filter kubernetes.**>
      @id filter_parser
      @type parser
      key_name log
      reserve_data true
      remove_key_name_field true
      <parse>
        @type multi_format
        <pattern>
          format json
        </pattern>
        <pattern>
          format regexp
          expression /^(?<longtime>[^\]]*)\s*(?<severity>[^ ]*) \[(?<app>[^\]]*)\] (?<pid>[^ ]*) (?<unuse>[^ ]*) \[(?<thread>[^\]]*)\] (?<class>[^ ]*)\s*: (?<message>[\w\W]*)$/
          time_format %d/%b/%Y:%H:%M:%S.%NZ
        </pattern>
        <pattern>
          format regexp
          expression /^(?<longtime>[^ ]* [^ ]*)\s*(?<severity>[^ ]*) (?<pid>[^ ]*) (?<unuse>[^ ]*) \[(?<thread>[^\]]*)\] (?<class>[^ ]*)\s*: (?<message>[\w\W]*)$/
          time_format %d/%b/%Y:%H:%M:%S.%NZ
        </pattern>
        <pattern>
          format none
        </pattern>
      </parse>
    </filter>
  forward.input.conf: |-
    <source>
      @id forward
      @type forward
    </source>
  monitoring.conf: |-
    <source>
      @id prometheus
      @type prometheus
    </source>

    <source>
      @id monitor_agent
      @type monitor_agent
    </source>

    <source>
      @id prometheus_monitor
      @type prometheus_monitor
      <labels>
        host ${hostname}
      </labels>
    </source>

    <source>
      @id prometheus_output_monitor
      @type prometheus_output_monitor
      <labels>
        host ${hostname}
      </labels>
    </source>

    <source>
      @id prometheus_tail_monitor
      @type prometheus_tail_monitor
      <labels>
        host ${hostname}
      </labels>
    </source>
  output.conf: |-

    <filter messages.**>
      @type record_transformer
      enable_ruby true
      <record>
        @timestamp "${time.strftime('%Y-%m-%dT%H:%M:%S%z')}"
      </record>
    </filter>

    <match messages.**>
      @id kubernetes
      @type elasticsearch
      @log_level info
      include_tag_key true
      type_name _doc
      host "#{ENV['OUTPUT_HOST']}"
      port "#{ENV['OUTPUT_PORT']}"
      scheme "#{ENV['OUTPUT_SCHEME']}"
      ssl_verify "#{ENV['OUTPUT_SSL_VERIFY']}"
      ssl_version "#{ENV['OUTPUT_SSL_VERSION']}"
      logstash_format true
      logstash_prefix messages
      reconnect_on_error true
    </match>

    <match  kubernetes.**>
      @id elasticsearch
      @type elasticsearch
      @log_level info
      include_tag_key true
      type_name _doc
      host "#{ENV['OUTPUT_HOST']}"
      port "#{ENV['OUTPUT_PORT']}"
      scheme "#{ENV['OUTPUT_SCHEME']}"
      ssl_verify "#{ENV['OUTPUT_SSL_VERIFY']}"
      ssl_version "#{ENV['OUTPUT_SSL_VERSION']}"
      logstash_format true
      logstash_prefix service
      reconnect_on_error true
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_thread_count 2
        flush_interval 5s
        retry_forever
        retry_max_interval 30
        chunk_limit_size "#{ENV['OUTPUT_BUFFER_CHUNK_LIMIT']}"
        queue_limit_length "#{ENV['OUTPUT_BUFFER_QUEUE_LIMIT']}"
        overflow_action block
      </buffer>
    </match>
  system.conf: |-
    <system>
      root_dir /tmp/fluentd-buffers/
    </system>
  system.input.conf: |-
    <source>
      @id messages.log
      @type tail
      <parse>
        @type regexp
        expression /^(?<timestamp>[^ ]*  [^ ]* [^ ]*) (?<node>[^ ]*) (?<app>[^ ]*): (?<message>.*)$/
      </parse>
      path /var/log/messages
      pos_file /var/log/messages.pos
      tag messages
    </source>

```
为了让es索引能够通过 `@timestamp` 时间排序，需要注意的是在这里会有时区的差距，可以使用podpreset 挂载时区来解决这个问题。

`@timestamp "${time.strftime('%Y-%m-%dT%H:%M:%S%z')}"`

##### 生产环境

生产环境输出的是json格式的日志，使用了不同的正则表达式

``` regex
^{"@timestamp":"(?<longtime>[^"]*)","rest":"(?<message>(([\\]["])+|[^"])*)","class":"(?<class>[^"]*)","severity":"(?<severity>[^"]*)","service":"(?<service>[^"]*)","XForwardedFor":"(?<XForwardedFor>[^"]*)","trace":"(?<trace>[^"]*)","span":"(?<span>[^"]*)","parent":"(?<parent>[^"]*)","exportable":"(?<exportable>[^"]*)","pid":"(?<pid>[^"]*)","thread":"(?<thread>[^"]*)"}$
```

在正则表达式中，使用 **分组语法** 中的 **捕获 ** 语法将日志保存到对应的json节点中，组的命名就是 json 的 key，捕获的数据就是 value。

其中 `([\\]["])+|[^"])*` 是防止在 `""` 中出现  `\"` 的情况

### logback.xml

对于 开发和测试环境，输出便于查看的日志，到了生产环境，收集全量数据，用于分析。

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true">
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    <springProperty scope="context" name="springAppName" source="spring.application.name"/>
    <springProperty scope="context" name="logPath" source="hanclouds.logging.path" defaultValue="/app/log"/>
    <!-- 文件日志位置 -->
    <!--<property name="LOG_FILE" value="${logPath}/${springAppName}"/>-->

    <property name="LOG_FILE" value="${logPath}/${springAppName}/${springAppName}"/>
    <!-- 控制台日志格式 -->
    <property name="CONSOLE_LOG_PATTERN"
              value="%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint}  %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx} "/>
    <!-- 控制台日志appender  -->
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        </filter>
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>utf8</charset>
        </encoder>
    </appender>
    <appender name="jsonConsole" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        </filter>
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <pattern>
                    <pattern>
                        {
                        "rest": "%message %ex{6}",
                        "class": "%logger{40}",
                        "severity": "%level",
                        "service": "${springAppName:-}",
                        "XForwardedFor":"%X{X-Forwarded-For:-}",
                        "trace": "%X{X-B3-TraceId:-}",
                        "span": "%X{X-B3-SpanId:-}",
                        "parent": "%X{X-B3-ParentSpanId:-}",
                        "exportable": "%X{X-Span-Export:-}",
                        "pid": "${PID:-}",
                        "thread": "%thread"
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
    </appender>

    <root level="INFO">
        <springProfile name="dev,test">
            <appender-ref ref="console"/>
        </springProfile>
        <springProfile name="prod">
            <appender-ref ref="jsonConsole"/>
        </springProfile>
    </root>

</configuration>
```

### 日志的清理

使用 curator 创建一个 Cronjob 定时清理索引

这里展示的是删除 `kubernetes-%Y-%m-%d` ，同理可以用到 `messages-%Y-%m-%d`上

每次删除7天前的索引

``` yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: action
data:
  action.yml: |
    actions:
      1:
        action: delete_indices
        description: >-
          Delete indices older than 7 days , for service-
          prefixed indices. Ignore the error if the filter does not result in an
          actionable list of indices (ignore_empty_list) and exit cleanly.
        options:
          ignore_empty_list: True
          disable_action: False
        filters:
        - filtertype: pattern
          kind: prefix
          value: service-
        - filtertype: age
          source: name
          direction: older
          timestring: '%Y.%m.%d'
          unit: days
          unit_count: 7
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
data:
  config.yml: |
    client:
      hosts:
        - {{ .Values.config.host }}
      port: {{ .Values.config.port }}
      url_prefix:
      use_ssl: False
      certificate:
      client_cert:
      client_key:
      ssl_no_validate: False
      http_auth:
      timeout: 30
      master_only: False

    logging:
      loglevel: INFO
      logfile:
      logformat: default
      blacklist: ['elasticsearch', 'urllib3']
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: curator
  labels:
    app: curator
spec:
  schedule: {{ .Values.curator.schedule }}
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: curator
              image: bobrik/curator:5.7.6
              args: ["--config", "/etc/curator/config.yml","/etc/curator/action.yml"]
              volumeMounts:
                - mountPath: "/etc/curator/action.yml"
                  name: action
                  subPath: action.yml
                - mountPath: "/etc/curator/config.yml"
                  name: config
                  subPath: config.yml
          restartPolicy: OnFailure
          volumes:
            - name: action
              configMap:
                name: action
                defaultMode: 0777
            - name: config
              configMap:
                name: config
                defaultMode: 0777

```




### 日志告警

使用 elastalert 对日志告警

使用的 helm/charts 仓库 ：<https://github.com/helm/charts/tree/master/stable/elastalert> 

相关配置

``` yaml
rules:
  outofmemory: |-
    es_host: es-elasticsearch-client.elk.svc.cluster.local
    es_port: 9200
    name: OOM alert
    type: frequency
    index: service-*
    num_events: 1
    timeframe:
        minutes: 1
    filter:
    - query:
        query_string:
          query: "outofmemory"
    alert:
    - "email"
  outofmemory2: |-
    es_host: es-elasticsearch-client.elk.svc.cluster.local
    es_port: 9200
    name: OOM alert2
    type: frequency
    index: service-*
    num_events: 1
    timeframe:
        minutes: 1
    filter:
    - query:
        query_string:
          query: "\"out of memory\""
    alert:
    - "email"
```

 对含有字段  `outofmemory`  和 `out of memory` 的日志进行告警，邮箱的配置直接配置在的 templates 的 config.yaml 里面

### 日志查看

使用kibana 的插件 logtrail 查看日志

这里展示的是查看`kubernetes-%Y-%m-%d` 的日志，同理可以用到 `messages-%Y-%m-%d`上

logtrail.json

``` json
{
  "version" : 2,
  "index_patterns" : [
    {      
      "es": {
        "default_index": "service-*"
      },
      "tail_interval_in_seconds": 10,
      "es_index_time_offset_in_seconds": 0,
      "display_timezone": "Etc/CST",
      "display_timestamp_format": "YYYY MMM DD HH:mm:ss",
      "max_buckets": 500,
      "default_time_range_in_days" : 0,
      "max_hosts": 1000,
      "max_events_to_keep_in_viewer": 5000,
      "default_search": "",
      "fields" : {
        "mapping" : {
            "timestamp" : "@timestamp",
            "display_timestamp" : "@timestamp",
            "hostname" : "kubernetes.container_name",
            "program": "kubernetes.pod_name",
            "message": "message"
        },
        "message_format": "{{{kubernetes.namespace_name}}} {{{severity}}} [{{{thread}}}] {{{class}}} {{{message}}} ",
        "keyword_suffix" : "keyword"
      },
      "color_mapping" : {
	"field" : "severity",
        "mapping" : {
          "ERROR": "#ff3232",
          "WARN": "#ff7f24",
          "DEBUG": "#ffb90f",
          "TRACE": "#a2cd5a"
        }
      }
    }
  ]
}
```



### 参考

[fluentd官方文档](https://docs.fluentd.org/)

[Kubernetes中的Taint和Toleration](https://jimmysong.io/posts/kubernetes-taint-and-toleration/)

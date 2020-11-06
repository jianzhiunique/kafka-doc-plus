# Kafka监控

Kafka本身没有开箱即用的监控系统，仅仅通过JMX输出了很多项监控指标。

在Kafka监控方面，大家的做法基本上是使用JMX exporter + Prometheus + Grafana

1. JMX exporter用来将Kafka暴露的JMX指标转换为Prometheus可以用的数据接口
2. Prometheus是时序数据库产品，用来采集JMX exporter的接口数据
3. Grafana是图表展示程序，可以将多种数据库的数据作图

除此之外，推荐使用其他的exporter来收集topic相关的信息、消费者相关的lag信息等。

## Prometheus相关文档

```
# 安装prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.17.1/prometheus-2.17.1.linux-amd64.tar.gz
tar xvfz prometheus-*.tar.gz

# 进入prometheus目录
cd prometheus-*
./prometheus --help

# prometheus的配置文件是这样的
# 可以看到只需要把要采集的URL加进去就可以了
global:
  scrape_interval:     15s # 这里是全局采集频率
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

scrape_configs:
  - job_name: prometheus  # 自定义的采集任务的名字
    static_configs:
      - targets: ['localhost:9090'] # 定期采集的接口URL


# 启动prometheus
./prometheus --config.file=prometheus.yml

# 如果需要自定义listen端口
./prometheus --config.file=prometheus.yml --web.listen-address=0.0.0.0:8080

# 附赠一个实用功能，查询所有标签
localhost:9090/api/v1/label/__name__/values
```

## Grafana相关文档
```
-----------------------------------------
# 安装可以使用仓库，centos上添加/etc/yum.repos.d/grafana.repo

[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt

# 然后安装
yum install grafana

-----------------------------------------
# 常用的配置文件修改/etc/grafana/grafana.ini
# 管理员口令
admin_password = xxxx

# 允许匿名用户查看
[auth.anonymous]
# enable anonymous access
enabled = ture

# 如果需要以xxx.yyy.com/grafana 访问
[server]
domain = xxx.yyy.com
root_url = %(protocol)s://%(domain)s/grafana/

# 主题设置
default_theme = light

# 允许其他页面iframe嵌套
allow_embedding = true
-----------------------------------------
# nginx 配置
location ^~ /grafana/ {
    proxy_pass http://localhost:3000/;
}
-----------------------------------------
# 启动服务
service grafana-server start
```

## Kafka集群JMX监控方法

JMX exporter一般是以javaagent的方式启动的，但这样有不方便的地方

1. 修改exporter配置后需要重启Kafka，但一般我们不希望因监控指标收集的修改而重启kafka
2. 对于开始没有部署监控的kafka集群，我们也需要在配置后重启kafka服务

所以我们用一个空java项目以javaagent的方式把exporter带起来，然后让这个程序去收集JMX的指标，但这样有个问题是不能拿到JVM的信息，但好在Kafka暴露的指标里也有JVM的相关信息。

#### exporter部署过程

1. 开启Kafka JMX端口，JMX_PORT=9999
2. 下载jmx_prometheus_javaagent-0.13.0.jar，https://github.com/prometheus/jmx_exporter
3. 用springboot写一个空web项目
4. 编写exporter配置文件
5. 将以上文件放在一起，以javaagent的方式启动空web项目
6. 修改prometheus的配置文件，使其搜刮监控数据
7. 配置grafana模板（提供grafana模板）

```
#!/bin/bash
JAVA_BIN=`whereis java | awk '{print $2}'`
echo $JAVA_BIN
ps aux | grep kafka-jmx | grep -v grep | awk '{print $2}' | xargs kill -9
nohup $JAVA_BIN -javaagent:/home/www/jmx_prometheus_javaagent-0.13.0.jar=8070:/home/www/config-0.11.0.3.yaml -jar /home/www/kafka-jmx-exporter-1.0.0.jar &
```

#### 配置文件示例
[Kafka 0.11.0.x exporter配置文件](kafka_0110_jmx_monitor.md)

## Kafka Topic/Consumer 监控方法

为了使用Kafka的业务方能够看到topic的历史offset情况以及消费者组的消费lag情况，一般还会部署一个业务层面的监控

这里推荐使用https://github.com/danielqsj/kafka_exporter

```
kafka_exporter --kafka.server=kafka:9092 [--kafka.server=another-server ...]
```

#### Grafana模板

```
{
  "__inputs": [
    {
      "name": "DS_PROMETHEUS-KAFKA",
      "label": "Prometheus-kafka",
      "description": "",
      "type": "datasource",
      "pluginId": "prometheus",
      "pluginName": "Prometheus"
    }
  ],
  "__requires": [
    {
      "type": "grafana",
      "id": "grafana",
      "name": "Grafana",
      "version": "7.0.0"
    },
    {
      "type": "panel",
      "id": "graph",
      "name": "Graph",
      "version": ""
    },
    {
      "type": "datasource",
      "id": "prometheus",
      "name": "Prometheus",
      "version": "1.0.0"
    },
    {
      "type": "panel",
      "id": "stat",
      "name": "Stat",
      "version": ""
    }
  ],
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": "-- Grafana --",
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "gnetId": null,
  "graphTooltip": 0,
  "id": null,
  "iteration": 1598518606360,
  "links": [],
  "panels": [
    {
      "datasource": "${DS_PROMETHEUS-KAFKA}",
      "fieldConfig": {
        "defaults": {
          "custom": {
            "align": null
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 0,
        "y": 0
      },
      "id": 2,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "values": false
        }
      },
      "pluginVersion": "7.0.0",
      "targets": [
        {
          "expr": "kafka_brokers{job=\"$cluster\"}",
          "interval": "",
          "legendFormat": "",
          "refId": "A"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "集群节点数",
      "type": "stat"
    },
    {
      "datasource": "${DS_PROMETHEUS-KAFKA}",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "mappings": [],
          "noValue": "0",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 1000
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 4,
        "y": 0
      },
      "id": 17,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "values": false
        }
      },
      "pluginVersion": "7.0.0",
      "targets": [
        {
          "expr": "count((sum by (topic) (kafka_topic_partitions{job=\"$cluster\"})))",
          "instant": true,
          "interval": "",
          "legendFormat": "{{topic}}:{{partition}}",
          "refId": "A"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "topic数",
      "type": "stat"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS-KAFKA}",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 8,
        "x": 8,
        "y": 0
      },
      "hiddenSeries": false,
      "id": 4,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "kafka_topic_partition_current_offset{job=\"$cluster\",topic=\"$topic\"}",
          "interval": "",
          "legendFormat": "topic {{topic}} : {{partition}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "topic分区最新offset",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:33567",
          "format": "none",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:33568",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS-KAFKA}",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 8,
        "x": 16,
        "y": 0
      },
      "hiddenSeries": false,
      "id": 6,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "kafka_topic_partition_oldest_offset{job=\"$cluster\",topic=\"$topic\"}",
          "interval": "",
          "legendFormat": "{{topic}} : {{partition}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "topic分区最早offset",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:33616",
          "format": "none",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:33617",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "datasource": "${DS_PROMETHEUS-KAFKA}",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {
            "align": null
          },
          "mappings": [],
          "noValue": "0",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "orange",
                "value": 1
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 0,
        "y": 4
      },
      "id": 14,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "values": false
        }
      },
      "pluginVersion": "7.0.0",
      "targets": [
        {
          "$$hashKey": "object:25",
          "aggregation": "Last",
          "decimals": 2,
          "displayAliasType": "Warning / Critical",
          "displayType": "Regular",
          "displayValueWithAlias": "Never",
          "expr": "count(kafka_topic_partition_leader_is_preferred{job=\"$cluster\"}==0)",
          "format": "time_series",
          "instant": true,
          "interval": "",
          "legendFormat": "{{topic}}:{{partition}}",
          "refId": "A",
          "units": "none",
          "valueHandler": "Number Threshold"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "非首选副本",
      "type": "stat"
    },
    {
      "datasource": "${DS_PROMETHEUS-KAFKA}",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "mappings": [],
          "noValue": "0",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 1
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 4,
        "y": 4
      },
      "id": 10,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "values": false
        }
      },
      "pluginVersion": "7.0.0",
      "targets": [
        {
          "expr": "count(kafka_topic_partition_under_replicated_partition{job=\"$cluster\"}==1)",
          "instant": true,
          "interval": "",
          "legendFormat": "{{topic}}:{{partition}}",
          "refId": "A"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "离线副本",
      "type": "stat"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS-KAFKA}",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 10,
        "w": 8,
        "x": 0,
        "y": 8
      },
      "hiddenSeries": false,
      "id": 15,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum(rate(kafka_topic_partition_current_offset{job=\"$cluster\"}[1m]))",
          "interval": "",
          "legendFormat": "topic {{topic}} : {{partition}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "集群offset变化率",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS-KAFKA}",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 10,
        "w": 8,
        "x": 8,
        "y": 8
      },
      "hiddenSeries": false,
      "id": 12,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "kafka_consumergroup_lag{job=\"$cluster\",topic=\"$topic\"}",
          "interval": "",
          "legendFormat": "[{{consumergroup}}] topic:{{topic}}  partition:{{partition}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "消费者组lag",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:108",
          "format": "none",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:109",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS-KAFKA}",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 10,
        "w": 8,
        "x": 16,
        "y": 8
      },
      "hiddenSeries": false,
      "id": 18,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "kafka_consumergroup_current_offset{job=\"$cluster\",topic=\"$topic\"}",
          "interval": "",
          "legendFormat": "[{{consumergroup}}] topic:{{topic}}  partition:{{partition}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "消费者组当前提交位置",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:108",
          "format": "none",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:109",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS-KAFKA}",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 10,
        "w": 8,
        "x": 0,
        "y": 18
      },
      "hiddenSeries": false,
      "id": 16,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum(rate(kafka_topic_partition_current_offset{job=\"$cluster\",topic=\"$topic\"}[1m]))",
          "interval": "",
          "legendFormat": "topic {{topic}} : {{partition}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "topic offset变化率",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS-KAFKA}",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 10,
        "w": 8,
        "x": 8,
        "y": 18
      },
      "hiddenSeries": false,
      "id": 19,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "kafka_consumergroup_lag{job=\"$cluster\",topic=\"$topic\", consumergroup=\"$group\"}",
          "interval": "",
          "legendFormat": "[{{consumergroup}}] topic:{{topic}}  partition:{{partition}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "消费者组lag - 筛选",
      "tooltip": {
        "shared": true,
        "sort": 2,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:108",
          "format": "none",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:109",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "${DS_PROMETHEUS-KAFKA}",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 10,
        "w": 8,
        "x": 16,
        "y": 18
      },
      "hiddenSeries": false,
      "id": 20,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "kafka_consumergroup_current_offset{job=\"$cluster\",topic=\"$topic\", consumergroup=\"$group\"}",
          "interval": "",
          "legendFormat": "[{{consumergroup}}] topic:{{topic}}  partition:{{partition}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "消费者组当前提交位置 - 筛选",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:108",
          "format": "none",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:109",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    }
  ],
  "schemaVersion": 25,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": [
      {
        "allValue": null,
        "current": {},
        "datasource": "${DS_PROMETHEUS-KAFKA}",
        "definition": "label_values(kafka_brokers,job)",
        "hide": 0,
        "includeAll": false,
        "label": null,
        "multi": false,
        "name": "cluster",
        "options": [],
        "query": "label_values(kafka_brokers,job)",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {},
        "datasource": "${DS_PROMETHEUS-KAFKA}",
        "definition": "label_values(kafka_topic_partitions,topic)",
        "hide": 0,
        "includeAll": false,
        "label": null,
        "multi": false,
        "name": "topic",
        "options": [],
        "query": "label_values(kafka_topic_partitions,topic)",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {},
        "datasource": "${DS_PROMETHEUS-KAFKA}",
        "definition": "label_values(kafka_consumergroup_lag,consumergroup)",
        "hide": 0,
        "includeAll": false,
        "label": null,
        "multi": true,
        "name": "group",
        "options": [],
        "query": "label_values(kafka_consumergroup_lag,consumergroup)",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      }
    ]
  },
  "time": {
    "from": "now-30m",
    "to": "now"
  },
  "timepicker": {
    "refresh_intervals": [
      "10s",
      "30s",
      "1m",
      "5m",
      "15m",
      "30m",
      "1h",
      "2h",
      "1d"
    ]
  },
  "timezone": "",
  "title": "Kafka 集群监控",
  "uid": "h6EGygkGk",
  "version": 36
}
```
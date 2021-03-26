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

https://github.com/danielqsj/kafka_exporter 是一个不错的选择，我曾经在线上环境使用了很长时间

```
kafka_exporter --kafka.server=kafka:9092 [--kafka.server=another-server ...]
```

直到某天我发现grafana上的图在业务高峰期间无法展示，原因是集群上的Topic有2000多个，高峰期数据流量很大，这个kafka_exporter是向Kafka发请求来获取所有消费者组以及信息的，高峰期Kafka需要处理很多请求，导致监控数据拿不到。于是我在空项目里加了lag指标的功能，这样只需要部署一个程序就可以拿到JMX数据和Lag数据了。

https://github.com/jianzhiunique/kafka-jmx-exporter

#### Grafana模板

Grafana模板

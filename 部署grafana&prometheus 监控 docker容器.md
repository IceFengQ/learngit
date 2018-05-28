# 部署grafana&prometheus监控docker容器
### 部署环境
主机：10.8.8.14 系统：centos7  docker版本：Docker version 17.12.1-ce 

组件：**promtheus**(监控数据整合存储查询)+ **node-exporter**(监控节点)+ **cadvisor**(监控容器)+ **grafana**（展示）

部署方式：容器方式

脚本位置：/home/master/docker-project

**prometheus**可以很好地记录任何纯数字时间序列。 它既适用于以机器为中心的监控，也适用于高度动态的面向服务的体系结构的监控。 在微服务世界中，它支持多维数据收集和查询是一个特别的优势。
**grafana**丰富的图形化展示，支持多种后台监控服务平台接入。


### 一、部署容器命令启动
1，启动cadvisor插件，并挂在docker数据目录。

```
docker run -d -p 8001:8080 \
  --restart=always \
  -v "/:/rootfs:ro" \
  -v "/var/run:/var/run:rw" \
  -v "/sys:/sys:ro" \
  -v "/var/lib/docker/:/var/lib/docker:ro" \ 
  --name cadvisor \
  google/cadvisor
``` 

2，启动node-exporter插件，监控各节点主机资源。

``` 
docker run -d -p 9100:9100  \
  -v "/proc:/host/proc:ro"  \
  -v "/sys:/host/sys:ro"  \
  -v "/:/rootfs:ro"  \
  --name prometheus_node-exporter  \
  prom/node-exporter  \
  --path.procfs /host/proc  \
  --path.sysfs /host/sys  \
  --collector.filesystem.ignored-mount-points "^(/rootfs|/host|)/(sys|proc|dev|host|etc)($$|/)"  \
  --collector.filesystem.ignored-fs-types "^(sys|proc|auto|cgroup|devpts|ns|au|fuse\.lxc|mqueue)(fs|)$$"
```

3，配置并启动prometheus服务。

prometheus.yml

```
# my global config

global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s # By default, scrape targets every 15 seconds.

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.

rule_files:
  # - "first.rules"
  #   # - "second.rules"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['10.8.8.14:9100', '10.8.8.14:8001']
        labels:
          env: 'prod_test'
      - targets: ['10.8.8.12:9100', '10.8.8.12:8001']
        labels:
          env: 'serviceCenter'
```

启动容器

```
docker run -d -p 9090:9090 \
  --restart=always \
  -v "/home/docker-project/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml" \
  -v "/home/docker-db/prometheus:/prometheus" \
  --name prometheus \
  prom/prometheus
```

4,启动grafana及持久化存储

```
docker run -d \
  -v "/home/docker-db/grafana:/var/lib/grafana" \
  --name grafana-storage \
  busybox:latest

docker run  -d -p 8000:3000  \
  --restart=always \
  --name=grafana \
  --volumes-from grafana-storage \
  -e "GF_SECURITY_ADMIN_PASSWORD=123456" \
  -e "GF_SMTP_ENABLED=true" \  ##测试未使用
  -e "GF_SMTP_SKIP_VERIFY=true" \  ##测试未使用
  -e "GF_SMTP_HOST=smtp.exmail.qq.com:25" \  ##测试未使用
  -e "GF_SMTP_USER=monitor@*.com" \  ##测试未使用
  -e "GF_SMTP_PASSWORD=*" \  ##测试未使用
  -e "GF_SMTP_FROM_ADDRESS=monitor@*.com" \  ##测试未使用
 grafana/grafana
```

### 二、检测状态
访问10.8.8.14:8008进入prometheus服务界面，选择status--targets查看状态。


访问10.8.8.14:8000，grafana web界面。

添加prometheus数据库按需配置。



# Prometheus on Kubernetes


本项目为Prometheus Operator部署监控相关的Kubernetes定义，快速部署请参考以下Quickstart


<details><summary>镜像列表与离线使用</summary>

---

- registry.cn-hangzhou.aliyuncs.com/clab-docker/prometheus-operator:v0.30.0
- registry.cn-hangzhou.aliyuncs.com/clab-docker/configmap-reload:v0.0.1
- registry.cn-hangzhou.aliyuncs.com/clab-docker/prometheus-config-reloader:v0.30.0
- registry.cn-hangzhou.aliyuncs.com/clab-docker/prometheus:v2.11.1
- registry.cn-hangzhou.aliyuncs.com/clab-docker/grafana:7.2.0
- registry.cn-hangzhou.aliyuncs.com/clab-docker/kafka_exporter:v0.0.1
- registry.cn-hangzhou.aliyuncs.com/clab-docker/kube-rbac-proxy:v0.4.1
- registry.cn-hangzhou.aliyuncs.com/clab-docker/kube-state-metrics:v1.5.0
- registry.cn-hangzhou.aliyuncs.com/clab-docker/addon-resizer:1.8.4
- registry.cn-hangzhou.aliyuncs.com/clab-docker/node-exporter:v1.3.1
- registry.cn-hangzhou.aliyuncs.com/clab-docker/statping:v0.0.5
- registry.cn-hangzhou.aliyuncs.com/clab-docker/alertmanager:v0.17.0
- registry.cn-hangzhou.aliyuncs.com/clab-docker/prometheus-webhook-dingtalk:1.4.0
- registry.cn-hangzhou.aliyuncs.com/clab-docker/ding2wechat:v0.1.5
- registry.cn-hangzhou.aliyuncs.com/clab-docker/cloudera_exporter:v0.0.1
- registry.cn-hangzhou.aliyuncs.com/clab-docker/yarn_exporter:v0.0.2
- registry.cn-hangzhou.aliyuncs.com/clab-docker/kudu_exporter:v0.0.4
- registry.cn-hangzhou.aliyuncs.com/clab-docker/prometheus-es-exporter:0.14.0
- registry.cn-hangzhou.aliyuncs.com/clab-docker/azure-scheduledevents-exporter:21.01.1
- registry.cn-hangzhou.aliyuncs.com/clab-docker/mysqld-exporter
- registry.cn-hangzhou.aliyuncs.com/clab-docker/elasticsearch_exporter:1.1.0rc1
- registry.cn-hangzhou.aliyuncs.com/clab-docker/mongodbexporter
- registry.cn-hangzhou.aliyuncs.com/clab-docker/redis_exporter

**Package Ver.20200810**

[oss://devops-components/prometheus/prometheus_20200810.images.gz](https://oss.console.aliyun.com/bucket/oss-cn-hangzhou/devops-components/object?path=prometheus%2F)

**Usage**

```
gzip -d -c prometheus_20200810.images.gz | docker load
newRepoPrefix=nexus.xsio.cn/test # 替换为本地仓库地址的前缀
docker images --format "{{.Repository}}:{{.Tag}}" -f reference="registry.cn-hangzhou.aliyuncs.com/clab-docker/*" | while read i;do echo "-> process $i" && docker tag $i ${i/registry.cn-hangzhou.aliyuncs.com\/clab-docker/$newRepoPrefix} && docker push ${i/registry.cn-hangzhou.aliyuncs.com\/clab-docker/$newRepoPrefix};done
```

</details>


## Quickstart

0. 拉取本项目

手动下载 [https://gitlab.cd.xsio.cn/devops/prometheus-operator/-/archive/master/prometheus-operator-master.tar.gz](https://gitlab.cd.xsio.cn/devops/prometheus-operator/-/archive/master/prometheus-operator-master.tar.gz)，然后上传到远程机器并解压

或者

在[这里](https://gitlab.cd.xsio.cn/devops/prometheus-operator/-/settings/repository)添加`Deploy Keys`，然后在远程机器上`git clone ssh://git@gitlab.cd.xsio.cn:22222/devops/prometheus-operator.git`

> （可选）创建钉钉机器人，关键词至少包括`service`,`FIRING`,`RESOLVED`，获取WEBHOOK URL备用

1. 为prometheus/grafana确认机器和数据目录

> 需要多节点或高可用，请另行使用NFS、云盘等存储

> 如果只想使用临时存储，请参考`3-prometheus/prometheus-prometheus.yaml`和`7-grafana/grafana-deployment.yaml`中的注释进行

> 以下为本地单点存储

```
kubectl get nodes # 挑选一台机器
kubectl label nodes xxxxxxxx prometheus-data=true # 为这台机器打标签
ssh xxxxxxxx mkdir -p /opt/prometheus/{prometheus,grafana} # ！登录选择的机器创建目录
kubectl apply -f create-local-storage.yaml # 创建对应PV和StorageClass
```

2. 启动promethues-operator

```
kubectl apply -f create-monitoring-namespace.yaml
kubectl apply -f 1-custom-resource-definition
kubectl apply -f 2-prometheus-operator
kubectl -nmonitoring get pod -l apps.kubernetes.io/name=prometheus-operator # 确认状态为Running
```

3. 启动promethues与exporter

```
kubectl apply -f 3-prometheus
kubectl -nmonitoring get pod -l app=prometheus -o wide # 确认状态为Running，节点为之前选择的机器
kubectl apply -f 6-exporter --recursive
# ingress相关报错可忽略，如果是托管版Kubernetes，可以跳过kube-scheduler和kube-controller-manager
```

此时可以打开`http://任意节点IP:30900/targets`确认prometheus工作正常，不方便访问的话可以跳过

4. 启动grafana

```
kubectl apply -f 7-grafana
kubectl -nmonitoring get pod -l app=grafana # 确认状态为Running
```

此时可以打开`http://任意节点IP:30300`确认grafana工作正常，默认密码为`admin/admin`

5. 启动其他监控


- kafka（推荐🌟）

```
vi 8-custom-exporter/kafka-exporter/kafka-exporter-deployment.yaml
# 编辑kafka地址，如果只有一个节点，请删除多余行
kubectl apply -f 8-custom-exporter/kafka-exporter
kubectl -nmonitoring get pod -l app=kafka-exporter # 确认状态为Running
```

- statping服务监控（推荐🌟）

```
vi 8-custom-exporter/statping/statping-configMap.yaml
# 按需修改services中需要监控的地址，注意批量替换域名和namespace，修改DINGTALK_URL为钉钉机器人地址或企业微信机器人地址
kubectl apply -f 8-custom-exporter/statping
kubectl -nmonitoring get pod -l app=statping # 确认状态为Running
```

此时可以打开`http://任意节点IP:30800`确认statping工作正常，默认密码为`admin/Convertlab0202`

- 外部机器监控（推荐🌟）

对Kubernetes之外的机器进行监控，在每台机器上执行

```
sudo yum install -y https://devops-components.oss-cn-hangzhou.aliyuncs.com/prometheus/node_exporter-1.3.1-clab.el7.centos.x86_64.rpm
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

之后，修改配置并启用

```
vi 8-custom-exporter/external-node-exporter/external-node-exporter-endpoints.yaml
# 修改addresses字段，更新为需要监控的机器IP列表
kubectl apply -f 8-custom-exporter/external-node-exporter
```

> 对外部机器的监控需要访问其9100端口，请注意防火墙开通情况


- CDH（推荐🌟）

```
vi 8-custom-exporter/cloudera-exporter/cloudera-exporter-configMap.yaml
# 修改host字段为CDH Manager的IP，按实际修改user，最好新建一个只读用户
kubectl apply -f 8-custom-exporter/cloudera-exporter
kubectl -nmonitoring get pod -l app=cloudera-exporter # 确认状态为Running
```

- Spring Boot Actuator

需要被监控的应用对应的Kubernetes Service YAML中需要打以下标签

```
actuator_should_be_scraped: "true"
actuator_scrape_port: "9999"
```

推荐在发布脚本中配置label，如果不方便，可以手动打label，例如

```
kubectl -ntest label svc app actuator_should_be_scraped=true
kubectl -ntest label svc app actuator_scrape_port=9999
```

需要被监控的应用启用`spring-boot-starter-actuator`和`micrometer-registry-prometheus`依赖，且在`application.yml`进行相关配置，推荐最小配置为

```
management:
  server:
    port: 9999
  endpoint:
    metrics:
      enabled: true
    prometheus:
      enabled: true
  endpoints:
    enabled-by-default: false
    web:
      exposure:
        include: 'prometheus'
```

默认已包含于`6-exporter`，新增监控不需要额外操作，已有监控执行

```
kubectl apply -f 6-exporter/spring-boot-actuator
```

- JMX (作为Spring Boot Actuator备用))

<details><summary>点此展开</summary>

---

JMX监控通过prometheus提供的jmx-exporter进行采集，其本身是一个java agent，在应用启动时跟随启动，并暴露供采集的端口。

为了最小化对现有应用的影响，监控相关的链路比较复杂，说明如下：

I. Prometheus Operator端配置ServiceMonitor

默认已包含于`6-exporter`，不需要额外操作

II. 在ConfigMap中配置启用jmx-exporter java agent

在SRE`全局配置`中搜索`JMX`，设置如下并更新配置

`JMX_EXPORTER_ENABLED`: `true`

`JMX_EXPORTER_BLACKLIST_OBJECT_NAMES`: `["kafka.producer:type=producer-topic-metrics,*","kafka.consumer:type=consumer-fetch-manager-metrics,topic=*,*","kafka.consumer:type=consumer-coordinator-metrics,*"]`

如果更多需求，具体可参考[这里](https://github.com/xsio/dockerimages/tree/master/alpine-oraclejdk8)

III. 确认发布脚本中为Service打上标签

需要被监控的应用对应的Kubernetes Service YAML中需要打以下标签

```
jmx_should_be_scraped: "true"
jmx_scrape_port: "7777"
```

添加后例如

```
apiVersion: v1
kind: Service
metadata:
  namespace: "{{ namespace }}"
  name: "{{ service_name }}"
  labels:
    jmx_should_be_scraped: "true"
    jmx_scrape_port: "7777"
spec:
  selector:
    app: "{{ service_name }}"
  ports:
  - name: web
    targetPort: web
    port: {{ server_port }}
{% for port in extra_ports %}
  - name: port{{ loop.index }}
    targetPort: {{ port }}
    port: {{ port }}
{% endfor %}
```

请确认当前环境对应的发布脚本中`service.j2`和`reactflow_service.j2`都已相应修改

IV. 重新发布服务

应用启动后，稍等两分钟，如果prometheus尚未采集到数据，排查如下

a. 进入应用容器，`netstat -nltp`确认`7777`端口是否监听

b. 如果未监听，确认`/opt/`下是否存在`jmx_prometheus_javaagent`目录，如果不存在，说明基础镜像未更新

c. 如果`jmx_prometheus_javaagent`目录存在，确认`start.sh`文件中是否包含`. .init_jmx_exporter.sh`一行，如果不包含，说明应用使用了自定义启动脚本

d. 如果`7777`已监听，`kubectl describe svc`确认Service标签是否打上，如果未打上，确认发布脚本工作是否正常

e. 如果标签打上，访问Prometheus网页界面，进入`Status`-`Targets`搜索`jmx`确认采集目标是否正常

</details>

- Azure计划事件（Azure云推荐🌟，特别地建议部署于Kudu节点）

用于获取Azure即将发生的维护事件，Kubernetes节点已默认包括，Kubernetes之外的机器需要执行

```
sudo yum install -y https://devops-components.oss-cn-hangzhou.aliyuncs.com/tools/azure-scheduledevents-exporter/21.01.1/azure-scheduledevents-exporter-21.01.1-1.el7.x86_64.rpm
sudo systemctl enable azure-scheduledevents-exporter --now
```

之后，修改配置并启用

```
vi 8-custom-exporter/azure-scheduledevents-exporter/external-azure-scheduledevents-exporter-endpoints.yaml
# 修改addresses字段，更新为需要监控的Kubernetes外部机器IP列表
kubectl apply -f 8-custom-exporter/azure-scheduledevents-exporter
```

> 对Kubernetes外部机器的监控需要访问其9879端口，请注意防火墙开通情况

- yarn（可选）

```
vi 8-custom-exporter/yarn-exporter/yarn-exporter-deployment.yaml
# 修改--resource-manager.address后的yarn resource manager地址，一般为8088端口
kubectl apply -f 8-custom-exporter/yarn-exporter
kubectl -nmonitoring get pod -l app=yarn-exporter # 确认状态为Running
```

- kudu（可选）

**要求kudu最低版本为1.10**

```
vi 8-custom-exporter/kudu-exporter/kudu-exporter-deployment.yaml
# 修改--kudu.bootstrap-address后的kudu master地址，一般为8051端口
kubectl apply -f 8-custom-exporter/kudu-exporter
kubectl -nmonitoring get pod -l app=kudu-exporter # 确认状态为Running
```

- 日志报错监控（可选）

*5分钟查询一次ES，非绝对准确*

```
vi 8-custom-exporter/elasticsearch-log-exporter/elasticsearch-log-exporter-deployment.yaml
# 修改--es-cluster后的elastic地址，如果为非9200端口，需要添加端口
vi 8-custom-exporter/elasticsearch-log-exporter/elasticsearch-log-exporter-configMap.yaml
# 修改QueryIndices后的index名称，调整为实际使用的前缀
kubectl apply -f 8-custom-exporter/elasticsearch-log-exporter
kubectl -nmonitoring get pod -l app=elasticsearch-log-exporter # 确认状态为Running
```

- redis（可选）

```
vi 8-custom-exporter/redis/redis-exporter-deployment.yaml
# 编辑redis地址，如果有密码,请填写,无则忽略
kubectl apply -f 8-custom-exporter/redis-exporter
kubectl -nmonitoring get pod -l app=redis-exporter # 确认状态为Running
```

- mongodb（可选）

```
vi 8-custom-exporter/mongodb-exporter/mongodb-exporter-deployment.yaml
# 编辑mongodb地址，可在demo中进行修改
kubectl apply -f 8-custom-exporter/mongodb-exporter
kubectl -nmonitoring get pod -l app=mongodb-exporter # 确认状态为Running
```

- mysql（可选）

```
vi 8-custom-exporter/mysql-exporter/mysql-exporter-deployment.yaml
# 编辑mysql地址,可在demo中进行修改
kubectl apply -f 8-custom-exporter/mysql-exporter
kubectl -nmonitoring get pod -l app=mysql-exporter # 确认状态为Running
```

- elasticsearch（可选）

```
vi 8-custom-exporter/elasticsearch-exporter/elasticsearch-exporter-deployment.yaml
# 编辑elasticsearch地址，可在demo中进行修改
vi 8-custom-exporter/elasticsearch-exporter/elasticsearch-log-exporter-deployment.yaml
# 编辑收集日志的elasticsearch地址,如果elasticsearch共用,请排除此文件执行
kubectl apply -f 8-custom-exporter/elasticsearch-exporter
kubectl -nmonitoring get pod -l app=elasticsearch-exporter # 确认状态为Running
```

6. 启动alertmanager报警（选择其一）

- 钉钉

```
vi 9-webhook/prometheus-webhook-dingtalk/prometheus-webhook-dingtalk-deployment.yaml 
# 修改ding.profile中钉钉机器人地址
kubectl apply -f 5-alertmanager
kubectl apply -f 9-webhook/prometheus-webhook-dingtalk
```

- 企业微信

```
vi 9-webhook/prometheus-webhook-qiyeweixin/prometheus-webhook-qiyeweixin-configMap.yaml
# 修改ding2wechat.yml字段中url定义的企业微信群机器人地址
kubectl apply -f 5-alertmanager
kubectl apply -f 9-webhook/prometheus-webhook-qiyeweixin
```

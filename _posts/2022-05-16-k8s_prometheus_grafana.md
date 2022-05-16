---
layout: post
title: 基于Prometheus+grafana+mysql搭建集群监控
date: 2022-05-16
tags: monitor   
---

# 基于Prometheus+grafana+mysql的监控开发

## 一、Prometheus搭建

参考：https://www.aneasystone.com/archives/2018/11/prometheus-in-action.html

Prometheus监控指标：https://segmentfault.com/a/1190000038888544

### 1.1 直接使用编译好的文件

[官网地址](https://prometheus.io/download/)直接对应版本的文件

```
$ wget https://github.com/prometheus/prometheus/releases/download/v2.4.3/prometheus-2.4.3.linux-amd64.tar.gz
$ tar xvfz prometheus-2.4.3.linux-amd64.tar.gz
```

然后切换到解压目录，检查 Prometheus 版本：

```
$ cd prometheus-2.4.3.linux-amd64
$ ./prometheus --version
prometheus, version 2.4.3 (branch: HEAD, revision: 167a4b4e73a8eca8df648d2d2043e21bdb9a7449)
  build user:       root@1e42b46043e9
  build date:       20181004-08:42:02
  go version:       go1.11.1
```

运行 Prometheus server：

```
$ ./prometheus --config.file=prometheus.yml
```

### 1.2 使用docker安装

直接运行docker即可

```
sudo docker run -d -p 9090:9090 prom/prometheus
```

一般会指定配置文件

```
sudo docker run -d -p 9090:9090 \
    -v ~/docker/prometheus/:/etc/prometheus/ \
    prom/prometheus
```

我们把配置文件放在本地 `~/docker/prometheus/prometheus.yml`，这样可以方便编辑和查看，通过 `-v` 参数将本地的配置文件挂载到 `/etc/prometheus/` 位置，这是 prometheus 在容器中默认加载的配置文件位置

### 1.3 配置Prometheus

正如上面两节看到的，Prometheus 有一个配置文件，通过参数 `--config.file` 来指定，配置文件格式为 [YAML](http://yaml.org/start.html)。我们可以打开默认的配置文件 `prometheus.yml` 看下里面的内容：

```
/etc/prometheus $ cat prometheus.yml 
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
 
# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093
 
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
 
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
 
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
 
    static_configs:
    - targets: ['localhost:9090']
```

Prometheus 默认的配置文件分为四大块：

- global 块：Prometheus 的全局配置，比如 `scrape_interval` 表示 Prometheus 多久抓取一次数据，`evaluation_interval` 表示多久检测一次告警规则；
- alerting 块：关于 Alertmanager 的配置，这个我们后面再看；
- rule_files 块：告警规则，这个我们后面再看；
- crape_config 块：这里定义了 Prometheus 要抓取的目标，我们可以看到默认已经配置了一个名称为 `prometheus` 的 job，这是因为 Prometheus 在启动的时候也会通过 HTTP 接口暴露自身的指标数据，这就相当于 Prometheus 自己监控自己，虽然这在真正使用 Prometheus 时没啥用处，但是我们可以通过这个例子来学习如何使用 Prometheus；可以访问 `http://localhost:9090/metrics` 查看 Prometheus 暴露了哪些指标；可以实现自定义指标的监控。

[更多的配置参数](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

### 1.4 数据类型

Prometheus定义了4种不同的指标类型(metric type)：Counter（计数器）、Gauge（仪表盘）、Histogram（直方图）、Summary（摘要）。

- Counter：只增不减的计数器

  见的监控指标，如http_requests_total，node_cpu都是Counter类型的监控指标。 一般在定义Counter类型指标的名称时推荐使用_total作为后缀。

  结合rate()函数函数可以获取变化率

- Gauge：可增可减的仪表盘

  与Counter不同，Gauge类型的指标侧重于反应系统的当前状态。因此这类指标的样本数据可增可减。常见指标如：node_memory_MemFree（主机当前空闲的内容大小）、node_memory_MemAvailable（可用内存大小）都是Gauge类型的监控指标。

  对于Gauge类型的监控指标，通过PromQL内置函数delta()可以获取样本在一段时间返回内的变化情况。例如，计算CPU温度在两个小时内的差异：`delta(cpu_temp_celsius{host="zeus"}[2h])`

- 使用Histogram和Summary分析数据分布情况

  为了区分是平均的慢还是长尾的慢，最简单的方式就是按照请求延迟的范围进行分组。例如，统计延迟在0~10ms之间的请求数有多少而10~20ms之间的请求数又有多少。通过这种方式可以快速分析系统慢的原因。Histogram和Summary都是为了能够解决这样问题的存在，通过Histogram和Summary类型的监控指标，我们可以快速了解监控样本的分布情况。

  例如，指标prometheus_tsdb_wal_fsync_duration_seconds的指标类型为Summary。 它记录了Prometheus Server中wal_fsync处理的处理时间，通过访问Prometheus Server的/metrics地址，可以获取到以下监控样本数据：

  ```
  # HELP prometheus_tsdb_wal_fsync_duration_seconds Duration of WAL fsync.
  # TYPE prometheus_tsdb_wal_fsync_duration_seconds summary
  prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.5"} 0.012352463
  prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.9"} 0.014458005
  prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.99"} 0.017316173
  prometheus_tsdb_wal_fsync_duration_seconds_sum 2.888716127000002
  prometheus_tsdb_wal_fsync_duration_seconds_count 216
  ```

  从上面的样本中可以得知当前Prometheus Server进行wal_fsync操作的总次数为216次，耗时2.888716127000002s。其中中位数（quantile=0.5）的耗时为0.012352463，9分位数（quantile=0.9）的耗时为0.014458005s。

### 1.5 常用指标

- 容器CPU使用情况

```
current: container_cpu_usage_seconds_total

request: kube_pod_container_resource_requests_cpu_cores

limit: kube_pod_container_resource_limits_cpu_cores
```

- 容器内存使用情况

```
current: container_memory_working_set_bytes

request: kube_pod_container_resource_requests_memory_bytes

limit: kube_pod_container_resource_limits_memory_bytes
```

-  网络收发情况

```
发送：container_network_transmit_bytes_total

接收：container_network_receive_bytes_total
```





## 二、grafana部署

#### 2.1 docker方式部署

```
docker run -d -p 3000:3000 grafana/grafana
```

【注意】docker方式部署，搭建监控面板之后需要将面板持久化以防止容器挂掉导致服务不可用。方式：`docker commit`打包镜像

#### 2.2 docker部署修改配置文件

- 匿名登录

  当grafana嵌入其他系统中使用时，想要隐去登录操作，此时需要修改配置文件

  ```
  [auth.anonymous]
  # enable anonymous access
  # 去掉注释，改为true，允许匿名访问
  enabled = true
  
  # specify organization name that should be used for unauthenticated users
  # 匿名用户属于的组织
  org_name = Main Org.
  
  # specify role for unauthenticated users
  # 匿名用户的角色/权限
  org_role = Viewer
  ```

  如果是用docker方式启动的，通过挂载的方式修改grafana的配置文件

  ```
   普通启动，挂载数据盘
  docker run  -d --name grafana -p 3000:3000   -v /data/grafana:/var/lib/grafana  grafana/grafana
  
  ## 复制出配置文件
  docker cp grafan:/etc/grafana/grafana.ini /data/grafana-data/etc/
  ## 修改配置文件，比如加上域名，比如修改端口为80，比如。。。
  
  ## kill重启
  docker kill grafana
  docker rm grafana
  docker run --user root  -d --name grafana -p 3000:3000  -v /data/grafana-data/etc:/etc/grafana/ 
  ```

- 跨域访问

  grafana嵌入其他系统中通常需要跨域访问，此时也需要修改配置文件

  ```
  /etc/grafana/grafana.ini配置文件修改allow_embedding = true
  
  kiosk=tv
  ```

- 使用修改之后的配置文件启动grafana

  ```
  docker run -d --name grafana_zl -p 3004:3000 -v /home/work/zhanglei/grafana:/etc/grafana/ jx-bd-hadoop13.zeus.lianjia.com:801/k8s/runonce/zl_grafana:v2.1-20211206
  ```



## 三、grafana面板搭建

### 3.1 数据源

搭建面板的第一步就是添加数据源，可以是Prometheus和mysql，公司的数据源包括：

- Prometheus数据源

  公司：

  ```
  http://new-api.insight.intra.ke.com
  ```

  监控推理服务的机器

  ```
  http://10.200.20.232:8096
  ```

  监控通过dcgm暴露的指标文件

  ```
  http://10.200.20.231:8096
  ```

- mysql数据源

  ```
  hostname：m6689.zeus.mysql.ljnode.com:6689
  
  database：aicloud
  
  user：aicloud_service
  
  password：39rH57YRMo936DK6
  ```



### 3.2 Promql

- 基本查询格式

  ```
  metric{kabel="value"}
  ```

- 操作符

  - 算术运算符

    ```
    +，-，*，/，%，^
    ```

  - 比较运算符

    ```
    ==，!=，>，<，>=，<= =~ !~
    ```

  - 逻辑运算符

    ```
    and，or，unless
    ```

  - 聚合运算符

    ```
    sum，min，max，avg，stddev，stdvar，count，count_values，bottomk，topk，quantile
    ```

- 内置函数

  ```
  rate、abs、ceil、floor等
  ```

- 使用示例：

  - rate()

  ```
  ### 求过去一天的CPU平均利用率
  rate(container_cpu_usage_seconds_total_srinfo{group_name=~"$GroupName",image!="",name!~"k8s_POD.*"} [1d])
  ```

  - avg_over_time()

  ```
  ### 获取过去一天的内存平均使用量
  avg_over_time(container_memory_working_set_bytes_srinfo{group_name=~"$GroupName",image!="",name!~"k8s_POD.*"} [1d])
  ```
  
  - **通过pod label监控容器相关指标(来自cadvisor)**
  
    由于容器的指标采集来自cadvisor，所以在采集的指标label中并没有自定义的pod的标签，但是在监控过程中想要对容器指标做向上聚合。官方没有给的方案是采用联表查询
  
    ```
    sum(rate(container_cpu_usage_seconds_total{image!="",container!="POD",pod="$PodName"}[1m])) by (pod,container) * on (pod) group_left(label_bclever_id) kube_pod_labels{label_bclever_id=~"$ServiceName"}
    
    sum(rate(container_cpu_usage_seconds_total{image!="",container!="POD",pod=~"cv-infer-carrier-cv-face-det-0-cv-face-det-8485dd474-pcjht"}[1m])) by (pod,container) * on (pod) group_left(label_bclever_id) kube_pod_labels{label_bclever_id=~"cv-infer-carrier"}
    
    sum(kube_pod_info) by (namespace) * on (namespace) group_left(label_bclever_id) kube_pod_labels{label_bclever_id=~"czptest"}
    
    sum by(label_bclever_namespace)(container_memory_working_set_bytes{image!="",container!="POD"} * on (pod) group_left(label_bclever_namespace) kube_pod_labels{label_bclever_namespace=~"$NameSpace"})
    ```
  
    

### 3.3 grafana使用mysql，自定义sql语句

编辑sql语句

<img src="../images/image-20211114165219477.png" width = "50%"   height = "70%"  alt="图片名称" align=left />

选择table 然后点击apply

<img src="../images/image-20211114164924319.png" width = "50%"   alt="图片名称" align=left />

### 3.4 grafana权限管理

- 添加用户及用户组

  参考：https://blog.csdn.net/qq_34355232/article/details/106274551

- 为dashboard添加权限管理

  首先，Dashboard的权限是继承自所在**Folder**的权限，所以你会看到部分已有权限后面有一个小锁的标志，代表权限不能修改，如若修改，只能通过修改对应Folder的权限；也就是说，**所谓Grafana Dashboard的权限控制，即Folder(文件夹)的权限控制**；

  <img src="../images/202005131725420526.png" width = "60%"   height = "70%"  alt="图片名称" align=left />

  如果想要给单个dashboard添加相应权限，可在"Dashboard settings"--> "Permissions"-->“Add Permission”，为某用户或小组添加权限；

## 四、基于Prometheus的k8s集群信息监控

#### 1、部署

- 获取官方源码

```
git clone https://github.com/coreos/kube-prometheus.git
```

- 部署CRD和监控

```
# CRD
cd kube-prometheus/manifests/setup
kubectl apply -f .

cd kube-prometheus/manifests
kubectl apply -f .
```

查看CRD

kubectl get crd | grep coreos

```shell
alertmanagers.monitoring.coreos.com     2021-12-15T06:56:28Z
podmonitors.monitoring.coreos.com       2021-12-15T06:56:28Z
prometheuses.monitoring.coreos.com      2021-12-15T06:56:28Z
prometheusrules.monitoring.coreos.com   2021-12-15T06:56:28Z
servicemonitors.monitoring.coreos.com   2021-12-15T06:56:28Z
```

查看pod

kubectl get pod -n monitoring 

```shell
alertmanager-main-0                   2/2     Running   0          42d
alertmanager-main-1                   2/2     Running   0          42d
alertmanager-main-2                   2/2     Running   0          42d
kube-state-metrics-78b46c84d8-klllv   3/3     Running   0          42d
prometheus-adapter-5cd5798d96-kj6r5   1/1     Running   0          42d
prometheus-k8s-0                      3/3     Running   1          42d
prometheus-operator-99dccdc56-lf6vm   1/1     Running   0          42d
```

查看service

kubectl get svc -n monitoring

```sehll
alertmanager-main       ClusterIP   11.1.126.171   <none>        9093/TCP                     42d
alertmanager-operated   ClusterIP   None           <none>        9093/TCP,9094/TCP,9094/UDP   42d
kube-state-metrics      ClusterIP   None           <none>        8443/TCP,9443/TCP            42d
prometheus-adapter      ClusterIP   11.1.178.197   <none>        443/TCP                      42d
prometheus-k8s          NodePort    11.1.253.224   <none>        9090:8098/TCP                42d
prometheus-operated     ClusterIP   None           <none>        9090/TCP                     42d
prometheus-operator     ClusterIP   None           <none>        8080/TCP                     42d
```

- 修改Prometheus为NodePort：

```
prometheus-service.yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    prometheus: k8s
  name: prometheus-k8s
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - name: web
    port: 9090
    nodePort: 8098
    targetPort: 9090
  selector:
    app: prometheus
    prometheus: k8s
  sessionAffinity: ClientIP

```

Prometheus添加节点选择

 ```
prometheus-prometheus.yaml

kubernetes.io/hostname: hpc01.data.lianjia.com
 ```

#### 2、持久化存储

节点配置ChubaoFS CSI

https://chubaofs.readthedocs.io/zh_CN/latest/user-guide/csi-driver.html

部署chubao csi插件

```
kubectl apply -f deploy/csi-rbac.yaml
kubectl apply -f deploy/csi-controller-deployment.yaml
kubectl apply -f deploy/csi-node-daemonset.yaml
```

创建StorageClass

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: chubaofs-sc-pro
provisioner: csi.chubaofs.com
reclaimPolicy: Delete
parameters:
  masterAddr: "https://chubaofs-tz-prod-01.ke.com"
  owner: "AiPrometheus"
  consulAddr: "http://consul.cfs.com"
  logLevel: "info"
```



节点加label

```
chubaofs-csi-controller=enabled
chubaofs-csi-node=enabled
```



```
 prometheus-prometheus.yaml
 
 storage:                  #----添加持久化配置，指定StorageClass为上面创建的fast
    volumeClaimTemplate:
      spec:
        storageClassName: chubaofs-sc-test #---指定为fast
        resources:
          requests:
            storage: 200Gi
```



部署：

```
cd /home/work/zhanglei/kube-prometheus-0.3.0/manifests/setup && kubectl apply -f .
cd /home/work/zhanglei/kube-prometheus-0.3.0/manifests && kubectl apply -f .
```

- 监控集群外节点

```
添加需要监控的数据
cd /home/work/zhanglei/kube-prometheus-0.3.0/addtional_node

```

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: kubelet
  name: test-exporter
  #namespace: monitoring
  namespace: kube-system
spec:
  clusterIP: None
  ports:
  - name: port
    port: 10255
    targetPort: port

---
apiVersion: v1
kind: Endpoints
metadata:
  name: test-exporter
  #namespace: monitoring
  namespace: kube-system
  labels:
    k8s-app: kubelet
subsets:
- addresses:
  - ip: 10.200.20.201
  - ip: 10.201.101.42
  ports:
  - name: port
    port: 10255
    protocol: TCP

```

```
在serviceMonitor中添加需要监控的ep

  endpoints:
    - port: port
      interval: 30s
      scheme: http
      path: /metrics/cadvisor

```





### pod监控

### 容器监控


























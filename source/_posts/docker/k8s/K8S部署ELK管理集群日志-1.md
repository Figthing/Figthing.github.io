title: K8S部署ELK管理集群日志（一）
author: Figthing
tags:
  - docker
  - k8s
  - elk
categories:
  - docker
  - k8s
  - elk
date: 2021-01-15 10:53:00
---
> 前言：ELK是目前主流的日志解决方案，尤其是容器化集群的今天，ELK几乎是集群必备的一部分能力；ELK在K8S落地有多种组合模式：
> 比如：fluentd+ELK或filebeat+ELK或log-pilot+ELK
> 而本文采用的是功能更强大的后者：filebeat 采集--->logstash过滤加工--->ES存储与索引--->Kibana展示的方案。



### ElasticSearch 集群安装

要建立一个 Elastic 技术的监控栈，当然首先我们需要部署 ElasticSearch，它是用来存储所有的指标、日志和追踪的数据库，这里我们通过3个不同角色的可扩展的节点组成一个集群。


<!-- more -->

#### 安装 ElasticSearch 主节点

设置集群的第一个节点为 Master 主节点，来负责控制整个集群。首先创建一个 ConfigMap 对象，用来描述集群的一些配置信息，以方便将 ElasticSearch 的主节点配置到集群中并开启安全认证功能。对应的资源清单文件如下所示：

```yaml
# elasticsearch-master.configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: kube-system
  name: elasticsearch-master-config
  labels:
    app: elasticsearch
    role: master
data:
  elasticsearch.yml: |-
    cluster.name: ${CLUSTER_NAME}
    node.name: ${NODE_NAME}
    discovery.seed_hosts: ${NODE_LIST}
    cluster.initial_master_nodes: ${MASTER_NODES}

    network.host: 0.0.0.0

    node:
      master: true
      data: false
      ingest: false

	# 是否开启验证
    xpack.security.enabled: true
    xpack.monitoring.collection.enabled: true
```

然后创建一个 Service 对象，在 Master 节点下，我们只需要通过用于集群通信的 9300 端口进行通信。资源清单文件如下所示：

```yaml
# elasticsearch-master.service.yaml
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: elasticsearch-master
  labels:
    app: elasticsearch
    role: master
spec:
  ports:
  - port: 9300
    name: transport
  selector:
    app: elasticsearch
    role: master
```

最后使用一个 Deployment 对象来定义 Master 节点应用，资源清单文件如下所示：

```yaml
# elasticsearch-master.deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: kube-system
  name: elasticsearch-master
  labels:
    app: elasticsearch
    role: master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
      role: master
  template:
    metadata:
      labels:
        app: elasticsearch
        role: master
    spec:
      containers:
      - name: elasticsearch-master
        image: registry.cn-chengdu.aliyuncs.com/zhouqi-kubernetes/es:7.6.1
        env:
        - name: CLUSTER_NAME
          value: elasticsearch
        - name: NODE_NAME
          value: elasticsearch-master
        - name: NODE_LIST
          value: elasticsearch-master,elasticsearch-data,elasticsearch-client
        - name: MASTER_NODES
          value: elasticsearch-master
        - name: "ES_JAVA_OPTS"
          value: "-Xms512m -Xmx512m"
        ports:
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: config
          mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
          readOnly: true
          subPath: elasticsearch.yml
        - name: storage
          mountPath: /data
      volumes:
      - name: config
        configMap:
          name: elasticsearch-master-config
      - name: "storage"
        emptyDir:
          medium: ""
```

#### 安装 ElasticSearch 数据节点

现在我们需要安装的是集群的数据节点，它主要来负责集群的数据托管和执行查询。和 master 节点一样，我们使用一个 ConfigMap 对象来配置我们的数据节点：

```yaml
# elasticsearch-data.configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: kube-system
  name: elasticsearch-data-config
  labels:
    app: elasticsearch
    role: data
data:
  elasticsearch.yml: |-
    cluster.name: ${CLUSTER_NAME}
    node.name: ${NODE_NAME}
    discovery.seed_hosts: ${NODE_LIST}
    cluster.initial_master_nodes: ${MASTER_NODES}

    network.host: 0.0.0.0

    node:
      master: false
      data: true
      ingest: false

	# 是否开启验证
    xpack.security.enabled: true
    xpack.monitoring.collection.enabled: true
```

可以看到和上面的 master 配置非常类似，不过需要注意的是属性 node.data=true。

同样只需要通过 9300 端口和其他节点进行通信：

```yaml
# elasticsearch-data.service.yaml
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: elasticsearch-data
  labels:
    app: elasticsearch
    role: data
spec:
  ports:
  - port: 9300
    name: transport
  selector:
    app: elasticsearch
    role: data
```

最后创建一个 StatefulSet 的控制器，因为可能会有多个数据节点，每一个节点的数据不是一样的，需要单独存储，所以也使用了一个 volumeClaimTemplates 来分别创建存储卷，对应的资源清单文件如下所示：

```yaml
# NFS挂载
#kind: PersistentVolumeClaim
#apiVersion: v1
#metadata:
#  name: elasticsearch-data-claim
#  namespace: kube-system
#  annotations:
#    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
#spec:
#  storageClassName: managed-nfs-storage
#  accessModes:
#    - ReadWriteMany
#  resources:
#    requests:
#      storage: 1G
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: kube-system
  name: elasticsearch-data
  labels:
    app: elasticsearch
    role: data
spec:
  serviceName: "elasticsearch-data"
  selector:
    matchLabels:
      app: elasticsearch
      role: data
  template:
    metadata:
      labels:
        app: elasticsearch
        role: data
    spec:
      containers:
      - name: elasticsearch-data
        image: registry.cn-chengdu.aliyuncs.com/zhouqi-kubernetes/es:7.6.1
        env:
        - name: CLUSTER_NAME
          value: elasticsearch
        - name: NODE_NAME
          value: elasticsearch-data
        - name: NODE_LIST
          value: elasticsearch-master,elasticsearch-data,elasticsearch-client
        - name: MASTER_NODES
          value: elasticsearch-master
        - name: "ES_JAVA_OPTS"
          value: "-Xms1024m -Xmx1024m"
        ports:
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: config
          mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
          readOnly: true
          subPath: elasticsearch.yml
        - name: elasticsearch-data-persistent-storage
          mountPath: /data/db          
      volumes:
      - name: config
        configMap:
          name: elasticsearch-data-config
# NFS挂载          
#      - name: elasticsearch-data-persistent-storage
#        persistentVolumeClaim:
#          claimName: elasticsearch-data-claim
  volumeClaimTemplates:
  - metadata:
      name: elasticsearch-data-persistent-storage
      annotations:
        volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi   
```

#### 安装 ElasticSearch 客户端节点

最后来安装配置 ElasticSearch 的客户端节点，该节点主要负责暴露一个 HTTP 接口将查询数据传递给数据节点获取数据。同样使用一个 ConfigMap 对象来配置该节点：

```yaml
# elasticsearch-client.configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: kube-system
  name: elasticsearch-client-config
  labels:
    app: elasticsearch
    role: client
data:
  elasticsearch.yml: |-
    cluster.name: ${CLUSTER_NAME}
    node.name: ${NODE_NAME}
    discovery.seed_hosts: ${NODE_LIST}
    cluster.initial_master_nodes: ${MASTER_NODES}

    network.host: 0.0.0.0

    node:
      master: false
      data: false
      ingest: true

    xpack.security.enabled: true
    xpack.monitoring.collection.enabled: true
```

客户端节点需要暴露两个端口，9300端口用于与集群的其他节点进行通信，9200端口用于 HTTP API。对应的 Service 对象如下所示：

```yaml
# elasticsearch-client.service.yaml
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: elasticsearch-client
  labels:
    app: elasticsearch
    role: client
spec:
  ports:
  - port: 9200
    name: client
    nodePort: 30008   # NodePort     
  - port: 9300
    name: transport
  type: NodePort       
  selector:
    app: elasticsearch
    role: client
```

使用一个 Deployment 对象来描述客户端节点：

```yaml
# elasticsearch-client.deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: kube-system
  name: elasticsearch-client
  labels:
    app: elasticsearch
    role: client
spec:
  selector:
    matchLabels:
      app: elasticsearch
      role: client
  template:
    metadata:
      labels:
        app: elasticsearch
        role: client
    spec:
      containers:
      - name: elasticsearch-client
        image: registry.cn-chengdu.aliyuncs.com/zhouqi-kubernetes/es:7.6.1
        env:
        - name: CLUSTER_NAME
          value: elasticsearch
        - name: NODE_NAME
          value: elasticsearch-client
        - name: NODE_LIST
          value: elasticsearch-master,elasticsearch-data,elasticsearch-client
        - name: MASTER_NODES
          value: elasticsearch-master
        - name: "ES_JAVA_OPTS"
          value: "-Xms256m -Xmx256m"
        ports:
        - containerPort: 9200
          name: client
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: config
          mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
          readOnly: true
          subPath: elasticsearch.yml
        - name: storage
          mountPath: /data
      volumes:
      - name: config
        configMap:
          name: elasticsearch-client-config
      - name: "storage"
        emptyDir:
          medium: ""
```

#### 生成密码
我们启用了 xpack 安全模块来保护我们的集群，所以我们需要一个初始化的密码。我们可以执行如下所示的命令，在客户端节点容器内运行 `bin/elasticsearch-setup-passwords` 命令来生成默认的用户名和密码：

```shell
$ kubectl exec $(kubectl get pods -n kube-system | grep elasticsearch-client | sed -n 1p | awk '{print $1}') \
    -n elastic \
    -- bin/elasticsearch-setup-passwords auto -b

Changed password for user apm_system
PASSWORD apm_system = 3Lhx61s6woNLvoL5Bb7t

Changed password for user kibana_system
PASSWORD kibana_system = NpZv9Cvhq4roFCMzpja3

Changed password for user kibana
PASSWORD kibana = NpZv9Cvhq4roFCMzpja3

Changed password for user logstash_system
PASSWORD logstash_system = nNnGnwxu08xxbsiRGk2C

Changed password for user beats_system
PASSWORD beats_system = fen759y5qxyeJmqj6UPp

Changed password for user remote_monitoring_user
PASSWORD remote_monitoring_user = mCP77zjCATGmbcTFFgOX

Changed password for user elastic
PASSWORD elastic = wmxhvsJFeti2dSjbQEAH
```

注意需要将 elastic 用户名和密码也添加到 Kubernetes 的 Secret 对象中：

```shell
$ kubectl create secret generic elasticsearch-pw-elastic \
    -n kube-system \
    --from-literal password=wmxhvsJFeti2dSjbQEAH
secret/elasticsearch-pw-elastic created
```
#### 常用查询
- 校验节点是否正常`http://ip:9200/_cat/nodes?pretty`
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/es/1.png)

- 查看ES信息`http://ip:9200
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/es/2.png)
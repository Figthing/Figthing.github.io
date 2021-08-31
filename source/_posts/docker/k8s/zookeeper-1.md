title: K8s搭建-Zookeeper集群
author: Figthing
tags:
  - docker
  - k8s
categories:
  - docker
  - k8s
date: 2019-06-28 15:18:00
---
## K8S搭建Zookeeper集群

### 服务器资源

| 服务器地址  | k8s        | NFS     |
| ----------- | ---------- | ------- |
| 172.24.2.67 | k8s-node-3 | service |
| 172.24.2.69 | k8s-master | client  |
| 172.24.2.70 | k8s-node-2 | client  |
| 172.24.2.71 | k8s-node-1 | client  |



### 安装NFS

1、在67服务器中安装NFS

```shell
#master节点安装nfs
yum -y install nfs-utils

#创建nfs目录
mkdir -p /nfs/data/

#修改权限
chmod -R 777 /nfs/data

#编辑export文件,这个文件就是nfs默认的配置文件
vi /etc/exports
/nfs/data *(rw,no_root_squash,sync)

#配置生效
exportfs -r
#查看生效
exportfs

#启动rpcbind、nfs服务
systemctl restart rpcbind && systemctl enable rpcbind
systemctl restart nfs && systemctl enable nfs

#查看 RPC 服务的注册状况
rpcinfo -p localhost

#showmount测试
showmount -e 172.24.2.67
```

![1561700173921](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/1561700173921.png)

2、所有node节点安装客户端，开机启动

```shell
yum -y install nfs-utils
systemctl start nfs && systemctl enable nfs
```

<!--more-->

### 部署NSF-Zookeeper-PV

1、新建一个`zk-nfs-pv.yaml`文件，内容如下：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zk-nfs-pv-01
  labels:
    pv: zk-nfs-pv-01
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: zk-nfs
  nfs:
    path: /nfs/data/zookeeper/pv-01
    server: 172.24.2.67
---    
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zk-nfs-pv-02
  labels:
    pv: zk-nfs-pv-02
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: zk-nfs
  nfs:
    path: /nfs/data/zookeeper/pv-02
    server: 172.24.2.67
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zk-nfs-pv-03
  labels:
    pv: zk-nfs-pv-03
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: zk-nfs
  nfs:
    path: /nfs/data/zookeeper/pv-03
    server: 172.24.2.67
```

2、启动zk-nfs-pv

```shell
kubectl create -f zk-nfs-pv.yaml
```

3、查看状态

![1561700126674](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/1561700126674.png)

### 部署Zookeeper集群

1、新建文件`zookeeper.yaml`，内容如下：

```yaml
apiVersion: v1
kind: Service
metadata:  
  name: zk-headless
  labels:
    app: zk-headless  
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
---
apiVersion: v1
kind: Service
metadata:  
  name: zk-cs
  labels:
    app: zk
spec:
  ports:
  - port: 2181
    name: client
    nodePort: 30002
  selector:
    app: zk    
  type: NodePort 
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: zk-config
data:
  ensemble: "zk-0;zk-1;zk-2"
  jvm.heap: "2G"
  tick: "2000"
  init: "10"
  sync: "5"
  client.cnxns: "60"
  snap.retain: "3"
  purge.interval: "1"
---
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-budget
spec:
  selector:
    matchLabels:
      app: zk
  minAvailable: 2
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zk
spec:
  selector:
    matchLabels:
      app: zk
  serviceName: zk-headless
  replicas: 3
  template:
    metadata:
      labels:
        app: zk
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values: 
                    - zk
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: kubernetes-zookeeper
        imagePullPolicy: Always
        image: "registry.cn-shenzhen.aliyuncs.com/zhouqi-kubernetes/k8szk:v3"
        resources:
          limits:
            cpu: 1
            memory: 1Gi
          requests:
            memory: 512Mi
            cpu: 500m
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        env:
        - name : ZK_REPLICAS
          value: "3"
        - name : ZK_ENSEMBLE
          valueFrom:
            configMapKeyRef:
              name: zk-config
              key: ensemble
        - name : ZK_HEAP_SIZE
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: jvm.heap
        - name : ZK_TICK_TIME
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: tick
        - name : ZK_INIT_LIMIT
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: init
        - name : ZK_SYNC_LIMIT
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: tick
        - name : ZK_MAX_CLIENT_CNXNS
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: client.cnxns
        - name: ZK_SNAP_RETAIN_COUNT
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: snap.retain
        - name: ZK_PURGE_INTERVAL
          valueFrom:
            configMapKeyRef:
                name: zk-config
                key: purge.interval
        - name: ZK_CLIENT_PORT
          value: "2181"
        - name: ZK_SERVER_PORT
          value: "2888"
        - name: ZK_ELECTION_PORT
          value: "3888"
        command:
        - sh
        - -c
        - zkGenConfig.sh && zkServer.sh start-foreground
        readinessProbe:
          exec:
            command:
            - "zkOk.sh"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        livenessProbe:
          exec:
            command:
            - "zkOk.sh"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/zookeeper
          subPath: data/
        - name: datadir
          mountPath: /opt/zookeeper/conf
          subPath: config/
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
  volumeClaimTemplates:
  - metadata:
      name: datadir
      annotations:
        volume.beta.kubernetes.io/storage-class: "zk-nfs"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 3Gi
```

2、启动`zookeeper.yaml`

```shell
kubectl create -f zookeeper.yaml
```

3、查看启动情况

```shell
kubectl get pod -o wide
```

![1561700410724](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/1561700410724.png)

4、最后来验证Zookeeper集群是否正常，查看集群节点状态

```shell
for i in 0 1 2; do kubectl exec zk-$i zkServer.sh status; done
```

![1561700478499](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/1561700478499.png)



参考文献：<https://www.jianshu.com/p/2633b95c244c>

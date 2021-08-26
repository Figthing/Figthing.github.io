title: K8s搭建-YAPI
author: Figthing
tags:
  - docker
  - k8s
categories:
  - docker
  - k8s
date: 2019-07-18 12:01:00
---
## YAPI搭建

### 概述

YApi 是一个可本地部署的、打通前后端及QA的、可视化的接口管理平台



### 准备工作

1、准备Yapi镜像

2、准备Mongo镜像

3、创建NFS共享文件

这些准备工作，都在我的博客中有写道，有疑问可以去找一下 <http://blog.appydm.com>

<!--more-->


### 安装/启动(Mongo)

#### Mongo-K8S配置

mongo-nfs-pv.yaml文件内容

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongo-nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /nfs/data/mongo
    server: 172.24.2.70
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-nfs-pvc
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi    
```

mongo.yaml文件内容

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    app: mongo
  name: mongo
spec:
  ports:
    - port: 19098
      targetPort: 27017
  selector:
      app: mongo
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mongo
  labels:
    app: mongo
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: mongo
    spec:
      initContainers:
      - name: 'busyboxplus'
        image: registry.cn-shenzhen.aliyuncs.com/zhouqi-kubernetes/busyboxplus:v1
        command: [
          "sh", 
          "-c", 
          "nslookup mysql-system.default.svc.cluster.local"
        ]
      containers:
      - name: mongo      
        image: mongo
        imagePullPolicy: Always
        resources:
          limits:
            cpu: 2
            memory: 2Gi
          requests:
            memory: "512Mi"
            cpu: "200m"            
        ports:
        - containerPort: 27017
        env:
          - name: "MONGO_INITDB_ROOT_USERNAME"
            value: "root"
          - name: "MONGO_INITDB_ROOT_PASSWORD"
            value: "48660960"
        volumeMounts:
        - name: datadir
          mountPath: /data/db
          subPath: data
      volumes:
        - name: datadir
          persistentVolumeClaim:
            claimName: mongo-nfs-pvc
```

#### Mongo-启动

```shell
kubectl apply -f mongo-nfs-pv.yaml
```

```shell
kubectl apply -f mongo.yaml
```



### 安装/启动(YAPI)

#### 创建YAPI用户和数据库

进入mongo容器中，执行下面命令

```shell
root@mongo-69f8cbc957-xw47b:/# mongo
MongoDB shell version v4.0.10
connecting to: mongodb://127.0.0.1:27017/?gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("d333ace7-d3e7-40bd-9d41-81a72b12771d") }
MongoDB server version: 4.0.10
> use admin
switched to db admin
> db.auth("root", "48660960")
1
> use yapi
> db.createUser({user:"yapi",pwd:"yapi",roles:[{"role":"readWrite","db":"yapi"}]})
```

#### 创建YAPI的config.json配置

```shell
cat << EOF > config.json
> 
{
  "port": "3000",
  "adminAccount": "admin@admin.com",
  "db": {
    "servername": "mongo.default.svc.cluster.local",
    "DATABASE":  "yapi",
    "port": 19098,
    "user": "yapi",
    "pass": "yapi",
    "authSource": ""
  },
  "mail": {
    "enable": true,
    "host": "smtp.163.com",
    "port": 465,
    "from": "***@163.com",
    "auth": {
        "user": "***@163.com",
        "pass": "*****"
    }
  }
}
> EOF
```

#### YAPI-K8S配置

yapi.yaml文件

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    app: yapi
  name: yapi
spec:
  ports:
    - port: 19080
      targetPort: 3000
  selector:
      app: yapi
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: yapi
  labels:
    app: yapi
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: yapi
    spec:
      initContainers:
      - name: 'busyboxplus'
        image: registry.cn-shenzhen.aliyuncs.com/zhouqi-kubernetes/busyboxplus:v1
        command: [
          "sh", 
          "-c", 
          "nslookup mongo.default.svc.cluster.local"
        ]
      containers:           
      - name: yapi      
        image: registry.cn-shenzhen.aliyuncs.com/zhouqi-kubernetes/yapi:v1.7.2
        imagePullPolicy: Always
        resources:
          limits:
            cpu: 1
            memory: 1Gi
          requests:
            memory: "512Mi"
            cpu: "200m"            
        ports:
        - containerPort: 3000
        command: [
          "/bin/bash",
          "-c",
          "[ ! -e /home/yapi/log/init.lock ] && npm run install-server && touch /home/yapi/log/init.lock; npm run start"          
        ]
        volumeMounts:
        - name: yapi-nfs
          mountPath: /home/yapi/config.json
          subPath: config.json
      volumes:
      - name: yapi-nfs
        nfs:
          server: 172.24.2.71
          path: /nfs/data/yapi   
```

#### YAPI-启动

```shell
kubectl apply -f yapi.yaml
```
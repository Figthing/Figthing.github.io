title: K8S部署ELK管理集群日志（二）
author: Figthing
tags:
  - docker
  - k8s
  - elk
categories:
  - docker
  - k8s
  - elk
date: 2021-01-15 15:15:00
---

> 上一篇文章讲述了k8s安装ES集群的方式，本章接着讲解K8S安装Kibana

### Kibana安装

ElasticSearch 集群安装完成后，接着我们可以来部署 Kibana，这是 ElasticSearch 的数据可视化工具，它提供了管理 ElasticSearch 集群和可视化数据的各种功能。

同样首先我们使用 ConfigMap 对象来提供一个文件文件，其中包括对 ElasticSearch 的访问（主机、用户名和密码），这些都是通过环境变量配置的。对应的资源清单文件如下所示：

<!--more-->

```yaml
# kibana.configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  labels:
    app: kibana
data:
  kibana.yml: |-
    server.host: 0.0.0.0
# 中文
#    i18n.locale: "zh-CN"

    elasticsearch:
      hosts: ${ELASTICSEARCH_HOSTS}
      username: ${ELASTICSEARCH_USER}
      password: ${ELASTICSEARCH_PASSWORD}
```

然后通过一个 NodePort 类型的服务来暴露 Kibana 服务：

```yaml
# kibana.service.yaml

apiVersion: v1
kind: Service
metadata:
  name: kibana
  labels:
    app: kibana
spec:
  type: NodePort
  ports:
  - port: 5601
    name: webinterface
  selector:
    app: kibana
```

最后通过 Deployment 来部署 Kibana 服务，由于需要通过环境变量提供密码，这里我们使用上面创建的 Secret 对象来引用：

```yaml
# kibana.deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: elastic
  name: kibana
  labels:
    app: kibana
spec:
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: registry.cn-chengdu.aliyuncs.com/zhouqi-kubernetes/kibana:7.6.1
        ports:
        - containerPort: 5601
          name: webinterface
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch-client.kube-system.svc.cluster.local:9200"
        - name: ELASTICSEARCH_USER
          value: "elastic"
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-pw-elastic
              key: password
        volumeMounts:
        - name: config
          mountPath: /usr/share/kibana/config/kibana.yml
          readOnly: true
          subPath: kibana.yml
      volumes:
      - name: config
        configMap:
          name: kibana-config
```

如下图所示，使用上面我们创建的 Secret 对象的 elastic 用户和生成的密码即可登录：
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/kibana/1.png)

登录成功后会自动跳转到 Kibana 首页：
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/kibana/2.png)

同样也可以自己创建一个新的超级用户，Management → Stack Management → Create User：
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/kibana/3.png)

使用新的用户名和密码，选择 superuser 这个角色来创建新的用户：
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/kibana/4.png)

创建成功后就可以使用上面新建的用户登录 Kibana，最后还可以通过 Management → Stack Monitoring 页面查看整个集群的健康状态：
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/kibana/5.png)

到这里我们就安装成功了 ElasticSearch 与 Kibana，它们将为我们来存储和可视化我们的应用数据（监控指标、日志和追踪）服务。

在下一篇文章中，我们将来学习如何安装和配置Logstash与Filebeat，通过 Filebeat 来收集指标监控 Spring Boot中的日志。
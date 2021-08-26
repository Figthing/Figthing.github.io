title: K8S部署ELK管理集群日志（三）
author: Figthing
tags:
  - docker
  - k8s
  - elk
categories:
  - docker
  - k8s
  - elk
date: 2021-01-15 15:29:00
---
> 
>
> 上一篇文章讲述了k8s安装Kibana，本章接着讲解K8S安装Logstash与Filebeat

### Logstash和Filebeat安装

在使用k8s安装Logstash时，先在本地做测试配置，如果已经非常熟悉的，就可以将这个步骤省略

#### Windows安装Logstash

1. 去官方下载Windows版本的[Logstash](https://www.elastic.co/cn/downloads/past-releases/logstash-7-6-1)，将ZIP文件解压，变成下面的目录结构

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/logstash/1.png" style="zoom: 67%;" />

<!--more-->

2. 进入bin文件夹中新建一个logstash.conf配置文件，并进行编辑，内容如下

```yaml
# 参考官方文档：https://www.elastic.co/guide/en/logstash/current/plugins-inputs-beats.html
# 输入
input {	
    beats{
		# 接受数据端口
		port => "5044"
		# 主要用于筛选器激活，字符串
		# type => "logs"
		# 额外添加字段，这里是为了区分来自哪一个插件
		# add_field => {"[fields][class]" => "beats"}
    }
	#这个插件需要和filebeat进行配很这里不做多讲，到时候结合起来一起介绍。
} 

# 输出
output {
    stdout{
		# 使用 ruby "awesome_print" 库输出事件数据这是 stdout 的默认编解码器。
		codec => rubydebug
    }
	elasticsearch {
		# 要用于查询的elasticsearch主机的列表
		hosts => "elasticsearch-client客户端IP:elasticsearch-client端口" 
		
		# 要搜索的索引名称的逗号列表;使用 或空字符串对所有索引执行操作。
		index => "log-%{+YYYY.MM.dd}"
	}
}
```

3. 执行logstash

```shell
logstash.bat -f logstash.conf
```

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/logstash/2.png" style="zoom: 67%;" />

可以通过9600端口访问页面，查看是否执行成功
<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/logstash/3.png" style="zoom: 67%;" />

#### Windows安装Filebeat

1. 去官方下载Windows版本的[Filebeat](https://www.elastic.co/cn/downloads/past-releases/filebeat-7-6-1)，将ZIP文件解压，变成下面的目录结构

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/filebeat/1.png" style="zoom: 67%;" />

2. 在根目录下编辑`filebeat.yml`文件，因我们是先将数据传递给Logstash，所以不用配置es的地址信息，注释掉就可以了

```yml
# 参考官方文档： https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-log.html
filebeat.inputs:

- type: log

  # 开启监视，不开不采集
  enabled: true

  paths:
    # 指定需要收集的日志文件的路径
    - G:\*\info\*.log
    
output.logstash:
  hosts: ["localhost:5044"]    
```

3. 执行

```powershell
filebeat -e -c filebeat.yml -d "publish"
```

<img src="http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/filebeat/2.png" style="zoom: 67%;" />

#### Windows测试ELK

1. 在Filebeat监听目录下新增一个*.log文件，并写入内容，并观察

<img src="http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/filebeat/3.png" style="zoom: 67%;" />

上面图片中，我们已经能够看见对监听的文件内容变更，进行push到logstash中

2. 我们在观察一下logstash的情况

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/logstash/4.png" style="zoom: 67%;" />

看见已经接收到传过来的信息了，然后将数据提交到ES中了

3. 我们在Kibana中查看我们的索引

<img src="http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/kibana/6.png" style="zoom: 67%;" />

4. 创建索引模式

<img src="http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/kibana/7.png" style="zoom: 67%;" />

<img src="http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/kibana/8.png" style="zoom: 67%;" />

5. 索引模式创建成功后，我们可以去面板看一下数据了

<img src="http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/kibana/9.png" style="zoom: 67%;" />

到这里，ELK的集成就基本结束了。后续是K8S部署Logstash和Spring项目中的使用



#### K8s安装Logstash安装

1. 在NFS目录下新建一个test.conf文件，并写入内容

```yaml
# test.conf
input {
    beats{
                port => "5044"
    }
}

output {
    stdout{
                codec => rubydebug
    }
        elasticsearch {
                hosts => "{elasticsearch-client IP地址}:9200"
                index => "log-%{+YYYY.MM.dd}"
        }
}
```

2. 执行下面yaml文件

```yaml
# logstash.yaml

kind: Service
apiVersion: v1
metadata:
  labels:
    app: logstash
    elk: logstash-service
  name: logstash-service-svc
spec:
  ports:
    - name: web
      port: 9600
      targetPort: web  
    - name: tcp
      port: 5044      
      targetPort: tcp        
  selector:
    elk: logstash-dep
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: logstash-service-dep
  labels:
    app: logstash
    elk: logstash-dep
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: logstash
        elk: logstash-dep
    spec:
      containers:
        - name: logstash
          image: logstash:7.6.1
          imagePullPolicy: Always
          resources:
            limits:
              cpu: 1
              memory: 1Gi
            requests:
              memory: "512Mi"
              cpu: "200m"
          ports:
            - name: web
              containerPort: 9600
            - name: tcp
              containerPort: 5044
          volumeMounts:
          - name: logstash-nfs
            mountPath: /usr/share/logstash/config/conf.d/
          - name: config
            mountPath: /usr/share/logstash/config/logstash.yml
            readOnly: true
            subPath: logstash.yml            
      volumes:
      - name: logstash-nfs
        nfs:
          server: #NFS IP地址
          path: /nfs/data/logstash    
      - name: config
        configMap: 
          name: logstash-config          
---                  
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  labels:
    app: logstash
    elk: logstash-service
data:
  logstash.yml: |-
    http.host: "0.0.0.0"
    xpack.monitoring.elasticsearch.hosts: [ "http://{elasticsearch-client IP地址}:9200" ]
    path.config: "/usr/share/logstash/config/conf.d/*.conf"
```

3. 查看启动情况

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/logstash/5.png" style="zoom: 67%;" />
# 二十六 项目实战

## 26.1  ElasticSearch深度集成kubernetes 6.5

### 安装和配置

##### 自制带插件的ES镜像

```dockerfile
FROM elasticsearch:6.5.0
#或者手动下载后然后安装也行
# COPY elasticsearch-analysis-ik-6.5.0.zip /
# elasticsearch-plugin install --batch file:///elasticsearch-analysis-ik-6.5.0.zip
    #IK Analyzer是一个开源的，基于java语言开发的中文分词工具包。是开源社区中处理中分分词非常热门的插件。 
RUN elasticsearch-plugin install --batch https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.5.0/elasticsearch-analysis-ik-6.5.0.zip && \
    # 拼音分词器
    elasticsearch-plugin install --batch https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v6.5.0/elasticsearch-analysis-pinyin-6.5.0.zip && \    
    # Smart Chinese Analysis Plugin
    elasticsearch-plugin install analysis-icu && \
    # 日文分词器
    elasticsearch-plugin install analysis-kuromoji && \
    # 语音分析
    elasticsearch-plugin install analysis-phonetic && \
    # 计算字符哈希
    elasticsearch-plugin install  mapper-murmur3 && \
    # 在_source中提供size字段
    elasticsearch-plugin install mapper-size
```

#### 编排文件

- RBAC 相关内容

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: elasticsearch-admin
  namespace: default
  labels:
    app: elasticsearch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: elasticsearch
  labels:
    app: elasticsearch
subjects:
- kind: ServiceAccount
  name: elasticsearch-admin
  namespace: default
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: elasticsearch
  apiGroup: ""
```

- 主体程序


存储使用了hostpath,需要先在宿主机闯将目录,并赋予适当的权限,不然会出错

```bash
cd /root/kubernetes/$(namespace)/elasticsearch/data
mkdir -p $(pwd)
sudo chmod 775  $(pwd) -R
chown 1000:0  $(pwd) -R
```

```
# kubectl get po -l app=myelasticsearch -o wide  -n default
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myelasticsearch
  namespace: default
  labels:
    app: myelasticsearch    
    elasticsearch-role: all
spec:
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
  serviceName: elasticsearch-master
  replicas: 2
  selector:
    matchLabels:
      app: myelasticsearch
      elasticsearch-role: all
  template:
    metadata:
      labels:
        app: myelasticsearch                     
        elasticsearch-role: all 
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: elasticsearch-test-ready
                operator: Exists
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:              
              - key: 'app'
                operator: In
                values:
                - myelasticsearch
              - key: 'elasticsearch-role'
                operator: In
                values:
                - all                 
            topologyKey: "kubernetes.io/hostname"
            namespaces: 
            - default    
      serviceAccountName: elasticsearch-admin    
      terminationGracePeriodSeconds: 180
      # Elasticsearch requires vm.max_map_count to be at least 262144.
      # If your OS already sets up this number to a higher value, feel free
      # to remove this init container.
      initContainers:
      - image: alpine:3.6
        command: ["/sbin/sysctl", "-w", "vm.max_map_count=262144"]
        name: elasticsearch-init
        securityContext:
          privileged: true      
      imagePullSecrets:
        - name: vpc-shenzhen      
      containers:
      - image: elasticsearch:6.5.0-plugin-in-remote-ik
        name: elasticsearch
        resources:
          # need more cpu upon initialization, therefore burstable class
          limits:
            # cpu: 2
            memory: 4Gi
          requests:
            # cpu: 1
            memory: 1Gi
        ports:
        - name: restful
          containerPort: 9200          
          protocol: TCP
        - name: discovery
          containerPort: 9300          
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          initialDelaySeconds: 2
          periodSeconds: 10
          successThreshold: 1
          tcpSocket:
            port: 9200
          timeoutSeconds: 1     
        # livenessProbe:
        #   failureThreshold: 3
        #   initialDelaySeconds: 7
        #   periodSeconds: 10
        #   successThreshold: 1
        #   tcpSocket:
        #     port: 9200
        #   timeoutSeconds: 1           
        volumeMounts:
        - name: host
          mountPath: /usr/share/elasticsearch/data
        env:
        - name: "NAMESPACE"
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        # - name: "ES_JAVA_OPTS"
        #   value: "-Xms256m -Xmx256m"              
        - name: "cluster.name"
          value: "myelasticsearch"
        - name: "bootstrap.memory_lock"
          value: "true"
        - name: "discovery.zen.ping.unicast.hosts"
          value: "myelasticsearch"
        - name: "discovery.zen.minimum_master_nodes"
          value: "2"
        - name: "discovery.zen.ping_timeout"
          value: "5s"
          #因为是测试,所以master,data,ingest都混用          
        - name: "node.master"
          value: "true"
        - name: "node.data"
          value: "true"
        - name: "node.ingest"
          value: "true"
        - name: xpack.monitoring.collection.enabled
          value: "true" 
        - name: node.name
          valueFrom: 
            fieldRef:
              fieldPath: metadata.name 
              # fieldPath: spec.nodeName 
        securityContext: 
          privileged: true
      volumes:
      - name: host
        hostPath:
          path: /root/kubernetes/default/myelasticsearch/data
          type: DirectoryOrCreate
---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: myelasticsearch
    elasticsearch-role: all
  name: myelasticsearch
  namespace: default
spec:
  ports:
    - name: discovery
      port: 9300
      targetPort: discovery
    - name: restful
      port: 9200
      protocol: TCP
      targetPort: restful
  selector:
    app: myelasticsearch
    elasticsearch-role: all
  type: NodePort
```

这个编排精髓的一点在于用了节点`affinity`使每一个节点最多会运行一个容器,确保了高可用.

如果要把节点的角色再抽取出来,那么其实抽取一个service作为相互发现的,即可.


```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    app: elasticsearch
  name: elasticsearch-discovery
  namespace: default
spec:
  ports:
    - port: 9300
      targetPort: discovery
  selector:
    app: myelasticsearch
```


1. [在Kubernetes上部署Elasticsearch集群](https://blog.csdn.net/chenleiking/article/details/79453460)
1. [重要配置的修改](https://www.elastic.co/guide/cn/elasticsearch/guide/current/important-configuration-changes.html)
1. [Install Elasticsearch with Docker](https://www.elastic.co/guide/en/elasticsearch/reference/6.x/docker.html)
1. [学习Elasticsearch之4：配置一个3节点Elasticsearch集群(不区分主节点和数据节点)](http://ethancai.github.io/2016/08/06/configure-smallest-elasticsearch-cluster/)
1. [谈一谈Elasticsearch的集群部署](https://blog.csdn.net/zwgdft/article/details/54585644)
1. [官方docker镜像构建项目](https://github.com/elastic/elasticsearch-docker/tree/master/templates)
1. [Elasticsearch模块功能之-自动发现（Discovery）](https://blog.csdn.net/changong28/article/details/38377863)
1. [订阅费用](https://www.elastic.co/subscriptions)
2. [故障转移](https://es.xiaoleilu.com/020_Distributed_Cluster/20_Add_failover.html)
3. [节点配置](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#coordinating-node)
4. [Elasticsearch 5.X集群多节点角色配置深入详解](https://blog.csdn.net/laoyang360/article/details/78290484)
1. [elasticsearch-cloud-kubernetes](https://github.com/fabric8io/elasticsearch-cloud-kubernetes)
1. [吃透Elasticsearch 堆内存](https://blog.csdn.net/laoyang360/article/details/79998974)

#### 安装插件

[elasticsearch-docker-plugin-management](https://www.elastic.co/blog/elasticsearch-docker-plugin-management)

GET /_cat/plugins?v&s=component&h=name,component,version,description


### 压力测试

```
esrally configure
esrally list tracks
esrally --pipeline=benchmark-only --target-hosts=127.0.0.1:9200 --track=geonames
```

```
datastore.type = elasticsearch
datastore.host = 127.0.0.1
datastore.port = 9200
datastore.secure = False
```


通过 Elasticsearch 官方提供的 benchmark 脚本 rally


### ElasticSearch集群的维护


更新的时候务必使用灰度更新,从序号最大的镜像开始更新,不然分片丢失了,相信我,你会死的很惨

```bash
kubectl patch statefulset elasticsearch -p \
'{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":0}}}}' \
-n test

kubectl patch statefulset elasticsearch -p \
'{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":1}}}}' \
-n test

kubectl patch statefulset elasticsearch -p \
'{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}' \
-n test
```

## 26.2 kubernetes搭建consul


注意,`/consul/data`这个存储被我注释掉了,请按需自行配置相应的volume

主要思路就是先启动3台server,彼此之间通过`consul-server`实现自动加入节点.并通过反亲和度确保每个节点只允许一个consul-server.实现真正高可用.

然后启动`consul-client`,通过`consul-server`实现自动加入节点.

### server

```yml
apiVersion: v1
kind: Service
metadata:
  namespace: $(namespace)
  name: consul-server
  labels:
    name: consul-server
spec:
  ports:
    - name: http
      port: 8500
    - name: serflan-tcp
      protocol: "TCP"
      port: 8301
    - name: serfwan-tcp
      protocol: "TCP"
      port: 8302
    - name: server
      port: 8300
    - name: consuldns
      port: 8600
  selector:
    app: consul
    consul-role: server
---
# kgpo -l app=consul
# kgpo -l app=consul  -o  wide -w
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: consul-server
spec:
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
  serviceName: consul-server
  replicas: 3
  template:
    metadata:
      labels:
        app: consul
        consul-role: server
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: "kubernetes.io/hostname"
              namespaces: 
              - $(namespace)
              labelSelector:
                matchExpressions:
                - key: 'consul-role'
                  operator: In
                  values: 
                   - "server"
      terminationGracePeriodSeconds: 10
      securityContext:
        fsGroup: 1000
      containers:
        - name: consul
          image: "consul:1.4.2"
          imagePullPolicy: Always
          resources:
            requests:
              memory: 500Mi
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          args:
            - "agent"
            - "-advertise=$(POD_IP)"
            - "-bind=0.0.0.0"
            - "-bootstrap-expect=3"
            - "-retry-join=consul-server"
            - "-client=0.0.0.0"
            - "-datacenter=dc1"
            - "-data-dir=/consul/data"
            - "-domain=cluster.local"
            - "-server"
            - "-ui"
            - "-disable-host-node-id"
            - '-recursor=114.114.114.114'
        #   volumeMounts:
        #     - name: data
        #       mountPath: /consul/data
          lifecycle:
            preStop:
              exec:
                command:
                - /bin/sh
                - -c
                - consul leave
          ports:
            - containerPort: 8500
              name: ui-port
            - containerPort: 8400
              name: alt-port
            - containerPort: 53
              name: udp-port
            - containerPort: 8301
              name: serflan
            - containerPort: 8302
              name: serfwan
            - containerPort: 8600
              name: consuldns
            - containerPort: 8300
              name: server
#   volumeClaimTemplates:
#   - metadata:
#       name: data
```

### client

```yml

---    
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: consul-client
spec:
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
  serviceName: consul-client
  replicas: 10
  template:
    metadata:
      labels:
        app: consul
        consul-role: client
    spec:
      terminationGracePeriodSeconds: 10
      securityContext:
        fsGroup: 1000
      containers:
        - name: consul
          image: "consul:1.4.2"
          imagePullPolicy: Always
          resources:
            requests:
              memory: 500Mi
          env:
            - name: podname
              valueFrom: 
                fieldRef:
                  fieldPath: metadata.name
          args:
          - agent
          - -ui
          - -retry-join=consul-server
          - -node=$(podname)
          - -bind=0.0.0.0
          - -client=0.0.0.0
          - '-recursor=114.114.114.114'
        #   volumeMounts:
        #     - name: data
        #       mountPath: /consul/data
          lifecycle:
            preStop:
              exec:
                command:
                - /bin/sh
                - -c
                - consul leave
          readinessProbe:
            # NOTE(mitchellh): when our HTTP status endpoints support the
            # proper status codes, we should switch to that. This is temporary.
            exec:
              command:
                - "/bin/sh"
                - "-ec"
                - |
                  curl http://127.0.0.1:8500/v1/status/leader 2>/dev/null | \
                  grep -E '".+"'
          ports:
            - containerPort: 8301
              name: serflan
            - containerPort: 8500
              name: ui-port
            - containerPort: 8600
              name: consuldns
---
apiVersion: v1
kind: Service
metadata:
  namespace: $(namespace)
  name: consul-client
  labels:
    name: consul-client
    consul-role: consul-client
spec:
  ports:
    - name: serflan-tcp
      protocol: "TCP"
      port: 8301
    - name: http
      port: 8500
    - name: consuldns
      port: 8600
  selector:
    app: consul
    consul-role: client
```

### UI

带`-ui`参数的节点都能作为ui呈现,记住是用8500这个端口.例子我就不写了.

### 不足之处

重启机制没做好.应该在server那里配置`livenessProbe`,当自身离开时自动重启,不过这个问题不是很大,consul自身挺稳定的,本身很少出事.

主要是`consul-client`,`consul-client`在检测离开了server节点之后,应该直接重启,重新加入.但是这一点我没做.

### 其他问题

#### 加密通讯

consul还支持彼此间加密通讯,但是我之前配置client的时候失败了,这就比较遗憾了.加密通讯要加多一些配置,比较麻烦,所以我改成无加密通讯了.

#### 反注册失败

这个问题遇到很多次了,有些服务需要手动反注册3次(可能因为我有个server节点).有些流氓服务不管多少次也是反注册失败,相当残念.

#### consul很卡

consul的架构,server一定要跟client分离.如果直接往server注册服务,server担任了服务健康检查的角色,就会使整个consul变得非常的卡,我本想通过反注册服务给它降低负荷,但还是失败了,搞得最后我迁移了配置,重新搭了一套consul,相当蛋疼.

### 常用api

```
# 反注册服务
put /v1/agent/service/deregister/<serviceid>
# 获取配置
get /v1/kv/goms/config/<config>
# 获取服务列表
get /v1/agent/services
# 查询节点状态
get /v1/status/leader
```

### 参考链接

https://github.com/hashicorp/consul-helm

## 26.3 采集kubernetes的容器日志


需求是采集kubernetes的容器日志并导入到 elasticsearch 分析

`/var/log/containers`下面的文件其实是软链接

真正的日志文件在`/var/lib/docker/containers`这个目录

可选方案:

1. Logstash(过于消耗内存,尽量不要用这个)
2. fluentd
3. filebeat
4. 不使用docker-driver

### 日志的格式

/var/log/containers

```json
{
	"log": "17:56:04.176 [http-nio-8080-exec-5] INFO  c.a.goods.proxy.GoodsGetServiceProxy - ------ request_id=514136795198259200,zdid=42,gid=108908071,从缓存中获取数据:失败 ------\n",
	"stream": "stdout",
	"time": "2018-11-19T09:56:04.176713636Z"
}

{
	"log": "[][WARN ][2018-11-19 18:13:48,896][http-nio-10080-exec-2][c.h.o.p.p.s.impl.PictureServiceImpl][[msg:图片不符合要求:null];[code:400.imageUrl.invalid];[params:https://img.alicdn.com/bao/uploaded/i2/2680224805/TB2w5C9bY_I8KJjy1XaXXbsxpXa_!!2680224805.jpg];[stack:{\"requestId\":\"514141260156502016\",\"code\":\"400.imageUrl.invalid\",\"msg\":\"\",\"stackTrace\":[],\"suppressedExceptions\":[]}];]\n",
	"stream": "stdout",
	"time": "2018-11-19T10:13:48.896892566Z"
}
```

### Logstash

- filebeat.yml

```
filebeat:
  prospectors:
  - type: log
    //开启监视，不开不采集
    enable: true
    paths:  # 采集日志的路径这里是容器内的path
    - /var/log/elkTest/error/*.log
    # 日志多行合并采集
    multiline.pattern: '^\['
    multiline.negate: true
    multiline.match: after
    # 为每个项目标识,或者分组，可区分不同格式的日志
    tags: ["java-logs"]
    # 这个文件记录日志读取的位置，如果容器重启，可以从记录的位置开始取日志
    registry_file: /usr/share/filebeat/data/registry

output:
  # 输出到logstash中
  logstash:
    hosts: ["0.0.0.0:5044"]
```

注：6.0以上该filebeat.yml需要挂载到/usr/share/filebeat/filebeat.yml,另外还需要挂载/usr/share/filebeat/data/registry 文件，避免filebeat容器挂了后，新起的重复收集日志。  


- logstash.conf

```
input {
	beats {
	    port => 5044
	}
}
filter {
    
   if "java-logs" in [tags]{ 
     grok {
        match => {
	       "message" => "(?<date>\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2},\d{3})\]\[(?<level>[A-Z]{4,5})\]\[(?<thread>[A-Za-z0-9/-]{4,40})\]\[(?<class>[A-Za-z0-9/.]{4,40})\]\[(?<msg>.*)"
        }
        remove_field => ["message"]
     }
    }
	#if ([message] =~ "^\[") {
    #    drop {}
    #}
	# 不匹配正则，匹配正则用=~
	if [level] !~ "(ERROR|WARN|INFO)" {
        drop {}
    }
}

## Add your filters / logstash plugins configuration here

output {
	elasticsearch {
		hosts => "0.0.0.0:9200"
	}
}

```

### fluentd

[fluentd-es-image镜像](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch/fluentd-es-image)

[Kubernetes-基于EFK进行统一的日志管理](https://www.kubernetes.org.cn/4278.html)


[Docker Logging via EFK (Elasticsearch + Fluentd + Kibana) Stack with Docker Compose](https://docs.fluentd.org/v0.12/articles/docker-logging-efk-compose)


### filebeat+ES pipeline


#### 定义pipeline

- 定义java专用的管道

```

PUT /_ingest/pipeline/java
{
	"description": "[0]java[1]nginx[last]通用规则",
	"processors": [{
		"grok": {
			"field": "message",
			"patterns": [
				"\\[%{LOGLEVEL:level}\\s+?\\]\\[(?<date>\\d{4}-\\d{2}-\\d{2}\\s\\d{2}:\\d{2}:\\d{2},\\d{3})\\]\\[(?<thread>[A-Za-z0-9/-]+?)\\]\\[%{JAVACLASS:class}\\]\\[(?<msg>[\\s\\S]*?)\\]\\[(?<stack>.*?)\\]"
			]
		},"remove": {
              "field": "message"
            }
	}]
}

PUT /_ingest/pipeline/nginx
{
	"description": "[0]java[1]nginx[last]通用规则",
	"processors": [{
		"grok": {
			"field": "message",
			"patterns": [
				"%{IP:client} - - \\[(?<date>.*?)\\] \"(?<method>[A-Za-z]+?) (?<url>.*?)\" %{NUMBER:statuscode} %{NUMBER:duration} \"(?<refer>.*?)\" \"(?<user-agent>.*?)\"" 
			]
		},"remove": {
              "field": "message"
            }
	}]
}

PUT /_ingest/pipeline/default
{
	"description": "[0]java[1]nginx[last]通用规则",
	"processors": []
}

PUT /_ingest/pipeline/all
{
	"description": "[0]java[1]nginx[last]通用规则",
	"processors": [{
		"grok": {
			"field": "message",
			"patterns": [
				"\\[%{LOGLEVEL:level}\\s+?\\]\\[(?<date>\\d{4}-\\d{2}-\\d{2}\\s\\d{2}:\\d{2}:\\d{2},\\d{3})\\]\\[(?<thread>[A-Za-z0-9/-]+?)\\]\\[%{JAVACLASS:class}\\]\\[(?<msg>[\\s\\S]*?)\\]\\[(?<stack>.*?)\\]",
				
				"%{IP:client} - - \\[(?<date>.*?)\\] \"(?<method>[A-Za-z]+?) (?<url>.*?)\" %{NUMBER:statuscode} %{NUMBER:duration} \"(?<refer>.*?)\" \"(?<user-agent>.*?)\"",
				
				".+"
			]
		}
	}]
}

```

#### filebeat.yml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: kube-system
  labels:
    k8s-app: filebeat
data:
  filebeat.yml: |-
    filebeat.config:
        inputs:
          # Mounted `filebeat-inputs` configmap:
          path: ${path.config}/inputs.d/*.yml
          # Reload inputs configs as they change:
          reload.enabled: false
        modules:
          path: ${path.config}/modules.d/*.yml
          # Reload module configs as they change:
          reload.enabled: false
    setup.template.settings:
        index.number_of_replicas: 0

    # https://www.elastic.co/guide/en/beats/filebeat/6.5/filebeat-reference-yml.html
    # https://www.elastic.co/guide/en/beats/filebeat/current/configuration-autodiscover.html
    filebeat.autodiscover:
     providers:
       - type: kubernetes
         templates:
             config:
               - type: docker
                 containers.ids:
                  # - "${data.kubernetes.container.id}" 
                  - "*"
                 enable: true
                 processors:
                  - add_kubernetes_metadata:
                      # include_annotations:
                      #   - annotation_to_include        
                      in_cluster: true
                  - add_cloud_metadata:

    cloud.id: ${ELASTIC_CLOUD_ID}
    cloud.auth: ${ELASTIC_CLOUD_AUTH}

    output:
      elasticsearch:
        hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
        # username: ${ELASTICSEARCH_USERNAME}
        # password: ${ELASTICSEARCH_PASSWORD}
        # pipelines:          
        #   - pipeline: "nginx"
        #     when.contains:
        #       kubernetes.container.name: "nginx-"
        #   - pipeline: "java"
        #     when.contains:
        #       kubernetes.container.name: "java-"              
        #   - pipeline: "default"  
        #     when.contains:
        #       kubernetes.container.name: ""
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat
spec:
  template:
    metadata:
      labels:
        k8s-app: filebeat
    spec:
      tolerations:
        - key: "elasticsearch-exclusive"
          operator: "Exists"
          effect: "NoSchedule"         
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
        - name: filebeat
          imagePullPolicy: Always          
          image: 'filebeat:6.6.0'
          args: [
            "-c", 
            "/etc/filebeat.yml",
            "-e",
          ]         
          env:
          - name: ELASTICSEARCH_HOST
            value: 0.0.0.0
          - name: ELASTICSEARCH_PORT
            value: "9200"
          # - name: ELASTICSEARCH_USERNAME
          #   value: elastic
          # - name: ELASTICSEARCH_PASSWORD
          #   value: changeme
          # - name: ELASTIC_CLOUD_ID
          #   value:
          # - name: ELASTIC_CLOUD_AUTH
          #   value:
          securityContext:
            runAsUser: 0
            # If using Red Hat OpenShift uncomment this:
            #privileged: true
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 100Mi
          volumeMounts:
          - name: config
            mountPath: /etc/filebeat.yml
            readOnly: true
            subPath: filebeat.yml
          - name: data
            mountPath: /usr/share/filebeat/data
          - name: varlibdockercontainers
            mountPath: /var/lib/docker/containers
            readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      # data folder stores a registry of read status for all files, so we don't send everything again on a Filebeat pod restart
      - name: data
        hostPath:
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  verbs:
  - get
  - watch
  - list
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat
```

如果output是单节点elasticsearch,可以通过修改模板把导出的filebeat*设置为0个副本

```
curl -X PUT "10.10.10.10:9200/_template/template_log" -H 'Content-Type: application/json' -d'
{
    "index_patterns" : ["filebeat*"],
    "order" : 0,
    "settings" : {
        "number_of_replicas" : 0
    }
}
'
```



参考链接:
1. [running-on-kubernetes](https://www.elastic.co/guide/en/beats/filebeat/current/running-on-kubernetes.html)
1. [ELK+Filebeat 集中式日志解决方案详解](https://www.ibm.com/developerworks/cn/opensource/os-cn-elk-filebeat/index.html)
2. [filebeat.yml（中文配置详解）](http://www.cnblogs.com/zlslch/p/6622079.html)
3. [Elasticsearch Pipeline 详解](https://www.felayman.com/articles/2017/11/24/1511527532643.html)
4. [es number_of_shards和number_of_replicas](https://www.cnblogs.com/mikeluwen/p/8031813.html)



### 其他方案

有些是sidecar模式,sidecar模式可以做得比较细致.

1. [使用filebeat收集kubernetes中的应用日志](https://jimmysong.io/posts/kubernetes-filebeat/)
1. [使用Logstash收集Kubernetes的应用日志](https://jimmysong.io/posts/kubernetes-logstash/)
2. 


#### 阿里云的方案

1. [Kubernetes日志采集流程](https://help.aliyun.com/document_detail/66654.html?spm=5176.8665266.sug_det.5.bbdc9gVU9gVUmc)


#### 跟随docker启动
1. [docker驱动](https://www.fluentd.org/guides/recipes/docker-logging)


```bash
kubectl delete po $pod  -n kube-system
kubectl get po -l k8s-app=fluentd-es -n kube-system
pod=`kubectl get po -l k8s-app=fluentd-es -n kube-system | grep -Eoi 'fluentd-es-([a-z]|-|[0-9])+'` && kubectl logs $pod -n kube-system
kubectl get events -n kube-system | grep $pod
```

## 26.4 多种方式请求Kubernetes api-server


连接api-server一般分3种情况：

1. Kubernetes Node通过kubectl proxy中转连接
2. 通过授权验证,直接连接(kubectl和各种client就是这种情况)
  - `kubectl`加载`~/.kube/config`作为授权信息,请求远端的`api-server`的resetful API.`api-server`根据你提交的授权信息判断有没有权限,有权限的话就将对应的结果返回给你。
3. 容器内部通过`ServiceAccount`连接


### 容器请求api-server

![img](https://d33wubrfki0l68.cloudfront.net/673dbafd771491a080c02c6de3fdd41b09623c90/50100/images/docs/admin/access-control-overview.svg)

`Kubernetes`这套RBAC的机制在[之前的文章](https://www.zeusro.tech/2019/01/17/kubernetes-rbac/)有提过.这里就不解释了

为了方便起见,我直接使用`kube-system`的`admin`作为例子.

```yaml
# {% raw %}

apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"name":"admin","namespace":"kube-system"}}
  name: admin
  namespace: kube-system
  resourceVersion: "383"
secrets:
- name: admin-token-wggwk
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
  resourceVersion: "51"
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
  resourceVersion: "102"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:masters

# {% endraw %}
```

简单地说,容器通过`ServiceAccount`配合RBAC这套机制,让容器拥有访问`api-server`的权限.

原本我打算在`kube-system`下面创建一个nginx容器,去访问,但是curl失败了,后来我找了个centos的镜像去测试.大家记得配置好`serviceAccount`就行


    metadata.spec.template.spec.serviceAccount: admin



#### deploy声明sa(`ServiceAccount`)的本质

在deploy声明sa的本质是把sa的对应的secret挂载到`/var/run/secrets/kubernetes.io/serviceaccount`目录中.

不声明sa,则把`default`作为sa挂载进去

```bash
# k edit secret admin-token-wggwk
# 用edit加载的secret内容都会以base64形式表示
# base64(kube-system):a3ViZS1zeXN0ZW0=
apiVersion: v1
data:
  ca.crt: *******
  namespace: a3ViZS1zeXN0ZW0=
  token: *******
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: admin
    kubernetes.io/service-account.uid: 9911ff2a-8c46-4179-80ac-727f48012229 
  name: admin-token-wggwk
  namespace: kube-system
  resourceVersion: "378"
type: kubernetes.io/service-account-token
```

所以deploy衍生的每一个pod里面的容器,
`/var/run/secrets/kubernetes.io/serviceaccount`目录下面都会有这3个文件

```bash
/run/secrets/kubernetes.io/serviceaccount # ls -l
total 0
lrwxrwxrwx    1 root     root            13 Apr 19 06:46 ca.crt -> ..data/ca.crt
lrwxrwxrwx    1 root     root            16 Apr 19 06:46 namespace -> ..data/namespace
lrwxrwxrwx    1 root     root            12 Apr 19 06:46 token -> ..data/token
```

虽然这3个文件都是软链接而且最终指向了下面那个带日期的文件夹,但是我们不用管它.

```bash
/run/secrets/kubernetes.io/serviceaccount # ls -a -l
total 4
drwxrwxrwt    3 root     root           140 Apr 19 06:46 .
drwxr-xr-x    3 root     root          4096 Apr 19 06:46 ..
drwxr-xr-x    2 root     root           100 Apr 19 06:46 ..2019_04_19_06_46_10.877180351
lrwxrwxrwx    1 root     root            31 Apr 19 06:46 ..data -> ..2019_04_19_06_46_10.877180351
lrwxrwxrwx    1 root     root            13 Apr 19 06:46 ca.crt -> ..data/ca.crt
lrwxrwxrwx    1 root     root            16 Apr 19 06:46 namespace -> ..data/namespace
lrwxrwxrwx    1 root     root            12 Apr 19 06:46 token -> ..data/token
```

### curl请求api-server

集群就绪之后,在`default`这个命名空间下会有`kubernetes`这个svc,容器透过ca.crt作为证书去请求即可.跨ns的访问方式为`https://kubernetes.default.svc:443`

#### 前期准备

```bash
kubectl exec -it $po sh -n kube-system
cd /var/run/secrets/kubernetes.io/serviceaccount
TOKEN=$(cat token)
APISERVER=https://kubernetes.default.svc:443

```

#### 先伪装成一个流氓去访问`api-server`

```bash
sh-4.2# curl -voa  -s  $APISERVER/version
* About to connect() to kubernetes.default.svc port 443 (#0)
*   Trying 172.30.0.1...
* Connected to kubernetes.default.svc (172.30.0.1) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
*   CAfile: /etc/pki/tls/certs/ca-bundle.crt
  CApath: none
* Server certificate:
* 	subject: CN=kube-apiserver
* 	start date: Jul 24 06:31:00 2018 GMT
* 	expire date: Jul 24 06:44:02 2019 GMT
* 	common name: kube-apiserver
* 	issuer: CN=cc95defe1ffd6401d8ede6d4efb0f0f7c,OU=default,O=cc95defe1ffd6401d8ede6d4efb0f0f7c
* NSS error -8179 (SEC_ERROR_UNKNOWN_ISSUER)
* Peer's Certificate issuer is not recognized.
* Closing connection 0
```

可以看到,用默认的`/etc/pki/tls/certs/ca-bundle.crt`公钥去访问,直接就报证书对不上了(Peer's Certificate issuer is not recognized.)

#### 带证书去访问`api-server`

```bash
curl -s $APISERVER/version  \
--header "Authorization: Bearer $TOKEN" \
--cacert ca.crt 
{
  "major": "1",
  "minor": "11",
  "gitVersion": "v1.11.5",
  "gitCommit": "753b2dbc622f5cc417845f0ff8a77f539a4213ea",
  "gitTreeState": "clean",
  "buildDate": "2018-11-26T14:31:35Z",
  "goVersion": "go1.10.3",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

那这样思路就很明确了,curl的时候带上正确的证书(ca.crt)和请求头就行了.

### 使用curl访问常见API

这里先要介绍一个概念`selfLink`.在`kubernetes`里面,所有事物皆资源/对象.`selfLink`就是每一个资源对应的`api-server`地址.`selfLink`跟资源是一一对应的.

`selfLink`是有规律的,由`namespace`,`type`,`apiVersion`,`name`等组成.


#### [get node](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#list-node-v1-core)

    kubectl get no

```bash
curl \
-s $APISERVER/api/v1/nodes?watch \
--header "Authorization: Bearer $TOKEN" \
--cacert  ca.crt
```

#### [get pod](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#list-pod-v1-core)


    kubectl get po -n kube-system -w


```bash
curl \
-s $APISERVER/api/v1/namespaces/kube-system/pods?watch \
--header "Authorization: Bearer $TOKEN" \
--cacert  ca.crt
```

#### [get pod log](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#read-log-pod-v1-core)

    kubectl logs  -f -c logtail  -n kube-system  logtail-ds-vvpfr


```bash
curl \
-s $APISERVER"/api/v1/namespaces/kube-system/pods/logtail-ds-vvpfr/log?container=logtail&follow" \
--header "Authorization: Bearer $TOKEN" \
--cacert  ca.crt
```

完整API见[kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/)


### 使用JavaScript客户端访问 api-server

2019-08-23，我在部署 kubeflow 的时候,发现里面有个组件是用 nodejs 去请求 api service 的,观察了一下代码,加载配置的地方大致如此.

```ts

public loadFromDefault() {
        if (process.env.KUBECONFIG && process.env.KUBECONFIG.length > 0) {
            const files = process.env.KUBECONFIG.split(path.delimiter);
            this.loadFromFile(files[0]);
            for (let i = 1; i < files.length; i++) {
                const kc = new KubeConfig();
                kc.loadFromFile(files[i]);
                this.mergeConfig(kc);
            }
            return;
        }
        const home = findHomeDir();
        if (home) {
            const config = path.join(home, '.kube', 'config');
            if (fileExists(config)) {
                this.loadFromFile(config);
                return;
            }
        }
        if (process.platform === 'win32' && shelljs.which('wsl.exe')) {
            // TODO: Handle if someome set $KUBECONFIG in wsl here...
            try {
                const result = execa.sync('wsl.exe', ['cat', shelljs.homedir() + '/.kube/config']);
                if (result.code === 0) {
                    this.loadFromString(result.stdout);
                    return;
                }
            } catch (err) {
                // Falling back to alternative auth
            }
        }

        if (fileExists(Config.SERVICEACCOUNT_TOKEN_PATH)) {
            this.loadFromCluster();
            return;
        }

        this.loadFromClusterAndUser(
            { name: 'cluster', server: 'http://localhost:8080' } as Cluster,
            { name: 'user' } as User,
        );
    }
......


    public loadFromCluster(pathPrefix: string = '') {
        const host = process.env.KUBERNETES_SERVICE_HOST;
        const port = process.env.KUBERNETES_SERVICE_PORT;
        const clusterName = 'inCluster';
        const userName = 'inClusterUser';
        const contextName = 'inClusterContext';

        let scheme = 'https';
        if (port === '80' || port === '8080' || port === '8001') {
            scheme = 'http';
        }

        this.clusters = [
            {
                name: clusterName,
                caFile: `${pathPrefix}${Config.SERVICEACCOUNT_CA_PATH}`,
                server: `${scheme}://${host}:${port}`,
                skipTLSVerify: false,
            },
        ];
        this.users = [
            {
                name: userName,
                authProvider: {
                    name: 'tokenFile',
                    config: {
                        tokenFile: `${pathPrefix}${Config.SERVICEACCOUNT_TOKEN_PATH}`,
                    },
                },
            },
        ];
        this.contexts = [
            {
                cluster: clusterName,
                name: contextName,
                user: userName,
            },
        ];
        this.currentContext = contextName;
    }
```

可以看到,加载配置是有先后顺序的. sa 排在比较靠后的优先级.

host 和 port 通过读取相应 env 得出(实际上,就算在yaml没有配置ENV, kubernetes 本身也会注入大量ENV,这些ENV大多是svc的ip地址和端口等)

而且默认的客户端 `skipTLSVerify: false,`

那么使用默认的客户端,要取消SSL验证咋办呢?这里提供一个比较蠢但是万无一失的办法:

```ts
import * as k8s from '@kubernetes/client-node';
import { Cluster } from '@kubernetes/client-node/dist/config_types';

    this.kubeConfig.loadFromDefault();
    const context =
      this.kubeConfig.getContextObject(this.kubeConfig.getCurrentContext());
    if (context && context.namespace) {
      this.namespace = context.namespace;
    }
    let oldCluster = this.kubeConfig.getCurrentCluster()
    let cluster: Cluster = {
      name: oldCluster.name,
      caFile: oldCluster.caFile,
      server: oldCluster.server,
      skipTLSVerify: true,
    }
    kubeConfig.clusters = [cluster]
    this.coreAPI = this.kubeConfig.makeApiClient(k8s.Core_v1Api);
    this.customObjectsAPI =
      this.kubeConfig.makeApiClient(k8s.Custom_objectsApi);
```

### 参考链接
1. [Access Clusters Using the Kubernetes API](https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/)
2. [cURLing the Kubernetes API server](https://medium.com/@nieldw/curling-the-kubernetes-api-server-d7675cfc398c)

## 26.5 容器编排的技巧

### wait-for-it

k8s目前没有没有类似docker-compose的`depends_on`依赖启动机制,建议使用[wait-for-it](https://blog.giantswarm.io/wait-for-it-using-readiness-probes-for-service-dependencies-in-kubernetes/)重写镜像的command.

### 在cmd中使用双引号的办法

```yaml

               - "/bin/sh"
               - "-ec"
               - |
                  curl -X POST --connect-timeout 5 -H 'Content-Type: application/json' \
                  elasticsearch-logs:9200/logs,tracing,tracing-test/_delete_by_query?conflicts=proceed  \
                  -d '{"query":{"range":{"@timestamp":{"lt":"now-90d","format": "epoch_millis"}}}}'
```

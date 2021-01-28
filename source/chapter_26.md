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
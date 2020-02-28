
二十一 新一代监控架构
=====================

21.1 核心指标流水线
-------------------

由 kubelet、metrics-server 以及由 apiserver 提供的 api 组成；主要
CPU累计使用率、内存实时使用率、POD 资源占用率及容器的磁盘占用率。

-  metrics-server（新一代的资源指标获取方式）

它是一个 apiserver ，它仅仅用于服务于核心指标服务的，它不是 k8s
的组成部分，仅仅是托管在 k8s 之上 POD。

k8s 的 apiserver 和 metrics-server 的 apiserver
前端应该加一个代理服务器，它就是一个聚合器，把来自多个不同的 apiserver
聚合成一个。它就是 kube-aggregator，经过它聚合后的 api 我么将通过
/apis/metrics.k8s.io/v1/beta1 来获取。

21.2监控流水线
--------------

用于从系统收集各种指标数据并提供终端用户、存储系统以及
HPA，它包含核心指标和非核心指标，非核心指标不能被 k8s
所理解，k8s-prometheus-adapter 就是转换为 k8s 所理解格式的一个插件

-  prometheus

CNCF下的第二大项目，收集各种维度的指标，

它收集的信息，来决定是否进行 HPA（自动伸缩） 的一个标准

prometheus
既作为监控系统使用，也作为特殊指标的提供者来使用，但是如果想要作为特殊指标提供给
HPA
这样的机制使用，需要转换格式，而这个转换为特殊指标的一个插件叫：k8s-prometheus-adapter。

21.3 安装 metrics-server
------------------------

-  官方仓库，这里我使用第一个

.. code:: bash

   https://github.com/kubernetes-incubator/metrics-server/tree/master/deploy/1.8%2B      # 插件官方地址
   https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/metrics-server    # k8s 官方插件示例

-  安装部署相关的文件：/tree/master/deploy/，修改
   metrics-server-deployment.yaml 文件

.. code:: yaml

         containers:
         - name: metrics-server
           image: k8s.gcr.io/metrics-server-amd64:v0.3.1
           imagePullPolicy: Always
           args:                                               # 添加参数
           - '--kubelet-preferred-address-types=InternalIP'    # 不使用主机名，使用 IP
           - '--kubelet-insecure-tls'                          # 不验证客户端证书
           volumeMounts:
           - name: tmp-dir
             mountPath: /tmp

.. code:: bash

   $ kubectl apply -f ./

-  查看 POD 和 Service 的启动情况

.. code:: bash

   $ kubectl get pods -n kube-system
   $ kubectl get svc -n kube-system

-  查看 API 中是否存在，metrics.k8s.io/v1beta1

.. code:: bash

   $ kubectl api-versions

-  通过测试接口获取监控数据，kubectl proxy –port 8080，kubectl top
   也可以正常使用了

.. code:: bash

   $ curl http://127.0.0.1:8080/apis/metrics.k8s.io/v1beta1
   $ kubectl top nodes

21.4 安装 prometheus
--------------------

-  工作原理

::

   -   prometheus 通过 pull metrilcs 指令从每个 Jobs/exporters 拉取数据
   -   其他的 short-lived jobs 也可以通过向 pushgateway 主动发送数据，由 prometheus 被动接收
   -   prometheus 自身实现了一个时间序列数据库，会将得到的数据存储到其中
   -   在 k8s 需要使用 service discovery 来发现服务取得需要监控的目标
   -   可以使用 apiclient、webui、Grafana、来将 prometheus 中的数据展示出来
   -   当需要报警的时候还会推送给 alertmanager 这个组件由这个组件来发送报警

-  部署文件

.. code:: bash

   https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/prometheus
   https://github.com/iKubernetes/k8s-prom

21.5 HPA命令行方式
------------------

-  创建 POD 和 service

.. code:: bash

   kubectl run myapp --image=ikubernetes/myapp:v1 --replicas=1 --requests='cpu=50m',memory='256Mi' --limits='cpu=50m,memory=256Mi' --labels='app=myapp' --expose --port=80

-  创建 HPA 控制器

::

   kubectl autoscale deployment myapp --min=1 --max=8 --cpu-percent=60

-  查看 HPA 控制器，kubectl get hpa

.. code:: bash

   NAME    REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
   myapp   Deployment/myapp   0%/60%    1         8         1          17s

-  开始压力测试

.. code:: bash

   ab -c 100 -n 5000000 http://172.16.100.102:32749/index.html

-  测试结果，自动扩容生效

.. code:: bash

   $ kubectl get hpa -w
   NAME    REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
   myapp   Deployment/myapp   0%/60%    1         8         1          7m35s
   myapp   Deployment/myapp   34%/60%   1         8         1          9m58s
   myapp   Deployment/myapp   102%/60%   1         8         1          11m
   myapp   Deployment/myapp   102%/60%   1         8         2          11m
   myapp   Deployment/myapp   96%/60%    1         8         2          12m
   myapp   Deployment/myapp   96%/60%    1         8         4          12m
   myapp   Deployment/myapp   31%/60%    1         8         4          13m
   myapp   Deployment/myapp   26%/60%    1         8         4          14m
   myapp   Deployment/myapp   0%/60%     1         8         4          15m
   myapp   Deployment/myapp   0%/60%     1         8         4          17m
   myapp   Deployment/myapp   0%/60%     1         8         3          18m

   $ kubectl get pods
   NAME                     READY   STATUS        RESTARTS   AGE
   myapp-64bf6764c5-45qwj   0/1     Terminating   0          7m1s
   myapp-64bf6764c5-72crv   1/1     Running       0          20m
   myapp-64bf6764c5-gmz6c   1/1     Running       0          8m1s

21.6 HPA清单
------------

-  清单定义详见：kubectl explain hpa.spec

.. code:: yaml

   maxReplicas                       <integer>         # 自动伸缩的 POD 数量上限
   minReplicas                       <integer>         # 自动伸缩的 POD 数量下限
   scaleTargetRef                    <Object>          # 其他的伸缩指标
     apiVersion                      <string>          # 指标 api 版本
     kind                            <string>          # 指标类型
     name                            <string>          # 可用指标
   targetCPUUtilizationPercentage    <integer>         # 根据目标 平均 CPU 利用率阈值评估自动伸缩

-  示例清单，它实现了对 myapp 这个 deployment 控制器下的 POD
   进行自动扩容

.. code:: yaml

   apiVersion: autoscaling/v2beta1
   kind: HorizontalPodAutoscaler
   metadata:
     name: myapp-hpa-v2
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: myapp
     minReplicas: 1
     maxReplicas: 10
     metrics:
     - type: Resource
       resource:
         name: cpu
         targetAverageUtilization: 55
     - type: Resource
       resource:
         name: memory
         targetAverageValue: 50Mi

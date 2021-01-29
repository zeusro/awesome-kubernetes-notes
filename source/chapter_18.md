
十八 高级调度设置
=================

节点选择器：nodeselector、nodeName

节点亲和调度：nodeAffinity

18.1 节点选择器
---------------

-  使用 nodeselector 来将预选范围缩小，没有被 nodeselector
   选中的节点将被预选阶段淘汰

.. code:: yaml

   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-schedule-demo
     namespace: default
     labels:
       app: myapp
   spec:
     containers:
     - name: myapp
       image: ikubernetes/myapp:v1
     nodeSelector:
       gpu: ok

-  此时如果没有任何一个节点有 gpu 这个标签，那么这个 POD
   的调度将会被挂起，Pending 状态，直到满足条件



   kubectl label nodes node2 gpu=ok --overwrite

-  查看 POD 被调度的节点，POD 已经被调度到 node2 节点了



   kubectl get pods -o wide

18.2 对节点的亲和性
-------------------

亲和性定义，详见：kubectl explain pods.spec.affinity.nodeAffinity

-  POD 对节点亲和性定义



   nodeAffinity             <Object>                                # POD 对 node 节点的亲和性
     preferredDuringSchedulingIgnoredDuringExecution  <[]Object>    # 软亲和性要求，尽量满足亲和性
       preference           <Object>                                # 亲和的节点对象
         matchExpressions   <[]Object>                              # 查找表达式
           key              <string>                                # 标签
           operator         <string>                                # 操作：比较
           values           <[]string>                              # 值
         matchFields        <[]Object>                              # 查找字段
           key              <string>                                # 标签
           operator         <string>                                # 操作：比较
           values           <[]string>                              # 值
       weight               <integer>                               # 权重 1 - 100
     requiredDuringSchedulingIgnoredDuringExecution   <Object>      # 硬亲和性要求，不满足则 Pending
       nodeSelectorTerms    <[]Object>                              # 选择器对象列表
         matchExpressions   <[]Object>                              # 选择器对象列表
           key              <string>                                # 标签
           operator         <string>                                # 操作：比较
           values           <[]string>                              # 值
         matchFields        <[]Object>                              # 查找字段
           key              <string>                                # 标签
           operator         <string>                                # 操作：比较
           values           <[]string>                              # 值

-  示例配置

.. code:: yaml

   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-nodeaffinity1-demo
     namespace: default
     labels:
       app: myapp
   spec:
     containers:
     - name: myapp
       image: ikubernetes/myapp:v1
     affinity:
       nodeAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:        # 硬亲和性要求，不满足就 Pending
           nodeSelectorTerms:
           - matchExpressions:
             - key: zone
               operator: In
               values:
               - foo
               - bar

   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-nodeaffinity2-demo
     namespace: default
     labels:
       app: myapp
   spec:
     containers:
     - name: myapp
       image: ikubernetes/myapp:v1
     affinity:
       nodeAffinity:
         preferredDuringSchedulingIgnoredDuringExecution:     # 软亲和性要求，不满足也可以对凑
         - preference:
             matchExpressions:
             - key: zone
               operator: In
               values:
               - foo
               - bar
           weight: 50

-  查看结果，kubectl get pods



   NAME                     READY   STATUS    RESTARTS   AGE
   pod-nodeaffinity1-demo   0/1     Pending   0          6m16s  # 硬亲和性要求，没有就等待
   pod-nodeaffinity2-demo   1/1     Running   0          7s     # 软亲和性要求，没有合适主机也可以凑和

18.3 对 POD 的亲和性
--------------------

POD 和 POD 出于高效的通信这种需求，所以需要将 POD 和 POD
组织在同一台机器，同一个机房，例如：LNMT 如果能运行在同一个主机上更好。

1. 想把一组 POD
   运行在一起，使用节点亲和性就可以实现，为了达成这个目的，我们需要：把节点标签精心编排，希望在一起运行的
   POD，就使用同一组标签选择器来选择节点，这种方式需要管理节点标签和 POD
   亲和性才能做到。

2. 想把一组 POD 运行在一起，使用 POD 亲和性，我们可以设置 POD 对某个 POD
   的亲和性，那么比如：LNMT，那么 MySQL 和 Tomcat 可以设置为更加亲和
   Ngninx 所在的主机或机柜，所以必须有个前提就是 POD 和 POD
   怎么才是最近的，这个标准是什么，也就是什么是同一位置，怎么才能知道
   node 和 node 是在一个机柜。

   所以可以为同一个机柜的 node 节点打上相同的标签。

3. MySQL 和 Tomcat 一定不能和 Nginx 运行在一起，这就是反亲和性。

-  POD 对其他 POD 的亲和性，详见：kubectl explain
   pods.spec.affinity.podAffinity

.. code:: yaml

   podAffinity                <Object>                              # POD 对其他 POD 的亲和性
     preferredDuringSchedulingIgnoredDuringExecution  <[]Object>    # 软性亲和性，尽量满足亲和性
       podAffinityTerm        <Object>                              # 亲和的 POD 对象
         labelSelector        <Object>                              # 标签选择器对象列表
           matchExpressions   <[]Object>                            # 标签选择器对象，选 POD 标签
             key              <string>                              # 标签
             operator         <string>                              # 操作：比较
             values           <[]string>                            # 值
           matchLabels        <map[string]string>                   # 集合标签选择器
         namespaces           <[]string>                            # 名称空间的列表
         topologyKey          <string>                              # 亲和判断条件
       weight                 <integer>                             # 权重 1 - 100
     requiredDuringSchedulingIgnoredDuringExecution   <[]Object>    # 硬性亲和性，不满足则 Pending
       labelSelector          <Object>                              # 标签选择器对象列表
         matchExpressions   <[]Object>                              # 标签选择器对象，选 POD 标签
           key              <string>                                # 标签
           operator         <string>                                # 操作：比较
           values           <[]string>                              # 值
         matchLabels        <map[string]string>                     # 集合标签选择器
       namespaces             <[]string>                            # 名称空间的列表
       topologyKey            <string>                              # 亲和判断条件

-  示例配置

.. code:: yaml

   apiVersion: v1
   kind: Pod
   metadata:
     name: pod1
     namespace: default
     labels:
       app: myapp
       tier: frontend
   spec:
     containers:
     - name: myapp
       image: ikubernetes/myapp:v1

   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod2
     namespace: default
     labels:
       app: db
       tier: db
   spec:
     containers:
     - name: busybox
       image: busybox:latest
       imagePullPolicy: IfNotPresent
       command:
       - "sh"
       - "-c"
       - "sleep 3600"
     affinity:
       podAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:   # 硬亲和性要求，不满足的 Pending
         - labelSelector:
             matchExpressions:
             - key: app
               operator: In
               values:
               - myapp
           topologyKey: kubernetes.io/hostname             # 亲和性的依据为同一个主机名则亲和

-  查看结果，kubectl get pods -o wide



   NAME   READY   STATUS    RESTARTS   AGE     IP           NODE    NOMINATED NODE   READINESS GATES
   pod1   1/1     Running   0          3m33s   10.244.2.4   node3   <none>           <none>
   pod2   1/1     Running   0          3m33s   10.244.2.5   node3   <none>           <none>

18.4 对 POD 的反亲和性
----------------------

-  POD 对其他 POD 的反亲和性，详见：kubectl explain
   pods.spec.affinity.podAntiAffinity

.. code:: yaml

   podAntiAffinity              <Object>                            # POD 对其他 POD 的反亲和性
     preferredDuringSchedulingIgnoredDuringExecution  <[]Object>    # 软性反亲和性，尽量满足亲和性
       podAffinityTerm        <Object>                              # 反亲和的 POD 对象
         labelSelector        <Object>                              # 标签选择器对象列表
           matchExpressions   <[]Object>                            # 标签选择器对象，选 POD 标签
             key              <string>                              # 标签
             operator         <string>                              # 操作：比较
             values           <[]string>                            # 值
           matchLabels        <map[string]string>                   # 集合标签选择器
         namespaces           <[]string>                            # 名称空间的列表
         topologyKey          <string>                              # 亲和判断条件
       weight                 <integer>                             # 权重 1 - 100
     requiredDuringSchedulingIgnoredDuringExecution   <[]Object>    # 硬性反亲和性，不满足则 Pending
       labelSelector          <Object>                              # 标签选择器对象列表
         matchExpressions   <[]Object>                              # 标签选择器对象，选 POD 标签
           key              <string>                                # 标签
           operator         <string>                                # 操作：比较
           values           <[]string>                              # 值
         matchLabels        <map[string]string>                     # 集合标签选择器
       namespaces             <[]string>                            # 名称空间的列表
       topologyKey            <string>                              # 亲和判断条件

-  配置清单

.. code:: yaml

   apiVersion: v1
   kind: Pod
   metadata:
     name: pod3
     namespace: default
     labels:
       app: myapp
       tier: frontend
   spec:
     containers:
     - name: myapp
       image: ikubernetes/myapp:v1

   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod4
     namespace: default
     labels:
       app: db
       tier: db
   spec:
     containers:
     - name: busybox
       image: busybox:latest
       imagePullPolicy: IfNotPresent
       command:
       - "sh"
       - "-c"
       - "sleep 3600"
     affinity:
       podAntiAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:   # 硬亲和性要求，不满足的 Pending
         - labelSelector:
             matchExpressions:
             - key: app
               operator: In
               values:
               - myapp
           topologyKey: kubernetes.io/hostname             # 反亲和性的依据为同一个主机名

18.5 node 污点
--------------

污点只用在 node
上的键值属性（nodes.spec.taints），它的作用是拒绝不能容忍这些污点的 POD
运行的，因此需要在 POD
上定义容忍度（pods.spec.tolerations），它也是键值数据，是一个列表，表示
POD 可以容忍的污点列表。

一个 POD 能不能运行在一个节点上，就是 pods.spec.tolerations
列表中是否包括了 nodes.spec.taints 中的数据。

-  node 污点清单格式，详见：kubectl explain node.spec.taints

.. code:: yaml

   taints          <[]Object>     # 污点对象列表
     effect        <string>       # 当 POD 不能容忍这个污点的时候，要采取的行为，也就是排斥不容忍污点的 POD
       NoSchedule                 # 影响调度过程，但是已经调度完成 POD 无影响
       PreferNoSchedule           # 影响调度过程，尝试驱逐调度已经完成的但不容忍新污点的 POD
       NoExecute                  # 新增的污点，影响新的调度过程，且强力驱逐调度已经完成的但不容忍新污点的 POD
     key           <string>       # 键
     timeAdded     <string>       # 
     value         <string>       # 值

-  给 node 打上污点，键为 node-type 值为 production，污点动作



   kubectl taint node node2 node-type=production:NoSchedule

-  删除 node 上的一个污点



   kubectl taint node node2 node-type-

-  测试清单

.. code:: yaml

   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: myapp-deploy
     namespace: default
   spec:
     replicas: 4
     selector:
       matchLabels:
         app: myapp
         release: canary
     template:
       metadata:
         labels:
           app: myapp
           release: canary
       spec:
         containers:
           - name: myapp
             image: ikubernetes/myapp:v2
             ports:
               - name: http
                 containerPort: 80

-  查看结果，kubectl get pods -o wide，因为 POD 没有定义容忍 node2
   的污点

.. code:: yaml

   NAME                            READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
   myapp-deploy-675558bfc5-4x5cf   1/1     Running   0          9s    10.244.2.13   node3   <none>           <none>
   myapp-deploy-675558bfc5-58f2s   1/1     Running   0          9s    10.244.2.10   node3   <none>           <none>
   myapp-deploy-675558bfc5-gz4kv   1/1     Running   0          9s    10.244.2.12   node3   <none>           <none>
   myapp-deploy-675558bfc5-hlxdd   1/1     Running   0          9s    10.244.2.11   node3   <none>           <none>

-  此时给 node3 也打上污点，并驱逐原有的 POD



   kubectl taint node node3 node-type=dev:NoExecute

-  查看结果，kubectl get pods -o wide，因为 node3
   新增的污点驱逐了不能容忍污点的 POD ，所以 POD 被挂起



   NAME                            READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
   myapp-deploy-675558bfc5-22wpj   0/1     Pending   0          10s   <none>   <none>   <none>           <none>
   myapp-deploy-675558bfc5-lctv5   0/1     Pending   0          14s   <none>   <none>   <none>           <none>
   myapp-deploy-675558bfc5-m5qdh   0/1     Pending   0          15s   <none>   <none>   <none>           <none>
   myapp-deploy-675558bfc5-z8c4q   0/1     Pending   0          14s   <none>   <none>   <none>           <none>

18.6 POD 污点容忍
-----------------

-  POD 容忍度，详见：kubectl explain pods.spec.tolerations

.. code:: yaml

   tolerations            <[]Object>    # 容忍度对象
     effect               <string>      # 能否容忍 node 上的污点驱逐策略，为空表示容忍任何驱逐策略
       NoSchedule                       # 能容忍 node 污点的 NoSchedule
       PreferNoSchedule                 # 能容忍 node 污点的 PreferNoSchedule
       NoExecute                        # 能容忍 node 污点的 NoExecute
     key                  <string>      # 污点的键
     operator             <string>      # Exists 污点存在不管什么值，Equal 污点的值必须等值
     tolerationSeconds    <integer>     # 容忍时间，即如果被驱逐，可以等多久再走，默认 0 秒，NoExecute 使用
     value                <string>      # 污点的值

-  给 node2 、node3 分别打污点



   kubectl taint node node2 node-type=production:NoSchedule
   kubectl taint node node3 node-type=dev:NoExecute

-  定义 POD 清单文件，容忍 node 上存在 node-type 值为 dev
   的污点、接受被驱逐。

.. code:: yaml

   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: myapp-deploy
     namespace: default
   spec:
     replicas: 4
     selector:
       matchLabels:
         app: myapp
         release: canary
     template:
       metadata:
         labels:
           app: myapp
           release: canary
       spec:
         containers:
           - name: myapp
             image: ikubernetes/myapp:v2
             ports:
               - name: http
                 containerPort: 80
         tolerations:
         - key: node-type
           operator: Equal
           value: dev
           effect: NoExecute

-  查看结果，kubectl get pods -o wide，运行在自己容忍的污点的节点上了

.. code:: yaml

   NAME                           READY   STATUS    RESTARTS   AGE     IP            NODE    NOMINATED NODE   READINESS GATES
   myapp-deploy-97578cf74-5v2r6   1/1     Running   0          6m22s   10.244.2.16   node3   <none>           <none>
   myapp-deploy-97578cf74-gbfj7   1/1     Running   0          6m22s   10.244.2.14   node3   <none>           <none>
   myapp-deploy-97578cf74-l4lbv   1/1     Running   0          6m22s   10.244.2.15   node3   <none>           <none>
   myapp-deploy-97578cf74-zvn8f   1/1     Running   0          6m20s   10.244.2.17   node3   <none>           <none>

-  为节点增加新的污点，设置驱离 POD

   kubectl taint node node3 disk=hdd:NoExecute --overwrite

-  查看结果，kubectl get pods -o wide，POD 不能容忍新的污点，结果被驱逐



   NAME                           READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
   myapp-deploy-97578cf74-84bfz   0/1     Pending   0          6s    <none>   <none>   <none>           <none>
   myapp-deploy-97578cf74-fxk2d   0/1     Pending   0          5s    <none>   <none>   <none>           <none>
   myapp-deploy-97578cf74-jp99j   0/1     Pending   0          6s    <none>   <none>   <none>           <none>
   myapp-deploy-97578cf74-vdkbx   0/1     Pending   0          6s    <none>   <none>   <none>           <none>


七 控制器配置清单
=================

7.1 ReplicaSet 
---------------------

   详见：kubectl explain replicaset

-  清单规范



   apiVersion  <string>    # api 版本号，一般为 apps/v1
   kind        <string>    # 资源类别，标记创建什么类型的资源
   metadata    <Object>    # POD 元数据
   spec        <Object>    # 元数据

7.1.1 replicaset.spec 规范
~~~~~~~~~~~~~~~~~~~~~~~~~~

1. replicas 副本数量，指定一个数字

2. selector 标签选择器，可以使用 matchLabels、matchExpressions
   两种类型的选择器来选中目标 POD



   matchLabels：直接给定键值
   matchExpressions：基于给定的表达式来定义使用标签选择器：{key:"KEY",operator:"OPERATOR",value:[VAL1,VAL2,...]}
       使用 key 与 value 进行 operator 运算，复合条件的才被选择
       操作符：
           In、NotIn：其 value 列表必须有值
           Exists、NotExists：其 value 必须为空

3. template 模板，这里面定义的就是一个 POD 对象，这个对象只包含了
   pod.metadata 和 pod.spec 两部分。

7.1.2 清单示例
~~~~~~~~~~~~~~



   apiVersion: apps/v1
   kind: ReplicaSet
   metadata:
     name: myrs
     namespace: default
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: myapp
         release: canary
     template:
       metadata:
         name: myapp-pod     # 这个其实没用，因为创建的 POD 以 rs 的名字开头
         labels:
           app: myapp        # 标签一定要符合 replicaset 标签选择器的规则，否则将陷入创建 pod 的死循环，直到资源耗尽
           release: canary
       spec:
         containers:
           - name: myapp-containers
             image: ikubernetes/myapp:v1
             ports:
               - name: http
                 containerPort: 80

7.2 Deployment控制器
--------------------

Deployment 通过控制 ReplicaSet 来实现功能，除了支持 ReplicaSet
的扩缩容意外，还支持滚动更新和回滚等，还提供了声明式的配置，这个是我们日常使用最多的控制器。它是用来管理无状态的应用。

Deployment 在滚动更新时候，通过控制多个 ReplicaSet 来实现，ReplicaSet
又控制多个 POD，多个 ReplicaSet 相当于多个应用的版本。

.. code:: mermaid

   graph TB
   Deployment[Deployment] --> replicaset1(replicaset1) 
   Deployment[Deployment] --> replicaset2(replicaset2)
   Deployment[Deployment] --> replicaset3(replicaset3)
   replicaset1(replicaset1) --> POD1{POD}
   replicaset1(replicaset1) --> POD2{POD}
   replicaset2(replicaset1) --> POD5{POD}
   replicaset2(replicaset1) --> POD6{POD}
   replicaset3(replicaset1) --> POD9{POD}
   replicaset3(replicaset1) --> POD10{POD}

-  清单规范，详见：kubectl explain deployment



   apiVersion  <string>    # apps/v1

   kind        <string>    # 资源类别，标记创建什么类型的资源

   metadata    <Object>    # POD 元数据

   spec        <Object>    # 元数据

7.2.1 replicaset.spec 对象规范
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. replicas 副本数量，指定一个数字

2. selector 标签选择器，可以使用 matchLabels、matchExpressions
   两种类型的选择器来选中目标 POD



   matchLabels：直接给定键值
   matchExpressions：基于给定的表达式来定义使用标签选择器：{key:"KEY",operator:"OPERATOR",value:[VAL1,VAL2,...]}
       使用 key 与 value 进行 operator 运算，复合条件的才被选择
       操作符：
           In、NotIn：其 value 列表必须有值
           Exists、NotExists：其 value 必须为空

3. template 模板，这里面定义的就是一个 POD 对象，这个对象只包含了
   pod.metadata 和 pod.spec 两部分。

4. strategy 更新策略，支持滚动更新、支持滚动更新的更新方式



   type：                # 更新类型，Recreate 滚动更新，RollingUpdate 滚动更新策略
   rollingUpdate：       # 滚动更新时候的策略，这是默认的更新策略
       maxSurge：        # 滚动更新时候允许临时超出多少个，可以指定数量或者百分比，默认 25%
       maxUnavailable：  # 最多允许多少个 POD 不可用，默认 25%

5. revisionHistoryLimit
   滚动更新后最多保存多少个更新的历史版本，值为一个数字

6. paused 当更新启动后控制是否暂停

.. _清单示例-1:

7.2.2 清单示例
~~~~~~~~~~~~~~



   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: myapp-deploy
     namespace: default
   spec:
     replicas: 2
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
             image: ikubernetes/myapp:v1
             ports:
               - name: http
                 containerPort: 80

7.2.3 关于更新
~~~~~~~~~~~~~~

1. 直接修改清单文件，kubectl apply -f deployment.yaml
2. 使用 kubectl patch 使用 json 格式给出更新的内容



   kubectl patch deployment myapp-deploy -p '{"spec":{"replicas":5}}'    # 修改 POD 副本数量

   kubectl patch deployment myapp-deploy -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'                                 # 修改更新策略

3. 仅更新镜像 kubectl set image



   kubectl set image deployment myapp-deploy myapp=ikubernetes/myapp:v3

7.2.4 模拟金丝雀发布
~~~~~~~~~~~~~~~~~~~~

-  在更新刚刚启动的时候，将更新过程暂停，那么只能更新一个，这实现了在集群中增加一个金丝雀版本



   kubectl set image deployment myapp-deploy myapp=ikubernetes/myapp:v3 && kubectl rollout pause deployment myapp-deploy

-  查看已经被更新中被暂停的控制器状态，可以看到一直处于暂停状态的
   deployment



   kubectl rollout status deployment myapp-deploy



   Waiting for deployment "myapp-deploy" rollout to finish: 1 out of 5 new replicas have been updated...

   等待部署"myapp-deploy"部署完成: 5个新副本中的1个已更新...

-  如果金丝雀没有问题，那么继续可以使用继续更新的命令



   kubectl rollout resume deployment myapp-deploy

7.2.5 更新策略
~~~~~~~~~~~~~~

-  最大不可用为 0 ，更新时候可以临时超出1个



   kubectl patch deployment myapp-deploy -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'

7.2.6 关于回滚
~~~~~~~~~~~~~~

1. rollout undo 是回滚的命令，默认滚回上一版本



   kubectl rollout undo deployment myapp-deploy

2. 查看可以回滚的版本



   kubectl rollout history deployment myapp-deploy

2. rollout undo 指定回滚的版本



   kubectl rollout undo deployment myapp-deploy --to-revision=2

3. 查看当前的工作版本



   kubectl get rs -o wide

7.3 DaemonSet控制器
-------------------

-  清单规范，详见 kubectl explain daemonset



   apiVersion  <string>    # apps/v1

   kind        <string>    # 资源类别，标记创建什么类型的资源

   metadata    <Object>    # POD 元数据

   spec        <Object>    # 元数据

7.3.1 DaemonSet.spec规范
~~~~~~~~~~~~~~~~~~~~~~~~

此处只列举不同之处

1. updateStrategy
   更新策略，支持滚动更新、支持滚动更新的更新方式，默认滚动更新每个 node



   rollingUpdate   # 滚动更新，它只有一个 rollingUpdate 参数，表示每次更新几个 node 上的  DaemonSet 任务
   OnDelete        # 在删除时更新

.. _清单示例-2:

7.3.2 清单示例
~~~~~~~~~~~~~~



   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: redis
     namespace: default
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: redis
         role: logstor
     template:
       metadata:
         labels:
           app: redis
           role: logstor
       spec:
         containers:
           - name: redis
             image: redis:4.0-alpine
             ports:
               - name: redis
                 containerPort: 6379
   ---                                         # 可以使用 --- 来分隔多个记录
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: filebeat-daemonset
     namespace: default
   spec:
     selector:
       matchLabels:
         app: filebeat
         release: stalbe
     template:
       metadata:
         labels:
           app: filebeat
           release: stalbe
       spec:
         containers:
           - name: filebeat
             image: ikubernetes/filebeat:5.6.5-alpine
             env:                                         # 向容器传递环境变量
               - name: REDIS_HOST                         # 容器内的环境变量名称
                 value: redis.default.svc.cluster.local   # 环境变量值，指向 redis service
               - name: REDIS_LOG_LEVEL
                 value: info

.. _关于更新-1:

7.3.3 关于更新
~~~~~~~~~~~~~~

-  更新 filebeat-daemonset 这个 daemonset 控制器下的 filebeat 容器的镜像



   kubectl set image daemonsets filebeat-daemonset filebeat=ikubernetes/filebeat:5.6.6-alpine

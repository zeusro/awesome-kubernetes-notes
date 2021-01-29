
十七 调度策略
=============

Master
主要是运行集群的控制平面组件的，apiserver、scheduler、controlermanager，Master
还依赖与 etcd 这样的存储节点。

kubeadm 部署的集群会将 Master 的控制组件都运行为静态 POD
了，从本质来讲，这些组件就是运行在 Master
节点专为集群提供服务的进程，Master 是不负责运行工作负载的。

node 节点是负责运行工作负载 POD 的相关节点，用户只需要将运行任务提交给
Master 就可以了，用户无需关心运行在哪个 node 节点上，Master 整合了所有
node 为一个虚拟的资源池。

17.1 POD创建流程
----------------

用户创建的任务最终应该运行在哪个 node 节点上，是由 Master 节点上的
scheduler 决定的，而 scheduler
也是允许用户定义它的工作特性的，默认情况下，我们没有定义它，其实是使用的默认的
scheduler 调度器。

当我们使用 kubectl describe pods myapp 查看 POD 信息时候，会有一个
Events 字段中，有关于调度结果的相关信息。

Scheduler 会从众多的 node 节点中挑选出符合 POD
运行要求的节点，然后将选定的 node 节点信息记录在 etcd 中，kubelet 始终
waitch 着 apiserver 上的关于本节点的信息变化，kubelet 就会去 apiserver
获取关于变化信息的配置清单，根据配置清单的定义去创建 POD。

17.2 Service创建过程
--------------------

当用户创建一个 service 的时候，这个请求会提交给 apiserver，apiserver
将清单文件写入 etcd 中，然后每个 node 节点上的 kube-proxy 会 waitch
apiserver 关于 service 资源的相关变动，发生变化时候，每个节点的
kube-proxy 会将 service 创建为 iptables/ipvs 规则。

从通信角度来讲：kubectl、kubelet、kube-proxy 都是 apiserver
的客户端，这些组件与 apiserver 进行交互时候，数据格式为
json，内部的数据序列化方式为 protocolbuff。

17.3 资源限制维度
-----------------

1. 资源需求：运行 POD 需要的最低资源需求
2. 资源限制：POD 可以占用的最高资源限额

17.4 Scheduler 调度过程
-----------------------

1. 预选阶段：排除完全不符合运行这个 POD
   的节点、例如资源最低要求、资源最高限额、端口是否被占用
2. 优选阶段：基于一系列的算法函数计算出每个节点的优先级，按照优先级排序，取得分最高的
   node
3. 选中阶段：如果优选阶段产生多个结果，那么随机挑选一个节点

-  POD 中影响调度的字段，在 kubectl explain pods.spec 中



   nodeName          # 直接指定 POD 的运行节点
   nodeSelector      # 根据 node 上的标签来对 node 进行预选，这就是节点的亲和性

-  其他影响调度的因素

节点亲和性调度：表现为 nodeSelector 字段

POD 间亲和性：POD 更倾向和某些 POD
运行在一起，例如同一个机房、同一个机器

POD 间反亲和性：POD 和某 POD
更倾向不能运行在一起，这叫反亲和性，例如：监听同一个
nodeport，有机密数据的

Taints（污点）：给某些 node 打上污点，

Tolerations（污点容忍）：一个 POD 能够容忍 node 上的污点，如果运行过程中
node 出现新的污点，那么 POD 可以

驱逐POD：node 给一个限定的时间，让 POD 离开这个节点。

17.4 预选因素
-------------

下面的预选条件需要满足所有的预选条件才可以通过预选

1. CheckNodeConditionPred



   检查节点是否正常

2. GeneralPredicates

+--------------+-------------------------------------------------------+
| 子策略       | 作用                                                  |
+==============+=======================================================+
| HostName     | 检查主机名称是不是 pod.spec.hostname 指定的 NodeName  |
+--------------+-------------------------------------------------------+
| PodFitsHostP | 检查 Pod 内每一个容器                                 |
| orts         | pods.spec.containers.ports.hostPort                   |
|              | 清单是否已被其它容器占用，如果有所需的 HostPort       |
|              | 不满足需求，那么 Pod 不能调度到这个主机上             |
+--------------+-------------------------------------------------------+
| MatchNodeSel | 检查 POD 容器上定义了 pods.spec.nodeSelector 查看     |
| ector        | node 标签是否能够匹配                                 |
+--------------+-------------------------------------------------------+
| PodFitsResou | 检查 node 是否有足够的资源运行此 POD 的基本要求       |
| rces         |                                                       |
+--------------+-------------------------------------------------------+

3. NoDiskConflict（默认没有启用）



   检查 pod 定义的存储是否在 node 节点上使用。

4. PodToleratesNodeTaints



   检查节点的污点 nodes.spec.taints 是否是 POD 污点容忍清单中 pods.spec.tolerations 的子集

5. PodToleratesNodeNoExecuteTaints



   检查 pod 是否容忍节点上有 NoExecute 污点。NoExecute 这个污点是啥意思呢。如果一个 pod 上运行在一个没有污点的节点上后，这个节点又给加上污点了，那么 NoExecute 表示这个新加污点的节点会驱逐其上正在运行的 pod；不加 NoExecute 不会驱逐节点上运行的 pod，表示接受既成事实，这是默认策略。

6. CheckNodeLabelPresence（默认没有启用）



   检查节点上指定标签的存在性，如果节点有pod指定的标签，那么这个节点就被选中。

7. CheckServiceAffinity（默认没有启用）



   一个 Service 下可以有多个 POD，比如这些 POD 都运行在 1、2、3 机器上，而没有运行在 4、5、6 机器上，那么CheckServiceAffinity 就表示新加入的 POD 都集中运行在 1、2、3 机器上，这样集中好处是一个 Service 下 POD 之间内部通信的效率变高了。 

8. MaxEBSVolumeCount



   确保已挂载的亚马逊 EBS 存储卷不超过设置的最大值，默认39

9. MaxGCEPDVolumeCount



   确保已挂载的GCE存储卷不超过设置的最大值，默认16

10 MaxAzureDiskVolumeCount



   确保已挂载的Azure存储卷不超过设置的最大值，默认16

11. CheckVolumeBinding



   检查节点上的 PVC 是否被别的 POD 绑定了 

12. NoVolumeZoneConflict



   检查给定的 zone (机房) 限制前提下，检查如果在此主机上部署 POD 是否存在卷冲突

13. CheckNodeMemoryPressure



   检查 node 节点上内存是否存在压力

14. CheckNodeDiskPressure



   检查磁盘 IO 是否压力过大

15. CheckNodePIDPressure



   检查 node 节点上 PID 资源是否存在压力

16. MatchInterPodAffinity



   检查 Pod 是否满足亲和性或者反亲和性

17.5 优选函数
-------------

在每个节点执行优选函数，将结果每个优选函数相加，得分最高的胜出。

1. least_requested.go



   选择消耗最小的节点（根据空闲比率评估 cpu(总容量-sum(已使用)*10/总容量)）

2. balanced_resource_allocation.go



   均衡资源的使用方式，表示以 cpu 和内存占用率的相近程度作为评估标准，二者占用越接近，得分就越高，得分高的胜出。

3. node_prefer_avoid_pods.go



   看节点是否有注解信息 "scheduler.alpha.kubernetes.io/preferAvoidPods" 。没有这个注解信息，说明这个节点是适合运行这个 POD 的。

4. taint_toleration.go



   将 pods.spec.tolerations 与 nodes.spec.taints 列表项进行匹配度检查，匹配的条目越多，得分越低。

5. selector_spreading.go



   查找当前 POD 对象对应的 service、statefulset、replicatset 等所匹配的标签选择器，在节点上运行的带有这样标签的 POD 越少得分越高，这样的 POD 优选被选出。 这就是说我们要把同一个标签选择器下运行的 POD 散开(spreading)到多个节点上。

6. interpod_affinity_test.go



   遍历 POD 对象亲和性的条目，并将那些能够匹配到节点权重相加，值越大的得分越高，得分高的胜出。

7. most_requested.go



   表示尽可能的把一个节点的资源先用完，这个和 least_requested 相反，二者不能同时使用。

8. node_label.go



   根据节点是否拥有标签，不关心标签是什么，来评估分数。

9. image_locality.go



   表示根据满足当前 POD 对象需求的已有镜的体积大小之和来选择节点的。

10. node_affinity.go



   根据 POD 对象中的 nodeselector，对节点进行匹配度检查，能够成功匹配的数量越多，得分就越高。

17.6 选择函数
-------------

当通过优选的节点有多个，那么从中随机选择一台

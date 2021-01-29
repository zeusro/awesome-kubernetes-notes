.. contents::
   :depth: 3
..


十九 容器资源限制
=================

起始值 requests 最低保障

终结值 limits 硬限制

-  CPU

::

   1 颗 CPU = 1000 millicores
   0.5 颗 CPU = 500 m

-  内存

::

   Ei、Pi、Ti、Gi、Mi、Ki

19.1 资源限制
-------------

-  清单格式，详见：kubectl explain pods.spec.containers.resources



   resources      <Object>               # 资源限制
     limits       <map[string]string>    # 资源最高限制
       cpu        <string>               # 单位 m
       memory     <string>               # 单位 Gi、Mi
     requests     <map[string]string>    # 资源最低要求
       cpu        <string>               # 单位 m
       memory     <string>               # 单位 Gi、Mi

-  清单示例，node 节点的 CPU 为 12 核心，cpu limits 设置为 1000m
   也就是允许



   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-resources-demo
     namespace: default
     labels:
       app: myapp
       tier: frontend
   spec:
     containers:
     - name: nginx
       image: ikubernetes/stress-ng
       command:
       - "/usr/bin/stress-ng"
       #- "-m 1"                       # 以单线程压测内存
       - "-c 1"                        # 以单线程压测CPU
       - "--metrics-brief"
       ports:
       - name: http
         containerPort: 80
       - name: https
         containerPort: 443
       resources:
         requests:
           cpu: 1000m                 # 它决定在预选阶段淘汰哪些主机
           memory: 512Mi
         limits:
           cpu: 1000m                 # 表示限制容器使用 node 节点的一颗 CPU，无论多少进程，它们最多只能占用 node 节点的可 CPU
           memory: 512Mi

-  查看结果



   Mem: 855392K used, 139916K free, 10188K shrd, 796K buff, 350368K cached
   CPU0:   0% usr   0% sys   0% nic  99% idle   0% io   0% irq   0% sirq
   CPU1: 100% usr   0% sys   0% nic   0% idle   0% io   0% irq   0% sirq         # 占满了一颗 CPU
   CPU2:   0% usr   0% sys   0% nic  99% idle   0% io   0% irq   0% sirq
   CPU3:   0% usr   0% sys   0% nic  99% idle   0% io   0% irq   0% sirq
   CPU4:   0% usr   0% sys   0% nic  99% idle   0% io   0% irq   0% sirq
   CPU5:   0% usr   0% sys   0% nic  99% idle   0% io   0% irq   0% sirq
   CPU6:   0% usr   0% sys   0% nic  99% idle   0% io   0% irq   0% sirq
   CPU7:   0% usr   0% sys   0% nic  99% idle   0% io   0% irq   0% sirq
   CPU8:   0% usr   0% sys   0% nic  99% idle   0% io   0% irq   0% sirq
   CPU9:   0% usr   0% sys   0% nic  99% idle   0% io   0% irq   0% sirq
   CPU10:   0% usr   0% sys   0% nic  99% idle   0% io   0% irq   0% sirq
   CPU11:   0% usr   0% sys   0% nic  99% idle   0% io   0% irq   0% sirq
   Load average: 0.84 0.50 0.40 3/485 11
     PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
       6     1 root     R     6888   1%   1   8% {stress-ng-cpu} /usr/bin/stress-ng -c 1 --metrics-brief
       1     0 root     S     6244   1%  10   0% /usr/bin/stress-ng -c 1 --metrics-brief
       7     0 root     R     1504   0%  11   0% top

19.2 qos 质量管理
-----------------

-  GuranteedW

::

   每个容器同时设置了 CPU 和内存的 requests 和 limits，而且
       cpu.limits = cpu.requests
       memory.limits = memory.requests
   那么它将优先被调度

-  Burstable

::

   至少有一个容器设置 CPU 或内存资源的 requests 属性

   那么它将具有中等优先级

-  BestEffort

::

   没有任何一个容器设置了 requests 或 limits 属性

   那么它将只有最低优先级，当资源不够用的时候，这个容器可能最先被终止，以腾出资源来，为 Burstable 和 Guranteed

-  oom 策略

::

   最先杀死占用量和需求量的比例大的

## 参考链接

-  `Kubernetes官网教程 <https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/>`__
-  `Kubernetes中文社区 <https://www.kubernetes.org.cn/k8s>`__
-  `从Kubernetes到Cloud
   Native <https://jimmysong.io/kubernetes-handbook/cloud-native/from-kubernetes-to-cloud-native.html>`__
-  `Kubernetes
   Handbook <https://www.bookstack.cn/read/feiskyer-kubernetes-handbook/appendix-ecosystem.md>`__
-  `Kubernetes从入门到实战 <https://www.kancloud.cn/huyipow/kubernetes/722822>`__
-  `Kubernetes指南 <https://kubernetes.feisky.xyz/>`__
-  `awesome-kubernetes <https://ramitsurana.github.io/awesome-kubernetes/>`__
-  `从Docker到Kubernetes进阶 <https://www.qikqiak.com/k8s-book/>`__
-  `python微服务实战 <https://www.qikqiak.com/tdd-book/>`__
-  `云原生之路 <https://jimmysong.io/kubernetes-handbook/cloud-native/from-kubernetes-to-cloud-native.html>`__
-  `CNCF Cloud Native Interactive
   Landscape <https://landscape.cncf.io/>`__

视频
~~~~

-  `微服务容器化实战 <https://www.acfun.cn/v/ac10232871>`__


四 kubectl 使用指南
===========


kubectl 是 apiserver 的客户端程序，这个客户端程序是通过连接 master
节点上的 apiserver ，实现各种 k8s 对象的增删改查等基本操作，在 k8s
可被管理的对象有很多个

   基本命令 (初级):
     create         从文件或标准输入创建资源
     expose         获取一个复制控制器, 服务, 部署或者暴露一个 POD 将其作为新的 Kubernetes 服务公开
     run            创建并运行特定的镜像, 创建使用 deployment 或 job 管理的容器
     set            设置对象的特定功能, 例如发布, 每次去set 不用的image tag

   基本命令 (中级):
     explain        文档或者资源, 可以用来查看资源清单写法
     get            显示一个或多个资源
     edit           编辑服务器上的资源
     delete         按文件名, 标准输入, 资源和名称或资源和标签选择器删除资源

   部署命令:
     rollout        管理资源的部署
     scale          为部署设置新大小, ReplicaSet, Replication Controller, Job
     autoscale      自动扩展一个部署, ReplicaSet, 或者 ReplicationController

   群集管理命令:
     certificate    修改证书资源
     cluster-info   显示群集信息
     top            显示资源(CPU / 内存/ 存储)使用情况, 需要安装metrics-server
     cordon         将节点标记为不可调度
     uncordon       将节点标记为可调度
     drain          设定 node 进入维护模式
     taint          更新一个或多个节点上的污点

   故障排除和调试命令:
     describe       显示特定资源或资源组的详细信息
     logs           在容器中打印容器的日志
     attach         附加到正在运行的容器
     exec           在容器中执行命令
     port-forward   将一个或多个本地端口转发到 pod
     proxy          运行代理到 Kubernetes API 服务器
     cp             将文件和目录复制到容器, 和从容器复制, 跨容器复制文件
     auth           检查授权

   高级命令:
     diff           针对将要应用的版本的 Diff 实时版本
     apply          通过文件名或标准输入将配置应用于资源
     patch          使用策略合并补丁更新资源的字段
     replace        用文件名或标准输入替换资源
     wait           实验阶段命令: 在一个或多个资源上等待特定条件, 定义一个触发器
     convert        在不同的API版本之间转换配置文件
     kustomize      从目录或远程 URL 构建 kustomization 目标

   设置命令:
     label          更新资源上的标签
     annotate       更新资源上的注释
     completion     命令补全相关功能

   其他命令:
     api-resources  在服务器上打印支持的API资源
     api-versions   以 "group/version" 的形式在服务器上打印支持的API版本
     config         修改 kubeconfig 文件
     plugin         提供与插件交互的实用程序
     version        打印客户端和服务器版本信息

4.1 get
-----------


```bash
    # 获取事件
    kubectl get events  --field-selector involvedObject.kind=Service --sort-by='.metadata.creationTimestamp'
    kubectl get event --all-namespaces  --field-selector involvedObject.name=$po
    kubectl get cs
    kubectl get svc --sort-by=.metadata.creationTimestamp
    # 查看节点
    kubectl get no --sort-by=.metadata.creationTimestamp    
    kubectl get po --field-selector spec.nodeName=xxxx
    kubectl get pods --all-namespaces --field-selector spec.nodeName=<node> -o wide
    kubectl get pods -o wide --all-namespaces
    kubectl get po -l app=nginx -w
    kubectl get po --all-namespaces    
    # 查看异常pod
    kubectl get po --all-namespaces --field-selector 'status.phase!=Running'
    kubectl get svc --all-namespaces=true
    #来watch ReplicaSet的变化。
    kubectl get rs -w

```


4.2 run
-------

-  创建控制器并运行镜像

创建一个名为 nginx 的 deployment，镜像为 nginx:latest
,如果不知道副本数，则为1



   kubectl run nginx --image=nginx:latest

-  指定运行的 POD 数量



   kubectl run nginx --image=nginx --replicas=5  # 启动 5 个 POD

-  不运行容器的默认命令，使用自定义的指令



   kubectl run nginx --image=nginx --command -- <cmd> <arg1> ... <argN>

-  运行一个周期任务



   kubectl run pi --schedule="0/5 * * * ?" --image=perl --restart=OnFailure -- perl -Mbignum=bpi -wle 'print bpi(2000)'

-  指定控制器名称运行 nginx 指定端口和副本数量，以测试模式运行

指定参数 dry-run 可以用来验证写的 yaml 文件是否存在异常，不会真正执行



   kubectl run nginx-deploy --image=nginx --port=80 --replicas=1 --dry-run=true

-  查看容器是否运行



   kubectl get deployment

-  查看被调度的主机



   kubectl get pod -o wide

-  通过 ip 地址直接访问，由于所有的 POD
   处于同一个网络中，所以在集群内部是可以访问的



   curl 10.244.2.2

-  假如现在删除刚创建的这个 POD，那么副本控制器会自动在其他的 node
   上重建这个 POD



   kubectl delete pods nginx-deploy-5c9b546997-jsmk6

-  再次执行查看，会发现容器已经被调度到其他节点上运行了



   kubectl get pod -o wide

4.3 expose
----------

现在存在一个问题，就是 POD 的 IP
地址可能随时发生变动，所以不能作为访问的入口，那么就需要 service 来代理
POD 来创建一个固定的端点。

-  创建一个 service 来暴露一个服务

在控制器 nginx-deploy 上创建名字为 nginx 的 service , 它工作端口为 80,
代理的后端容器端口 80, 协议为 TCP



   kubectl expose deployment nginx-deploy --name=nginx --port=80 --target-port=80 --protocol=TCP

-  可以看到刚刚创建的名字为 nginx 的 service ，现在就可以在集群内用
   service 的地址来访问了, 如果外部访问可以使用 NodePort 模式



   kubectl get service

-  删除一个任务



   kubectl delete deployment nginx-deploy

4.4 cp
------

-  拷贝宿主机文件或目录到pod中，⚠️要求tar二进制文件已经存在容器中，不然拷贝会失败

```

   kubectl cp /tmp/foo_dir <some-pod>:/tmp/bar_dir
    
   [root@master ~]# kubectl cp flannel.tar  nginx-58cd4d4f44-8pwb7:/usr/share/nginx/html 
   [root@master ~]# kubectl cp mainfile/  nginx-58cd4d4f44-8pwb7:/usr/share/nginx/html 
   [root@master ~]# kubectl exec -it nginx-58cd4d4f44-8pwb7 -- /bin/bash
   root@nginx-58cd4d4f44-8pwb7:/# ls -l /usr/share/nginx/html/
   total 54108
   -rw-r--r-- 1 root root      537 Jul 11  2017 50x.html
   -rw-r--r-- 1 root root      355 May 27 06:47 dashboard-adminuser.yaml
   -rw------- 1 root root 55390720 May 27 01:49 flannel.tar
   -rw-r--r-- 1 root root      612 Jul 11  2017 index.html
   drwxr-xr-x 4 root root       51 Aug 17 14:16 mainfile
```

4.5 port-forward
----------------

-  端口转发，将svc地址或着pods端口利用kubelet映射到宿主机上,将访问宿主机的8888端口的所有流量转发到8111svc

```
   kubectl port-forward --address 0.0.0.0 service/nginx 8888 8111
```

-  转发pods端口,将访问宿主机的8888端口流量转发到pod的5000端口

```
   kubectl port-forward pod/mypod 8888:5000
```

4.6 coredns
-----------

service 提供了对 pod 的固定访问端点，但是 service
本身的变动我们无法知晓，需要 coredns 对 service 做域名解析。

-  查看 coredns 运行状态



   kubectl get pods -n kube-system -o wide |grep coredns

-  查看各个 kube-system 命名空间运行的服务，可以看到 kube-dns 运行的 IP
   地址



   kubectl get service -n kube-system

-  使用 kube-dns 来解析 nginx 这个 service 的地址就可以正常解析了



   dig -t A nginx.default.svc.cluster.local @10.96.0.10

-  创建一个访问 nginx 客户端容器，并进入交互式模式，这个容器默认的 dns
   服务器就是 kube-dns 所在的服务器



   kubectl run client --image=busybox --replicas=1 -it --restart=Never



   / # cat /etc/resolv.conf 
   nameserver 10.96.0.10                                               # kube-dns 地址
   search default.svc.cluster.local svc.cluster.local cluster.local    # 默认的解析搜索域
   options ndots:5

-  在 busybox 这个容器中请求 nginx 这个域名的 service ，能够正常访问



   wget -O - -q http://nginx:80/

4.7 模拟 POD 被删除
-------------------

-  现在我们删除 service 后端的 POD ，副本控制器会自动创建新的 POD，而
   service 则会自动指向新创建的 POD



   kubectl delete pods nginx-deploy-5c9b546997-4w24n

-  查看由副本控制器自动创建的 POD



   kubectl get pods

-  在 busybox 这个容器中请求 nginx 这个域名的 service ，访问没有受到影响



   wget -O - -q http://nginx:80/

4.8 模拟 service 被删除
-----------------------

-  当我们删除 service 并且重新建立一个 service 再次查看 service
   的地址已经发生变化了



   kubectl delete service nginx



   kubectl expose deployment nginx-deploy --name=nginx --port=80 --target-port=80 --protocol=TCP



   kubectl get service

-  在 busybox 这个容器中请求 nginx 这个域名的 service
   ，访问没有仍然没有受到影响



   wget -O - -q http://nginx:80/

4.9 labels
----------

为什么 Pod 被删除后，servic 仍然能够正确的调度到新的 POD 上，这就是 k8s
的 labels 这个机制来保证的。

能够使用标签机制不止有 pod、在 k8s
中很多对象都可以使用标签，例如：node、service

-  查看 service 的详细信息，会发现标签选择器



   kubectl describe service nginx

```
   Name:              nginx
   Namespace:         default
   Labels:            run=nginx-deploy
   Annotations:       <none>
   Selector:          run=nginx-deploy       # 这个选择器会自动选中 run 标签，且值为 nginx-deploy 的 POD
   Type:              ClusterIP
   IP:                10.101.149.4
   Port:              <unset>  80/TCP
   TargetPort:        80/TCP
   Endpoints:         10.244.2.4:80          # 当 service 的后端，当 POD 发生变动则立即会更新
   Session Affinity:  None
   Events:            <none>
```

-  查看 POD 的标签，会看到拥有 run=nginx-deploy
   标签的容器，而人为删除一个 POD
   后，副本控制器创建的副本上的标签不会变化，所以标签又被 service 关联。



   kubectl get pods --show-labels


```
   NAME                            READY   STATUS    RESTARTS   AGE     LABELS
   client                          1/1     Running   0          21m     run=client
   nginx-deploy-5c9b546997-kh88w   1/1     Running   0          8m37s   pod-template-hash=5c9b546997,run=nginx-deploy
```

-  查看 POD 的详细信息，也可以查看到 POD 的详细信息



   kubectl describe deployment nginx-deploy

-  根据标签过滤，使用 -l 来指定标签名称或同时过滤其值



   kubectl get pods --show-labels -l run=nginx-deploy

-  标签选择器集中运算



```
   关系与:  KEY,KEY KEY=VALUE2,KEY=VALUE2       # -l run,app
   等值关系:KEY = VALUE KEY != VALUE           # -l run=nginx-deploy,app!=myapp
   集合关系:KYE in|not in (VALUE1,VALUE2)      # -l "release in (canary,bata,alpha)"  
```

-  显示指定的标签的值，下面显示了两个标签



   kubectl get pods --show-labels -L run,pod-template-hash

-  为指定的 POD 打标签，为 client 这个 POD 打上一个 release 标签，其值为
   canary



   kubectl label pods client release=canary

-  修改 POD 的标签，使用 –overwrite 进行修改原有标签



   kubectl label pods client release=stable --overwrite

-  删除指定的 nodes 上的标签，使用标签名称加 - 符号



   kubectl label nodes node2 disktype-

-  许多资源支持内嵌字段来定义其使用的标签选择器，例如 service 关联 pod
   时候：

```
   matchLabels: 直接给定键值
   matchExpressions: 基于给定的表达式来定义使用标签选择器: {key:"KEY",operator:"OPERATOR",value:[VAL1,VAL2,...]}
       使用 key 与 value 进行 operator 运算, 复合条件的才被选择
       操作符:
           In, NotIn: 其 value 列表必须有值
           Exists, NotExists: 其 value 必须为空
```

-  k8s 中很多对象都可以打标签，例如给 nodes
   打一个标记，随后在添加资源时候就可以让资源对节点有倾向性了

```
   kubectl label nodes node2 disktype=ssd
   kubectl get nodes --show-labels
```

4.10 动态扩容
-------------

-  扩容一个集群的的 POD，下面命令表示修改 deployment 控制器下的
   nginx-deply 容器的副本数量为2

```
   kubectl scale --replicas=5 deployment nginx-deploy
   kubectl scale deployment/nginx-deployment --replicas=2
   kubectl autoscale deployment <deployment-name> --min=2 --max=5 --cpu-percent=80
```

4.11 滚动升级
-------------

-  更换 nginx-deploy 这个控制器下的 nginx-deploy 容器镜像为
   ikubernetes/myapp:v2



   kubectl set image deployment nginx-deploy nginx-deploy=ikubernetes/myapp:v2

-  查看更新的过程，直到 5 个容器中运行的镜像全部更新完



   kubectl rollout status deployment nginx-deploy


```
   [root@node1 ~]# kubectl rollout status deployment nginx-deploy
   Waiting for deployment "nginx-deploy" rollout to finish: 3 out of 5 new replicas have been updated...
   Waiting for deployment "nginx-deploy" rollout to finish: 3 out of 5 new replicas have been updated...
   Waiting for deployment "nginx-deploy" rollout to finish: 3 out of 5 new replicas have been updated...
   Waiting for deployment "nginx-deploy" rollout to finish: 3 out of 5 new replicas have been updated...
   Waiting for deployment "nginx-deploy" rollout to finish: 3 out of 5 new replicas have been updated...
   Waiting for deployment "nginx-deploy" rollout to finish: 4 out of 5 new replicas have been updated...
   Waiting for deployment "nginx-deploy" rollout to finish: 4 out of 5 new replicas have been updated...
   Waiting for deployment "nginx-deploy" rollout to finish: 4 out of 5 new replicas have been updated...
   Waiting for deployment "nginx-deploy" rollout to finish: 4 out of 5 new replicas have been updated...
   Waiting for deployment "nginx-deploy" rollout to finish: 4 out of 5 new replicas have been updated...
   Waiting for deployment "nginx-deploy" rollout to finish: 2 old replicas are pending termination...
   Waiting for deployment "nginx-deploy" rollout to finish: 2 old replicas are pending termination...
   Waiting for deployment "nginx-deploy" rollout to finish: 2 old replicas are pending termination...
   Waiting for deployment "nginx-deploy" rollout to finish: 1 old replicas are pending termination...
   Waiting for deployment "nginx-deploy" rollout to finish: 1 old replicas are pending termination...
   Waiting for deployment "nginx-deploy" rollout to finish: 1 old replicas are pending termination...
   Waiting for deployment "nginx-deploy" rollout to finish: 4 of 5 updated replicas are available...
   deployment "nginx-deploy" successfully rolled out
```

-  回滚操作，不指定任何的镜像则为上一个版本的镜像

```
   kubectl rollout undo deployment/nginx-deployment --to-revision=2
   kubectl rollout undo deployment nginx-deploy
```

..

   如果防止更新过程中被调度，那么就需要学习就绪性检测才能实现

4.12 get event （获取事件）
---------------

 ```bash
    kubectl get events  --field-selector involvedObject.kind=Service --sort-by='.metadata.creationTimestamp'
    kubectl get event --all-namespaces  --field-selector involvedObject.name=$po
```

4.13 logs(排查日志)
-------------

-  查看一个 pod 的某个容器的运行日志

```
   kubectl logs pod-demo busybox
```

4.14 exec (连入 POD 容器)
------------------

```
   kubectl exec -it pod-demo -c myapp -- /bin/sh
```

## 4.15 delete （删除资源）

有时 删除pv/pvc时会有问题,这个使用得加2个命令参数`--grace-period=0 --force `

```bash
   # 删除所有失败的pod
  kubectl get po --all-namespaces --field-selector 'status.phase==Failed'
  kubectl delete po  --field-selector 'status.phase==Failed'
  #模糊删除pod
  key=
  kgpo -n default | grep $key | awk '{print $1}' | xargs kubectl delete po -n1 -n default
  
```

## 4.16 其他命令

```bash
   kubectl api-resources --namespaced=false
   kubectl api-resources --namespaced=true
   # -R表示递归目录下所有配置
   kubectl apply -R -f configs/
   kubectl drain <node-name>
   kubectl taint nodes node1 key=value:NoSchedule
   kubectl cluster-info dump
   kubectl delete po -l app=onekey-ali-web -n=$(namespace)
   kubectl top pod
   kubectl delete deployment,services -l app=nginx 
   kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
    # deployment "nginx-deployment" image updated
    kubectl set image deploy monitorapi-deployment  monitorapi=registry-vpc.cn-shenzhen.aliyuncs.com/amiba/monitorapi:1.2.4 -n=java









    
```

## 4.20 kubectl 拓展

推荐几个工具。

### [kubectx](https://github.com/ahmetb/kubectx)

kubectx:用来切换集群的访问

kubens:用来切换默认的namespace

### [kubectl-aliases](https://github.com/ahmetb/kubectl-aliases)

`kubectl`命令别名

### 自动完成

zsh

```bash
source <(kubectl completion zsh)  # setup autocomplete in zsh into the current shell
echo "if [ $commands[kubectl] ]; then source <(kubectl completion zsh); fi" >> ~/.zshrc # add autocomplete permanently to your zsh shell
```

其他的方式见[kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

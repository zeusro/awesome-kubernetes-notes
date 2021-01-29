
二十二 K8S包管理器
==================

Helm 是 Deis 开发的一个用于 Kubernetes 应用的包管理工具，主要用来管理
Charts。有点类似于 Ubuntu 中的 APT 或 CentOS 中的 YUM。

Helm Chart 是用来封装 Kubernetes 原生应用程序的一系列 YAML
文件。可以在你部署应用的时候自定义应用程序的一些
Metadata，以便于应用程序的分发。

对于应用发布者而言，可以通过 Helm
打包应用、管理应用依赖关系、管理应用版本并发布应用到软件仓库。

对于使用者而言，使用 Helm
后不用需要编写复杂的应用部署文件，可以以简单的方式在 Kubernetes
上查找、安装、升级、回滚、卸载应用程序。

22.1 基础概念
-------------

-  Helm

Helm 是一个命令行下的客户端工具。主要用于 Kubernetes 应用程序 Chart
的创建、打包、发布以及创建和管理本地和远程的 Chart 仓库。

-  Tiller

Tiller 是 Helm 的服务端，部署在 Kubernetes 集群中。Tiller 用于接收 Helm
的请求，并根据 Chart 生成 Kubernetes 的部署文件（Helm 称为
Release），然后提交给 Kubernetes 创建应用。Tiller 还提供了 Release
的升级、删除、回滚等一系列功能。

-  Chart

Helm 的软件包，采用 TAR 格式。类似于 APT 的 DEB 包或者 YUM 的 RPM
包，其包含了一组定义 Kubernetes 资源相关的 YAML 文件。

-  Repoistory

Helm 的软件仓库，Repository 本质上是一个 Web
服务器，该服务器保存了一系列的 Chart
软件包以供用户下载，并且提供了一个该 Repository 的 Chart
包的清单文件以供查询，Helm 可以同时管理多个不同的 Repository。

-  Release

使用 helm install 命令在 Kubernetes 集群中部署的 Chart 称为
Release。Chart 与 Release 的关系类似于面向对象中的类与实例的关系。

22.2 Helm 工作原理
------------------

-  Chart Install 过程

::

   1. Helm 从指定的目录或者 TAR 文件中解析出 Chart 结构信息。
   2. Helm 将指定的 Chart 结构和 Values 信息通过 gRPC 传递给 Tiller。
   3. Tiller 根据 Chart 和 Values 生成一个 Release。
   4. Tiller 将 Release 发送给 Kubernetes 用于生成 Release。

-  Chart Update 过程

::

   1. Helm 从指定的目录或者 TAR 文件中解析出 Chart 结构信息。
   2. Helm 将需要更新的 Release 的名称、Chart 结构和 Values 信息传递给 Tiller。
   3. Tiller 生成 Release 并更新指定名称的 Release 的 History。
   4. Tiller 将 Release 发送给 Kubernetes 用于更新 Release。

-  Chart Rollback 过程

::

   1. Helm 将要回滚的 Release 的名称传递给 Tiller。
   2. Tiller 根据 Release 的名称查找 History。
   3. Tiller 从 History 中获取上一个 Release。
   4. Tiller 将上一个 Release 发送给 Kubernetes 用于替换当前 Release。

-  Chart 处理依赖说明

::

   Tiller 在处理 Chart 时，直接将 Chart 以及其依赖的所有 Charts 合并为一个 Release，同时传递给 Kubernetes。
   因此 Tiller 并不负责管理依赖之间的启动顺序。Chart 中的应用需要能够自行处理依赖关系。

22.3 部署 Helm
--------------

官方 github 地址：https://github.com/helm/helm

-  下载二进制版本，解压并安装 helm

::

   $ wget https://storage.googleapis.com/kubernetes-helm/helm-v2.13.1-linux-amd64.tar.gz
   $ tar xf helm-v2.13.1-linux-amd64.tar.gz
   $ mv helm /usr/local/bin/

-  初始化 tiller 时候会自动读取 ~/.kube 目录，所以需要确保 config
   文件存在并认证成功
-  tiller 配置 rbac，新建 rbac-config.yaml，并应用



   https://github.com/helm/helm/blob/master/docs/rbac.md    # 在这个页面中找到 rbac-config.yaml 



   $ kubectl apply -f tiller-rbac.yaml

-  初始化 tiller 时候会自动读取 ~/.kube 目录，所以需要确保 config
   文件存在并认证成功



   $ helm init --service-account tiller

-  添加 incubator 源



   $ helm repo add incubator https://aliacs-app-catalog.oss-cn-hangzhou.aliyuncs.com/charts-incubator/
   $ helm repo update

-  安装完成，查看版本



   $ helm version



   Client: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
   Server: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}

-  helm 官方可用的 chart 仓库



   http://hub.kubeapps.com/

-  命令基本使用



   completion  # 为指定的shell生成自动完成脚本（bash或zsh）
   create      # 创建一个具有给定名称的新 chart
   delete      # 从 Kubernetes 删除指定名称的 release
   dependency  # 管理 chart 的依赖关系
   fetch       # 从存储库下载 chart 并（可选）将其解压缩到本地目录中
   get         # 下载一个命名 release
   help        # 列出所有帮助信息
   history     # 获取 release 历史
   home        # 显示 HELM_HOME 的位置
   init        # 在客户端和服务器上初始化Helm
   inspect     # 检查 chart 详细信息
   install     # 安装 chart 存档
   lint        # 对 chart 进行语法检查
   list        # releases 列表
   package     # 将 chart 目录打包成 chart 档案
   plugin      # 添加列表或删除 helm 插件
   repo        # 添加列表删除更新和索引 chart 存储库
   reset       # 从集群中卸载 Tiller
   rollback    # 将版本回滚到以前的版本
   search      # 在 chart 存储库中搜索关键字
   serve       # 启动本地http网络服务器
   status      # 显示指定 release 的状态
   template    # 本地渲染模板
   test        # 测试一个 release
   upgrade     # 升级一个 release
   verify      # 验证给定路径上的 chart 是否已签名且有效
   version     # 打印客户端/服务器版本信息
   dep         # 分析 Chart 并下载依赖

-  指定 values.yaml 部署一个 chart



   helm install --name els1 -f values.yaml stable/elasticsearch

-  升级一个 chart



   helm upgrade --set mysqlRootPassword=passwd db-mysql stable/mysql

-  回滚一个 chart



   helm rollback db-mysql 1

-  删除一个 release



   helm delete --purge db-mysql

-  只对模板进行渲染然后输出，不进行安装



   helm install/upgrade xxx --dry-run --debug

22.4 Chart文件组织
------------------



   myapp/                               # Chart 目录
   ├── charts                           # 这个 charts 依赖的其他 charts，始终被安装
   ├── Chart.yaml                       # 描述这个 Chart 的相关信息、包括名字、描述信息、版本等
   ├── templates                        # 模板目录
   │   ├── deployment.yaml              # deployment 控制器的 Go 模板文件
   │   ├── _helpers.tpl                 # 以 _ 开头的文件不会部署到 k8s 上，可用于定制通用信息
   │   ├── ingress.yaml                 # ingress 的模板文件
   │   ├── NOTES.txt                    # Chart 部署到集群后的一些信息，例如：如何使用、列出缺省值
   │   ├── service.yaml                 # service 的 Go 模板文件
   │   └── tests
   │       └── test-connection.yaml
   └── values.yaml                      # 模板的值文件，这些值会在安装时应用到 GO 模板生成部署文件

22.5 使用 Helm + Ceph 部署 EFK
------------------------------

本文使用 K8S 集群上运行 EFK，使用 Ceph 集群作为 ElasticSearch
集群的持久存储。

用到知识有：Storage Class、PVC、Helm，另外，很多服务镜像需要翻墙。

helm install 阻塞过程会下载镜像可能会比较慢。

helm
里面有很多可以定制的项目，这里我就不定制了，反正我的资源也够用，懒得调了。

22.6 Storage Class
------------------



   ---
   apiVersion: v1
   kind: Secret
   metadata:
     name: ceph-admin-secret
     namespace: kube-system
   type: "kubernetes.io/rbd"
   data:
     # ceph auth get-key client.admin | base64
     key: QVFER3U5TmMQNXQ4SlJBAAhHMGltdXZlNFZkUXAvN2tTZ1BENGc9PQ==


   ---
   apiVersion: v1
   kind: Secret
   metadata:
     name: ceph-secret
     namespace: kube-system
   type: "kubernetes.io/rbd"
   data:
     # ceph auth get-key client.kube | base64
     key: QVFCcUM5VmNWVDdQCCCCWR1NUxFNfVKeTAiazdUWVhOa3N2UWc9PQ==


   ---
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: ceph-rbd
   provisioner: ceph.com/rbd
   reclaimPolicy: Retain
   parameters:
     monitors: 172.16.100.9:6789
     pool: kube
     adminId: admin
     adminSecretName: ceph-admin-secret
     adminSecretNamespace: kube-system
     userId: kube
     userSecretName: ceph-secret
     userSecretNamespace: kube-system
     fsType: ext4
     imageFormat: "2"
     imageFeatures: "layering"

22.7 Helm Elasticsearch
-----------------------

-  下载 elasticsearch 的 StatfullSet 的 chart



   helm fetch stable/elasticsearch

-  编辑 values.yaml，修改 storageClass 指向上面创建的 storageClass



   storageClass: "ceph-rbd"

-  使用 helm 指定 values.yaml 部署 elasticsearch



   helm install --name els1 -f values.yaml stable/elasticsearch

-  安装后查看，调试直到全部处于 READY 状态



   $ kubectl get pods
   NAME                                         READY   STATUS    RESTARTS   AGE
   els1-elasticsearch-client-55696f5bdd-qczbf   1/1     Running   1          78m
   els1-elasticsearch-client-55696f5bdd-tdwdc   1/1     Running   1          78m
   els1-elasticsearch-data-0                    1/1     Running   1          78m
   els1-elasticsearch-data-1                    1/1     Running   1          56m
   els1-elasticsearch-master-0                  1/1     Running   1          78m
   els1-elasticsearch-master-1                  1/1     Running   1          53m
   els1-elasticsearch-master-2                  1/1     Running   1          52m
   rbd-provisioner-9b8ffbcc-nxdjd               1/1     Running   2          81m

-  也可以使用 helm 命令查看



   $ helm status els1

   LAST DEPLOYED: Sun May 12 16:28:56 2019
   NAMESPACE: default
   STATUS: DEPLOYED

   RESOURCES:
   ==> v1/ConfigMap
   NAME                     DATA  AGE
   els1-elasticsearch       4     88m
   els1-elasticsearch-test  1     88m

   ==> v1/Pod(related)
   NAME                                        READY  STATUS   RESTARTS  AGE
   els1-elasticsearch-client-55696f5bdd-qczbf  1/1    Running  1         88m
   els1-elasticsearch-client-55696f5bdd-tdwdc  1/1    Running  1         88m
   els1-elasticsearch-data-0                   1/1    Running  1         88m
   els1-elasticsearch-data-1                   1/1    Running  1         66m
   els1-elasticsearch-master-0                 1/1    Running  1         88m
   els1-elasticsearch-master-1                 1/1    Running  1         63m
   els1-elasticsearch-master-2                 1/1    Running  1         62m

   ==> v1/Service
   NAME                          TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)   AGE
   els1-elasticsearch-client     ClusterIP  10.98.197.185  <none>       9200/TCP  88m
   els1-elasticsearch-discovery  ClusterIP  None           <none>       9300/TCP  88m

   ==> v1/ServiceAccount
   NAME                       SECRETS  AGE
   els1-elasticsearch-client  1        88m
   els1-elasticsearch-data    1        88m
   els1-elasticsearch-master  1        88m

   ==> v1beta1/Deployment
   NAME                       READY  UP-TO-DATE  AVAILABLE  AGE
   els1-elasticsearch-client  2/2    2           2          88m

   ==> v1beta1/StatefulSet
   NAME                       READY  AGE
   els1-elasticsearch-data    2/2    88m
   els1-elasticsearch-master  3/3    88m


   NOTES:
   The elasticsearch cluster has been installed.

   Elasticsearch can be accessed:

     * Within your cluster, at the following DNS name at port 9200:

       els1-elasticsearch-client.default.svc

     * From outside the cluster, run these commands in the same shell:

       export POD_NAME=$(kubectl get pods --namespace default -l "app=elasticsearch,component=client,release=els1" -o jsonpath="{.items[0].metadata.name}")
       echo "Visit http://127.0.0.1:9200 to use Elasticsearch"
       kubectl port-forward --namespace default $POD_NAME 9200:9200

-  启动一个临时的容器，解析集群地址，测试集群信息，查看集群节点



   $ kubectl run cirros1 --rm -it --image=cirros -- /bin/sh

   / # nslookup els1-elasticsearch-client.default.svc
   Server:    10.96.0.10
   Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

   Name:      els1-elasticsearch-client.default.svc
   Address 1: 10.98.197.185 els1-elasticsearch-client.default.svc.cluster.local
   / # curl els1-elasticsearch-client.default.svc.cluster.local:9200/_cat/nodes
   10.244.2.28  7 96 2 0.85 0.26 0.16 di - els1-elasticsearch-data-0
   10.244.1.37  7 83 1 0.04 0.06 0.11 di - els1-elasticsearch-data-1
   10.244.2.25 19 96 2 0.85 0.26 0.16 i  - els1-elasticsearch-client-55696f5bdd-tdwdc
   10.244.2.27 28 96 2 0.85 0.26 0.16 mi * els1-elasticsearch-master-2
   10.244.1.39 19 83 1 0.04 0.06 0.11 i  - els1-elasticsearch-client-55696f5bdd-qczbf
   10.244.2.29 21 96 2 0.85 0.26 0.16 mi - els1-elasticsearch-master-1
   10.244.1.38 23 83 1 0.04 0.06 0.11 mi - els1-elasticsearch-master-0

22.8 Helm fluentd-elasticsearch
-------------------------------

-  安装 kiwigrid 源



   helm repo add kiwigrid https://kiwigrid.github.io

-  下载 fluentd-elasticsearch



   helm fetch kiwigrid/fluentd-elasticsearch

-  获取集群地址



   els1-elasticsearch-client.default.svc.cluster.local:9200

-  编辑修改 values.yaml，指定 elasticsearch 集群的位置



   elasticsearch:
     host: 'els1-elasticsearch-client.default.svc.cluster.local'
     port: 9200

-  修改对污点的容忍程度，使其容忍 Master 节点的污点，也运行在 Master
   节点上收集信息



   tolerations: 
     - key: node-role.kubernetes.io/master
       operator: Exists
       effect: NoSchedule

-  如果使用 prometheus 监控应该打开 prometheusRole 规则



   podAnnotations:
     prometheus.io/scrape: "true"
     prometheus.io/port: "24231"
     
   service:
     type: ClusterIP
     ports:
       - name: "monitor-agent"
         port: 24231

-  使用 helm 指定 values.yaml 部署 fluentd-elasticsearch



   helm install --name flu1 -f values.yaml kiwigrid/fluentd-elasticsearch

-  查看状态 flu1 这个 helm 服务的运行状态



   [root@master fluentd-elasticsearch]# helm status flu1
   LAST DEPLOYED: Sun May 12 18:13:12 2019
   NAMESPACE: default
   STATUS: DEPLOYED

   RESOURCES:
   ==> v1/ClusterRole
   NAME                        AGE
   flu1-fluentd-elasticsearch  17m

   ==> v1/ClusterRoleBinding
   NAME                        AGE
   flu1-fluentd-elasticsearch  17m

   ==> v1/ConfigMap
   NAME                        DATA  AGE
   flu1-fluentd-elasticsearch  6     17m

   ==> v1/DaemonSet
   NAME                        DESIRED  CURRENT  READY  UP-TO-DATE  AVAILABLE  NODE SELECTOR  AGE
   flu1-fluentd-elasticsearch  3        3        3      3           3          <none>         17m

   ==> v1/Pod(related)
   NAME                              READY  STATUS   RESTARTS  AGE
   flu1-fluentd-elasticsearch-p49fc  1/1    Running  1         17m
   flu1-fluentd-elasticsearch-q5b9k  1/1    Running  0         17m
   flu1-fluentd-elasticsearch-swfvt  1/1    Running  0         17m

   ==> v1/Service
   NAME                        TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)    AGE
   flu1-fluentd-elasticsearch  ClusterIP  10.106.106.209  <none>       24231/TCP  17m

   ==> v1/ServiceAccount
   NAME                        SECRETS  AGE
   flu1-fluentd-elasticsearch  1        17m


   NOTES:
   1. To verify that Fluentd has started, run:

     kubectl --namespace=default get pods -l "app.kubernetes.io/name=fluentd-elasticsearch,app.kubernetes.io/instance=flu1"

   THIS APPLICATION CAPTURES ALL CONSOLE OUTPUT AND FORWARDS IT TO elasticsearch . Anything that might be identifying,
   including things like IP addresses, container images, and object names will NOT be anonymized.
   2. Get the application URL by running these commands:
     export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=fluentd-elasticsearch,app.kubernetes.io/instance=flu1" -o jsonpath="{.items[0].metadata.name}")
     echo "Visit http://127.0.0.1:8080 to use your application"
     kubectl port-forward $POD_NAME 8080:80

-  是否生成了索引，直接使用访问 elasticsearch 的 RESTfull API 接口。



   $ kubectl run cirros1 --rm -it --image=cirros -- /bin/sh
   / # curl els1-elasticsearch-client.default.svc.cluster.local:9200/_cat/indices
   green open logstash-2019.05.10 a2b-GyKsSLOZPqGKbCpyJw 5 1   158 0 84.2kb   460b
   green open logstash-2019.05.09 CwYylNhdRf-A5UELhrzHow 5 1 71418 0 34.3mb 17.4mb
   green open logstash-2019.05.12 5qRFpV46RGG_bWC4xbsyVA 5 1 34496 0 26.1mb 13.2mb

22.9 Helm kibana
----------------

-  下载 stable/kibana



   helm fetch stable/kibana

-  编辑 values.yaml，修改 elasticsearch 指向 elasticsearch 集群的地址



   elasticsearch.hosts: http://els1-elasticsearch-client.default.svc.cluster.local:920

-  修改 service 的工作模式，使得可以从集群外部访问



   service:
     type: NodePort

-  使用 helm 指定 values.yaml 部署 kibana



   helm install --name kib1 -f values.yaml stable/kibana

-  获取 service 端口



   $ kubectl get svc
   NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
   els1-elasticsearch-client      ClusterIP   10.98.197.185   <none>        9200/TCP        4h51m
   els1-elasticsearch-discovery   ClusterIP   None            <none>        9300/TCP        4h51m
   flu1-fluentd-elasticsearch     ClusterIP   10.101.97.11    <none>        24231/TCP       157m
   kib1-kibana                    NodePort    10.103.7.215    <none>        443:31537/TCP   6m50s
   kubernetes                     ClusterIP   10.96.0.1       <none>        443/TCP         3d4h

-  由于 service 工作在 NodePort 模式下，所以可以在集群外部访问了



   172.16.100.6:31537

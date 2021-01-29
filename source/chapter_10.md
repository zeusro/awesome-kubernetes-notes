
十 POD 存储卷
=============

大部分有状态的应用都有持久存储，在 Docker
上我们将容器所需要的存储卷放在宿主机上，但是 k8s 上不行，因为 POD
会被在不同的 node 节点上创建删除，所以 k8s
需要一套另外的存储卷机制，它能脱离节点为整个集群提供持久存储。

k8s 提供了多种不同的存储卷，k8s 中存储卷属于 POD 而不是容器，POD
可以挂载，POD 为什么能有存储卷呢？这是因为在所有节点上运行了一个 Pause
的镜像，它是 POD 的基础架构容器，它拥有存储卷，同一个 POD
内的所有容器是一个网络名称空间的。

10.1 卷的类型
-------------

   查看 POD 支持的存储类型：kubectl explain pods.spec.volumes

1. HostPath：在节点本地新建一个路径，与容器建立关联关系，但节点挂了的数据也不存在了，所以也不具有持久性，容器被调度到别的
   node 时候不能跨节点使用HostPath。
2. Local：直接使用节点的设备、也支持一个目录类似于 HostPath。
3. EmptyDir：只在节点本地使用，一旦 POD
   删除，存储卷也会删除，它不具有持久性，当临时目录或者缓存。
4. 网络存储：iSCSI、NFS、Cifs、glusterfs、cephfs、EBS（AWS)、Disk（Azone）

10.2 容器挂载选项
-----------------

在 K8S 中卷是属于 POD 的，而不是容器，所以卷的定义在 POD 中，一个 POD
中可以定义多个卷。

-  在 POD 中挂载使用，kubectl explain pods.spec.containers.volumeMounts



   apiVersion: v1
   kind: Pod
   metadata:
     name: myapp
     namespace: default
     labels:
       app: myapp
   spec:
     containers:
     - name: myapp
       image: ikubernetes/myapp:v1
       volumeMounts        <[]Object>  # 卷挂载对象
         mountPath         <string>    # 挂载路径
         mountPropagation  <string>    # 确定挂载如何从主机传播到容器
         name              <string>    # 挂载哪个卷
         readOnly          <boolean>   # 是否只读挂载
         subPath           <string>    # 挂载在子路径下
         subPathExpr       <string>    # 与 subPath 类似，挂载在子路径下，不同的是可以使用 $(VAR_NAME) 表示容器扩展这个变量

10.3 节点存储
-------------

10.3.1 hostpath存储卷
~~~~~~~~~~~~~~~~~~~~~

在宿主机的路径挂载到 POD 上，POD 删除后，卷数据是不会随之删除的，但如果
node 节点挂掉，那么数据有可能丢失，如果 POD
被调度到其他的节点，那么原来卷的数据就访问不到了。

   https://kubernetes.io/docs/concepts/storage/volumes/#hostpath

-  定义参数，kubectl explain pods.spec.volumes.hostPath



   path    <string>  # 主机上目录的路径。 如果路径是符号链接，则会跟随真实路径的链接。
   type    <string>  # 见下表

+----------------+-----------------------------------------------------+
| 值             | 行为                                                |
+================+=====================================================+
|                | 空字符串（默认）用于向后兼容，这意味着在安装hostPath卷之前不会执行任何检查。 |
+----------------+-----------------------------------------------------+
| ``DirectoryOrC | 如果给定路径中不存在任何内容，则将根据需要创建一个空目录，权限设置为0755，与Kubelet具有相同 |
| reate``        | 的组和所有权。                                      |
+----------------+-----------------------------------------------------+
| ``Directory``  | 目录必须存在于给定路径中                            |
+----------------+-----------------------------------------------------+
| ``FileOrCreate | 如果给定路径中不存在任何内容，则会根据需要创建一个空文件，权限设置为0644，与Kubelet具有相同 |
| ``             | 的组和所有权。                                      |
+----------------+-----------------------------------------------------+
| ``File``       | 文件必须存在于给定路径中                            |
+----------------+-----------------------------------------------------+
| ``Socket``     | UNIX套接字必须存在于给定路径中                      |
+----------------+-----------------------------------------------------+
| ``CharDevice`` | 字符设备必须存在于给定路径中                        |
+----------------+-----------------------------------------------------+
| ``BlockDevice` | 块设备必须存在于给定路径中                          |
| `              |                                                     |
+----------------+-----------------------------------------------------+

-  示例



   apiVersion: v1
   kind: Pod
   metadata:
     name: myapp
     namespace: default
     labels:
       app: myapp
   spec:
     containers:
     - name: myapp
       image: ikubernetes/myapp:v1
       volumeMounts:                       # 容器挂载哪些卷
       - name: webstore                    # 挂载哪个卷
         mountPath: /usr/share/nginx/html  # 挂载到容器内哪个目录
         readOnly: false                   # 是否只读
     volumes:                              # 存储卷属于POD的（不属于容器)
     - name: webstore                      # 存储卷对象名字
       hostPath:                           # hostpath 类型的存储卷对象
         path: /data/myapp                 # 处于宿主机的目录
         type: DirectoryOrCreate           # 不存在则创建

10.3.2 gitRepo卷
~~~~~~~~~~~~~~~~

将 git 仓库的内容当作存储使用，在 POD
创建时候连接到仓库，并拉取仓库，并将它挂载到容器内当作一个存储卷。

它其实是建立在 emptyDir 的基础上，但是对卷的操作不会同步到 gitrepo 上。

10.3.3 emptyDir缓存卷
~~~~~~~~~~~~~~~~~~~~~

它使用宿主机一个目录作为挂载点，随着 POD
生命周期的结束，其中的数据也会丢失，但是它有一个非常大的优点就是可以使用内存当作存储空间挂载使用。

它可以用在 POD 中两个容器中有一些数据需要共享时候选用。

-  定义 emptyDir 参数，\ ``kubectl explain pods.spec.volumes.emptyDir``



   medium      <string>    # 使用 "" 表示使用 Disk 来存储，使用 Memory 表示使用内存
   sizeLimit   <string>    # 限制存储空间的大小

-  使用示例



   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-volume-demo
     namespace: default
     labels:
       app: myapp
       tier: frontend
   spec:
     volumes:
       - name: html
         emptyDir: {}      # 使用磁盘，且没有容量限制
     containers:
       - name: myapp
         image: ikubernetes/myapp:v1
         imagePullPolicy: IfNotPresent
         volumeMounts:
           - name: html
             mountPath: /usr/share/nginx/html/
         ports:
           - name: http
             containerPort: 80
           - name: https
             containerPort: 443
       - name: busybox
         image: busybox:latest
         imagePullPolicy: IfNotPresent
         volumeMounts:
           - name: html
             mountPath: /data/
         command:
           - "/bin/sh"
           - "-c"
           - "while true; do date >> /data/index.html; sleep 10; done"

-  使用示例



   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-volume-demo
     namespace: default
     labels:
       app: myapp
       tier: frontend
   spec:
     containers:
       - name: myapp
         image: ikubernetes/myapp:v1
         imagePullPolicy: IfNotPresent
         volumeMounts:
           - name: html
             mountPath: /usr/share/nginx/html/
         ports:
           - name: http
             containerPort: 80
           - name: https
             containerPort: 443
       - name: busybox
         image: busybox:latest
         imagePullPolicy: IfNotPresent
         volumeMounts:
           - name: html
             mountPath: /data/
         command:
           - "/bin/sh"
           - "-c"
           - "while true; do date >> /data/index.html; sleep 10; done"
     volumes:
       - name: html
         emptyDir:
           medium: ""
           sizeLimit: 1536Mi

10.4 网络存储
-------------

网络存储，就是脱离了节点生命周期的存储设备，即使 pod 被调度到别的 node
节点上，仍然可以挂载使用其中的数据。

10.4.1 nfs
~~~~~~~~~~

nfs 服务器是存在于集群之外的服务器，它不受 node 节点的影响，因而在 node
节点宕机后仍然能够提供持久存储给其他 POD。

-  在 k8s 的 node 找一个主机，安装配置 nfs 服务器并启动



   $ yum install nfs-utils                                                     # 安装 nfs 服务
   $ mkdir -p /data/volumes                                                    # 创建 volume 卷目录
   echo '/data/volumes  172.16.100.0/16(rw,no_root_squash)' >> /etc/exports    # 配置 nfs 服务器
   $ systemctl start nfs                                                       # 启动 nfs 服务器
   $ ss -tnl                                                                   # 确认监听端口，nfs 监听 TCP 2049 端口

-  在 k8s 集群的 node 节点安装 nfs 驱动，测试挂载是否正常



   $ yum install nfs-utils
   $ mount -t nfs 172.16.100.104:/data/volumes /mnt

-  定义 nfs 参数，kubectl explain pods.spec.volumes.nfs



   path      <string>       # nfs 服务器的路径
   readOnly  <boolean>      # 是否只读
   server    <string>       # nfs 服务器地址

-  使用示例



   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-vol-nfs-demo
     namespace: default
   spec:
     containers:
     - name: myapp
       image: ikubernetes/myapp:v1
       volumeMounts:
         - name: html
           mountPath: /usr/share/nginx/html/
     volumes:
       - name: html
         nfs:
           path: /data/volumes
           server: 172.16.100.104

10.5 分布式存储
---------------

分布式存储能提供脱离节点生命周期的存储，又比网络存储更加健壮，它是分布式的，有很强的高可用性，但是分布式存储配置复杂，在由
NFS 提供的网络储存中，用户需要知道分配给 POD 的 NFS
存储的地址才能使用，而在由分布式提供的存储能力的存储上，用户需要充分了解该分布式存储的配置参数，才能够使用这个分布式存储。

由此 K8S 提供了 PV、PVC
两种机制，让普通用户无需关心底层存储参数的配置，只需要说明需要使用多大的持久存储，就可以了。

一般 PV 与 PVC 是一对绑定的，PV属于全局，PVC 属于某个名称空间，当一个 PV
被一个 PVC 绑定，别的名称空间 PVC 就不可以再绑定了。请求绑定某个 PV
就是由 PVC 来完成的，被 PVC 绑定的 PV 称作 PV 的绑定状态。

PVC 绑定了一个 PV，那么 PVC 所处名称空间定义的 POD 就可以使用
persistentVolumeClaim 类型的 volumes 了，然后容器就可以通过 volumeMounts
挂载 PVC 类型的卷了。

persistentVolumeClaim 卷是否允许多路读写，这取决于 PV
定义时候的读写特性：单路读写、多路读写、多路只读。

如果某个 POD 不在需要了，我们把它删除了、同时也删除了 PVC、那么此时 PV
还可以有自己的回收策略： delete删除PV、Retain什么都不做。

10.5.1 PersistentVolume
~~~~~~~~~~~~~~~~~~~~~~~

由管理员添加的的一个存储的描述，是一个集群级别的全局资源，包含存储的类型，存储的大小和访问模式等。它的生命周期独立于Pod，例如当使用它的
Pod 销毁时对 PV 没有影响。

   见：kubectl explain PersistentVolume.spec

-  在 nfs 上定义存储，/etc/exports，并且导出 nfs 定义



   /data/volumes/v1    172.16.100.0/16(rw,no_root_squash)
   /data/volumes/v2    172.16.100.0/16(rw,no_root_squash)
   /data/volumes/v3    172.16.100.0/16(rw,no_root_squash)
   /data/volumes/v4    172.16.100.0/16(rw,no_root_squash)
   /data/volumes/v5    172.16.100.0/16(rw,no_root_squash)



   exportfs -arv

-  将 nfs 在 k8s 中定义为 PersistentVolume，详见：kubectl explain
   PersistentVolume.spec.nfs



   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: pv-001
     labels:
       name: pv001
   spec:
     accessModes:
       - ReadWriteMany
       - ReadWriteOnce
     capacity:
       storage: 1Gi
     nfs:
       path: /data/volumes/v1
       server: 172.16.100.104

   ---
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: pv-002
     labels:
       name: pv003
   spec:
     accessModes:
       - ReadWriteMany
       - ReadWriteOnce
     capacity:
       storage: 2Gi
     nfs:
       path: /data/volumes/v2
       server: 172.16.100.104

   ---
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: pv-003
     labels:
       name: pv003
   spec:
     accessModes:
       - ReadWriteMany
       - ReadWriteOnce
     capacity:
       storage: 3Gi
     nfs:
       path: /data/volumes/v3
       server: 172.16.100.104



   kubectl get persistentvolume
   NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
   pv-001   1Gi        RWO,RWX        Retain           Available                                   3m38s
   pv-002   2Gi        RWO,RWX        Retain           Available                                   3m38s
   pv-003   3Gi        RWO,RWX        Retain           Available                                   3m38s

10.5.2. PersistentVolumeClaim
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

是 Namespace 级别的资源，描述对 PV
的一个请求。请求信息包含存储大小，访问模式等。

-  定义 PVC，kubectl explain PersistentVolumeClaim.spec



   accessModes         <[]string>  # 设置访问模式
       ReadWriteOnce               # 单个节点以读写方式挂载
       ReadOnlyMany                # - 多节点以只读方式挂载
       ReadWriteMany               # - 多节点以读写方式挂载

   dataSource          <Object>    # 如果配置程序可以支持 Volume Snapshot 数据源，它将创建一个新卷，并且数据将同时还原到该卷。 
   resources           <Object>    # 资源表示 PersistentVolume 应具有的最小资源
   selector            <Object>    # 选择哪个 PersistentVolume
   storageClassName    <string>    # 存储类名称
   volumeMode          <string>    # 定义声明所需的 PersistentVolume 类型才能被选中
   volumeName          <string>    # 后端 PersistentVolume ，就是精确选择 PersistentVolume ，而不是使用 selector 来选定

-  在 volumes 中使用 PVC，kubectl explain
   pods.spec.volumes.persistentVolumeClaim



   persistentVolumeClaim
       claimName    <string>  # 在当前名称空间已经创建号的 PVC 名称
       readOnly     <boolean> # 是否只读

-  定义 PersistentVolumeClaim，详见：kubectl explain
   PersistentVolumeClaim.spec



   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: my-pvc
     namespace: default
   spec:
     accessModes:
       - ReadWriteMany        # 访问模式
     resources:               # 资源条件
       requests:              # 挑选 PV 时候必须满足的条件，不满足则一直等待
         storage: 2Gi         # 存储大小

-  在 pod 清单中定义 persistentVolumeClaim 类型的 volumes
   ，并在容器中挂载 volumeMounts。



   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-vol-nfs-demo
     namespace: default
   spec:
     containers:
     - name: myapp
       image: ikubernetes/myapp:v1
       volumeMounts:
         - name: html
           mountPath: /usr/share/nginx/html/
     volumes:
       - name: html
         persistentVolumeClaim:
           claimName: my-pvc            # 使用的 PVC 的名称

10.5.3 StorageClass
~~~~~~~~~~~~~~~~~~~

PVC 申请 PV 的时候，未必有符合条件的 PV，k8s 为我们准备了 StorageClass
可以在 PVC 申请 PV 的时候通过 StorageClass 动态生成 PV。

StorageClass 可以动态的到 CephFS 、NFS 等存储（或者云端存储）产生一个
PV，要求存储设备必须支持 RESTfull 风格的接口。

10.6 StorageClass Ceph RBD
--------------------------

10.6.1 配置 Ceph 储存池
~~~~~~~~~~~~~~~~~~~~~~~

-  创建 ceph 存储池



   yum install -y ceph-common                                                                   # 在所有节点安装 ceph-common



   ceph osd pool create kube 4096                                                               # 创建 pool
   ceph osd pool ls                                                                             # 查看 pool

   ceph auth get-or-create client.kube mon 'allow r' osd 'allow rwx pool=kube' -o /etc/ceph/ceph.client.kube.keyring
   ceph auth list                                                                               # 授权 client.kube 用户访问 kube 这个 pool

   scp /etc/ceph/ceph.client.kube.keyring node1:/etc/ceph/                                      # 将用户 keyring 文件拷贝到各个 ceph 节点
   scp /etc/ceph/ceph.client.kube.keyring node1:/etc/ceph/

10.6.2 安装 rbd-provisioner
~~~~~~~~~~~~~~~~~~~~~~~~~~~

-  1.12 版本后 kube-controller-manager 不再内置 rbd 命令，所以
   StorageClass 的 provisioner 而是通过外部的插件来实现



   https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd/deploy/rbac    # rbd-provisioner



   $ git clone https://github.com/kubernetes-incubator/external-storage.git                     # 下载 rbd-provisioner
   $ cat >>external-storage/ceph/rbd/deploy/rbac/clusterrole.yaml<<EOF                          # 允许 rbd-provisioner 访问 ceph 的密钥
     - apiGroups: [""]
       resources: ["secrets"]
       verbs: ["create", "get", "list", "watch"]
   EOF
   $ kubectl apply -f external-storage/ceph/rbd/deploy/rbac/                                    # 安装 rbd-provisioner

10.6.3 使用 StorageClass
~~~~~~~~~~~~~~~~~~~~~~~~

-  创建 CephX 验证 secret



   https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd/examples       # rbd-provisioner 使用 ceph rbd 的示例



   ---
   apiVersion: v1
   kind: Secret
   metadata:
     name: ceph-admin-secret
     namespace: kube-system
   type: "kubernetes.io/rbd"
   data:
     # ceph auth get-key client.admin | base64                                                  # 从这个命令中取得 keyring 认证的 base64 密钥串复制到下面
     key: QVFER3U5TmM1NXQ4SlJBQXhHMGltdXZlNFZkUXRvN2tTZ1BENGc9PQ==


   ---
   apiVersion: v1
   kind: Secret
   metadata:
     name: ceph-secret
     namespace: kube-system
   type: "kubernetes.io/rbd"
   data:
     # ceph auth get-key client.kube | base64                                                  # 从这个命令中取得 keyring 认证的 base64 密钥串复制到下面
     key: QVFCcUM5VmNWVDdQRlJBQWR1NUxFNzVKeThiazdUWVhOa3N2UWc9PQ==

-  创建 StorageClass 指向 rbd-provisioner，



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

-  创建 PersistentVolumeClaim



   ---
   kind: PersistentVolumeClaim
   apiVersion: v1
   metadata:
     name: ceph-rbd-pvc  data-kong-postgresql-0 
   spec:
     storageClassName: ceph-rbd
     accessModes:
     - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi

-  在 POD 中使用 PVC，最后在容器中挂载 PVC。



   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: ceph-sc-pvc-demo
     namespace: default
   spec:
     containers:
     - name: myapp
       image: ikubernetes/myapp:v1
       volumeMounts:
         - name: pvc-volume
           mountPath: /usr/share/nginx/html/
     volumes:
       - name: pvc-volume
         persistentVolumeClaim:
           claimName: ceph-rbd-pvc
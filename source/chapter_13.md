
十三 用户认证系统
=================

apiserver
是所有请求访问的网关接口，请求过程中，认证用于实现身份识别，授权用于实现权限检查，实际上，我们使用命令行：kubectl
apply -f ment.yaml，实际上是转换为 HTTP 协议向 apiserver
发起请求的，而认证是信息由 ~/.kube/config
这个文件提供的，这个文件记录了管理员权限的用户信息。

-  k8s 的 API 是 RESTfull 风格的，所以资源是由路径标明的，在 k8s
   中，资源只能属于两个地方：属于集群 或 属于名称空间。



   集群级别：namespace、pv
   名称空间：POD、deployment、daemonSet、 service、PCV

例如：请求 delfault 名称空间下的 myapp-deploy 控制器，就是下面的写法



   http://172.16.100.100/apis/apps/v1/namespaces/default/deployments/myapp-deploy

上面表示：http://172.16.100.100:6443 集群的 apis 下的 apps 组的 v1
版本的 namespaces 下寻找 default 下的 myapp-deploy 控制器

13.1 用户的类型
---------------

我们使用 kubectl 连接 k8s 集群进行控制，实际上是使用用户家目录下
.kube/config 这个文件中的用户连接到 apiserver 实现认证的，而有些 POD
（如：CoreDNS）也需要获取集群的信息，它们也需要连接到 k8s 集群中，所以
k8s 中用户的类型有两种：

1. 人类使用的用户：useraccount，处于用户家目录 .kube/config
   文件中，可使用 kubectl config –help 获取帮助创建
2. POD 使用的用户：serviceaccunt，是一种 k8s 对象，它可以使用 kubectl
   create serviceaccount –help 获取帮助创建

13.2 POD如何连接集群
--------------------

POD 需要使用 serviceaccount 连接并认证到集群，POD
之所以能够连接到集群是因为有一个内置的 service 将 POD 的请求代理至
apiserver 的地址了。

-  名字为 kubernetes 的 servie 为 POD 连接到 apiserver 提供了通信



   $ kubectl describe service kubernetes                            # 集群内部的 POD 与 apiserver 通信使用的 service ，但是注意 apiserver 需要认证的

   Name:              kubernetes
   Namespace:         default
   Labels:            component=apiserver
                      provider=kubernetes
   Annotations:       <none>
   Selector:          <none>
   Type:              ClusterIP
   IP:                10.96.0.1                                     # 集群内部访问 apiserver 的网关
   Port:              https  443/TCP
   TargetPort:        6443/TCP
   Endpoints:         172.16.100.101:6443                           # apiserver 工作的地址
   Session Affinity:  None
   Events:            <none>

13.3 serviceaccount 对象
------------------------

k8s 的认证有两种一种是：human user、一种是 serviceaccount，下面就是创建
serviceaccount 它是 POD 访问 apiserver 所用的一种对象，而 human
user，即使 kubectl 命令行通过读取 config 中的用户而认证到 apiserver 的。

-  创建一个 serviceaccount 对象，它会自动创建并关联一个 secret，这个
   serviceaccount 可以到 apiserver
   上进行认证，但是认证不代表有权限，所以需要授权



   $ kubectl create serviceaccount admin
   $ kubectl get secret

-  创建 POD 使用指定的 serviceaccount 对象

.. code:: yaml

   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-serviceaccount-demo
     namespace: default
     labels:
       app: myapp
       tier: frontend
   spec:
     containers:
       - name: nginx
         image: ikubernetes/myapp:v1
         ports:
           - name: http
             containerPort: 80
     serviceAccountName: admin

13.3.1 在 POD 中使用 serviceaccount
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

-  POD 连接 apiserver 时候，需要在清单中指定 serviceAccountName
   这个字段，详见：kubectl explain pods.spec

每个 POD 默认自带一个 volumes，这是一个 secret，这个存储卷保存着
default-token-bq2gn 用来访问 apiserver ，而这个 secret 权限仅仅能通过
api 访问当前 POD 自身的信息，如果想要一个 POD
拥有管理集群的权限，那么可以手动创建一个 secret 并通过 volumes 挂载到
POD 上。

serviceaccout 也属于标准的 k8s
对象，这个对象提供了账号信息，但是账号由没有权限需要 rbac 机制来决定。

13.4 kubectl 配置文件
---------------------

-  kubectl 配置文件解析，详见：kubectl config view

.. code:: yaml

   apiVersion: v1
   kind: Config
   clusters:                                             # 集群列表
   - cluster:                                            # 列表中的一个集群对象
       certificate-authority-data: DATA+OMITTED          # 服务器认证方式
       server: https://172.16.100.101:6443               # 集群的 apiserver 地址
     name: kubernetes                                    # 集群名称
   users:                                                # 用户列表
   - name: kubernetes-admin                              # 列表中的一个用户对象
     user:                                               # 
       client-certificate-data: REDACTED                 # 客户端证书
       client-key-data: REDACTED                         # 客户端私钥
   contexts:                                             # 上下文列表
   - context:                                            # 列表中的一个上下文对象
       cluster: kubernetes                               # 集群名称
       user: kubernetes-admin                            # 用户名称
     name: kubernetes-admin@kubernetes                   # 上下文名称
   current-context: kubernetes-admin@kubernetes          # 当前上下文
   preferences: {}

-  配置文件保存了：多个集群、多用户的配置，kubectl
   可以使用不同的用户访问不同的集群。



   集群列表：集群对象列表
   用户列表：用户对象列表
   上下文：是描述集群与用户的关系列表。
   当前上下文：表示当前使用哪个用户访问哪个集群



   自定义配置信息：详见：kubectl config  --help
   ca 和证书保存路径：/etc/kubernetes 保存了所有的 ca 和签发的证书信息。

13.5 添加证书用户到 config
--------------------------

k8s apiserver 认证方式有两种：ssl证书 和 token 认证，本次使用 ssl
证书创建用户

13.5.1 创建SSL证书用户
~~~~~~~~~~~~~~~~~~~~~~

-  创建连接 apiserver 的用户证书



   # 创建私钥
   (umask 077; openssl genrsa -out kaliarch.key 2048)

   # 生成证书签署请求，O 是组，CN 就是账号，这个账号被 k8s 用来识别身份，授权也需要授权这个账号
   openssl req -new -key kaliarch.key -out kaliarch.csr -subj "/CN=kaliarch"
   #penssl req -new -key kaliarch.key -out kaliarch.csr -subj "O=system:masters/CN=kaliarch/"

   # 使用 CA 签署证书，并且在 1800 天内有效
   openssl x509 -req -in kaliarch.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out kaliarch.crt -days 1800

   # 查看证书
   openssl x509 -in kaliarch.crt -text -noout

13.5.2 添加SSL证书用户到config
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

-  将 kaliarch 用户添加到 k8s 的 config 中，设置客户端证书为
   kaliarch.crt，设置客户端私钥为：kaliarch.key，使用 –embed-certs=true
   来隐藏这些机密信息



   kubectl config set-credentials kaliarch --client-certificate=./kaliarch.crt --client-key=./kaliarch.key --embed-certs=true

13.5.3 创建切换上下文
~~~~~~~~~~~~~~~~~~~~~

-  创建上下文对象，授权 kaliarch 用户访问名称为 kubernetes 的集群



   kubectl config set-context kaliarch@kubernetes --cluster=kubernetes --user=kaliarch

-  切换当前使用的上下文，到授权 kaliarch 到 kubernetes 的上下文上



   kubectl config use-context kaliarch@kubernetes

-  由于这个用户没有授权，所以这个用户是无法 get 到信息的，可以再切换回来



   $ kubectl get pods
   $ kubectl config use-context kubernetes-admin@kubernetes

13.6 创建新 config 文件
-----------------------

使用 kubectl config set-cluster 创建一个新的 config
文件，想要设定这个新创建的 config 文件可以使用
–kubeconfig=/tmp/test.conf 指明。

-  设置集群的连接的 ca 机构证书，–kubeconfig 可以指定 kubectl
   使用的配置文件位置，默认为用户家目录 .kube 目录中的 config



   kubectl config set-cluster k8s-cluster --server=https://172.16.100.101:6443 --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true --kubeconfig=/tmp/test.conf 

-  将 kaliarch 用户添加到 k8s 的 config 中，设置客户端证书为
   kaliarch.crt，设置客户端私钥为：kaliarch.key，使用 –embed-certs=true
   来隐藏这些机密信息



   kubectl config set-credentials kaliarch --client-certificate=./kaliarch.crt --client-key=./kaliarch.key --embed-certs=true

-  创建上下文对象，授权 kaliarch 用户访问名称为 kubernetes 的集群



   kubectl config set-context def-ns-admin@k8s-cluster --cluster=k8s-cluster --user=def-ns-admin --kubeconfig=/tmp/test.conf

-  切换当前使用的上下文，到授权 kaliarch 到 kubernetes 的上下文上



   kubectl config use-context def-ns-admin@k8s-cluster --kubeconfig=/tmp/test.con

13.7 基于 token 认证
--------------------

13.7.1 创建 serviceaccount
~~~~~~~~~~~~~~~~~~~~~~~~~~

-  为 POD 创建一个 serviceaccount 对象，它是 POD 访问 apiserver 的凭证



   kubectl create serviceaccount dashborad-admin -n kube-system

13.7.2 绑定集群管理员角色
~~~~~~~~~~~~~~~~~~~~~~~~~

-  创建 clusterrolebinding 将用户绑定至 cluster-admin
   集群管理员（最高权限）



   kubectl create clusterrolebinding dashborad-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashborad-admin

13.7.3 通过 serviceaccount 得到 Token
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

-  找到刚才创建的 serviceaccount 对象



   kubectl get secret -n kube-system

-  得到 serviceaccount 对象中的 Token



   kubectl describe secret -n kube-system dashborad-admin-token-skz95

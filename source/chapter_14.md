
十四 用户权限系统(RBAC)
=================

在 k8s 中的用户权限系统是使用 RBAC 模式的，RBAC 是 Role-Based AC
的缩写，全称：基于角色的访问控制。

我们可以让一个用户扮演一个角色，而这个角色拥有权限，而这个用户就拥有了这个权限，所以在
RBAC 中，用户授权就是授权某个角色。



   用户（user）：用户可以拥有某个角色。

   角色（role）：角色可以拥有某些许可。
       1. 操作
       2. 对象

   许可（permission）： 在一个对象上能施加的操作组合起来，称之为一个许可权限。

-  用户类型



   Human User：              # 用户账号
   Pod Service Account：     # 服务账号

-  角色类型



   - rule（角色）、rolebinding（角色绑定）
   - clausterrole（集群角色）、clusterrolebinding（集群角色绑定）

-  授权类型



   - 用户通过 rolebinding 去 bind rule，rolebinding 只能是当前命名空间中
   - 通过 clusterrolebinding 去 bind clausterrole，clusterrolebinding会在所有名称空间生效
   - 通过 rolebinding 去 bind clausterrole，由于 rolebinding 只在当前名称空间，所以 clausterrole 权限被限制为当前名称空间

-  通过 rolebinding 去 bind clausterrole 的好处



   如果有很多名称空间、如果用 rolebinding 绑定 rule，那么则需要在每个名称空间都定义 role
   如果使用 rolebinding 绑定一个 clausterrole ，由于 clausterrole 拥有所有名称空间的权限，而 rolebinding  只能绑定当前名称空间，那么就省去为每个名称空间都新建一个 role 的过程了。

14.1 权限列表
-------------



   kubectl get clusterrole admin -o yaml

14.2 创建 Role
--------------

-  命令行定义



   kubectl create role pods-reader --verb=get,list,watch --resource=pods

-  使用清单方式定义

.. code:: yaml

   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: pods-reder
     namespace: default
   rules:
   - apiGroups:                           # 对哪些 api 群组内的资源进行操作
     - ""
     resources:                           # 对哪些资源授权
     - pods
     verbs:                               # 授权做哪些操作
     - get
     - list
     - watch

14.3 创建 rolebinding
---------------------

-  使用 rolebinding 对象创建，用户与 role 的绑定



   kubectl create rolebinding kaliarch-read-pods --role=pods-reader --user=kaliarch

-  使用清单方式定义

.. code:: yaml

   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: kaliarch-read-pods
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: Role
     name: pods-reader
   subjects:
   - apiGroup: rbac.authorization.k8s.io
     kind: User
     name: kaliarch

-  切换用户和环境上下文



   $ kubectl config use-context kaliarch@kubernetes

-  测试用户是否拥有 get 权限



   kubectl get pods

14.4 创建 clusterrole
---------------------

-  命令行定义



   kubectl create clusterrole cluster-reader --verb=get,list,watch --resource=pods

-  使用清单方式定义



   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: cluster-reader
   rules:
   - apiGroups:
     - ""
     resources:
     - pods
     verbs:
     - get
     - list
     - watch

-  系统内置有非常多的 clusterrole，详见：kubectl get clusterrole



   NAME                                                                   AGE
   admin                                                                  5d16h
   cluster-admin                                                          5d16h
   cluster-reader                                                         4m32s
   edit                                                                   5d16h
   flannel                                                                5d6h
   system:aggregate-to-admin                                              5d16h
   system:aggregate-to-edit                                               5d16h
   system:aggregate-to-view                                               5d16h
   system:auth-delegator                                                  5d16h
   system:aws-cloud-provider                                              5d16h
   system:basic-user                                                      5d16h
   system:certificates.k8s.io:certificatesigningrequests:nodeclient       5d16h
   system:certificates.k8s.io:certificatesigningrequests:selfnodeclient   5d16h
   system:controller:attachdetach-controller                              5d16h
   system:controller:certificate-controller                               5d16h
   system:controller:clusterrole-aggregation-controller                   5d16h
   system:controller:cronjob-controller                                   5d16h
   system:controller:daemon-set-controller                                5d16h

14.5 创建 clusterrolebinding
----------------------------

-  命令行定义



   kubectl create clusterrolebinding kaliarch-read-all-pods --clusterrole=cluster-reader --user=kaliarch

-  清单定义

.. code:: yaml

   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: ClusterRoleBinding
   metadata:
     name: kaliarch-read-all-pods
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-reader
   subjects:
   - apiGroup: rbac.authorization.k8s.io
     kind: User
     name: kaliarch

-  切换用户和环境上下文



   $ kubectl config use-context kaliarch@kubernetes

-  测试用户是否拥有 get 权限



   $ kubectl get pods -n kube-system
   $ kubectl config use-context kubernetes-admin@kubernetes

14.6 rolebinding 与 clusterrole
-------------------------------

如果使用 rolebinding 绑定一个 clausterrole ，由于 clausterrole
拥有所有名称空间的权限，而 rolebinding
只能绑定当前名称空间，那么就省去为每个名称空间都新建一个 role 的过程了。

-  命令定义



   $ kubectl create rolebinding kaliarch-cluster-reader --clusterrole=cluster-reader --user=kaliarch

-  清单定义



   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: kaliarch-admin
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: admin
   subjects:
   - apiGroup: rbac.authorization.k8s.io
     kind: User
     name: kaliarch

-  切换用户和环境上下文



   $ kubectl config use-context kaliarch@kubernetes

-  测试用户是否拥有 get 权限，由于使用了 rolebinding ，所以
   cluster-reader 被限制到当前命名空间



   $ kubectl get pods -n kube-system
   $ kubectl config use-context kubernetes-admin@kubernetes

14.7 RBAC授权
-------------

在 bind 授权的时候，可以绑定的用户主体有：user、group

-  使用 rolebinding 和 clusterrolebinding 绑定



   绑定到 user：表示只有这一个用户拥有 role 或者 clusterrole 的权限
   绑定到 group：表示这个组内的所有用户都具有了 role 或者 clusterrole 的权限

-  创建用户时候加入组，加入组后账户自动集成该组的权限


```
   # 创建私钥
   (umask 077; openssl genrsa -out kaliarch.key 2048)

   # 生成证书签署请求，O 是组，CN 就是账号，这个账号被 k8s 用来识别身份，授权也需要授权这个账号
   openssl req -new -key kaliarch.key -out kaliarch.csr -subj "O=system:masters/CN=kaliarch/"

   # 使用 CA 签署证书，并且在 1800 天内有效
   openssl x509 -req -in kaliarch.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out kaliarch.crt -days 1800

   # 查看证书
   openssl x509 -in kaliarch.crt -text -noout
```

## 14.8 总结

一句话总结`ServiceAccount`,`Role`,`RoleBinding`,`ClusterRole`,`ClusterRoleBinding`的关系：
**`ClusterRoleBinding`,`RoleBinding`是一种任命,认命被授权的对象(users, groups, or service accounts)能够有什么样的权限(Role,ClusterRole)**


最后用一个表格整理一下

|资源类型| 说明|
|---|---|
|ServiceAccount |一个虚名|
|service-account-token|ServiceAccount的身份象征 | 
|Role| 授予对单个命名空间中的资源访问权限| 
|RoleBinding|将赋予被授权对象和Role| 
|ClusterRole |可视为Role的超集,是从集群角度做的一种授权| 
|ClusterRoleBinding|将赋予被授权对象和ClusterRole| 

理解`kubernetes`RBAC的最简单办法,就是进入kube-system内部,看看各类集群资源是怎么定义的.
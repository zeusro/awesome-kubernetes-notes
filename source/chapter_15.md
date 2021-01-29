

十五 dashboard
==============

它作为 k8s 集群的附件存在，是 kubernetes
官方的项目之一，详见：https://github.com/kubernetes/dashboard

15.1 部署流程
-------------

-  为 dashboard 提供 ssl 证书


```
   # 生成私钥
   (umask 077; openssl genrsa -out dashboard.key 2048)

   # 生成一个自签证书，注意 CN 的值必须要与自己的域名完全一致
   openssl req -new -x509 -key dashboard.key -out dashboard.crt -subj "/O=dashboard/CN=k8s.dashboard.com"

   # 查看证书
   openssl x509 -in dashboard.crt -text -noout
```

-  下载 dashboard 的清单文件

   wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

-  为 dashboard 创建 secret 对象



   kubectl -n kube-system create secret generic kubernetes-dashboard-certs --from-file=dashboard.crt=./dashboard.crt --from-file=dashboard.key=./dashboard.key

-  修改 dashboard 清单中 service 的工作模式为 nodeport



   sed -i '/targetPort: 8443/a\ \ type: NodePort' kubernetes-dashboard.yaml

-  注释掉 kubernetes-dashboard.yaml 清单文件中的 Dashboard Secret
   这个证书的清单定义


```
   # ------------------- Dashboard Secret ------------------- #
   #apiVersion: v1
   #kind: Secret
   #metadata:
   #  labels:
   #    k8s-app: kubernetes-dashboard
   #  name: kubernetes-dashboard-certs
   #  namespace: kube-system
   #type: Opaque
```

-  部署 dashboard 清单



   kubectl apply -f kubernetes-dashboard.yaml

-  取得 service 运行的端口



   kubectl get service -n kube-system

-  使用 chrome 访问 dashboard



   https://172.16.100.102:31097/

15.2 使用令牌登录
-----------------

-  为 POD 创建一个 serviceaccount 对象，它是 POD 访问 apiserver 的凭证



   kubectl create serviceaccount dashborad-admin -n kube-system

-  创建 clusterrolebinding 将用户绑定至 cluster-admin
   集群管理员（最高权限）



   kubectl create clusterrolebinding dashborad-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashborad-admin

-  找到刚才创建的 serviceaccount 对象



   kubectl get secret -n kube-system

-  得到 serviceaccount 对象中的 Token



   kubectl describe secret -n kube-system dashborad-admin

15.3 分级管理
-------------

现在需要创建一个只能管理 default 名称空间的用户，那么我们可以用
rolebinding 去绑定 admin 这个 clusterrolue
对象，那么就获得了当前名称空间的管理员权限了。

-  创建 serviceaccount 登录



   kubectl create serviceaccount def-ns-admin -n default

-  使用 rolebinding 对象，将 default 名称空间的 def-ns-admin 这个
   serviceaccunt 与 admin 这个 clusterrole 绑定



   kubectl create rolebinding def-ns-admin --clusterrole=admin --serviceaccount=default:def-ns-admin

-  找到刚才创建的 serviceaccount 对象



   kubectl get secret -n kube-system

-  得到 serviceaccount 对象中的 Token



   kubectl describe secret def-ns-admin

15.4 配置文件认证
-----------------

与之前基于 SSL 证书的 config 文件不同，这次使用是基于 Token 的 config
文件，可以不用创建证书了，使用已有的 serviceaccount 对象的 token。

-  设置集群的连接的 ca 机构证书，–kubeconfig 可以指定 kubectl
   使用的配置文件位置，默认为用户家目录 .kube 目录中的 config



   kubectl config set-cluster k8s-cluster --server=https://172.16.100.101:6443 --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true --kubeconfig=/tmp/test.conf 

-  取得一个已经绑定角色的 serviceaccount 对象的 Token



   kubectl describe secret def-ns-admin

-  使用 Token 来创建配置文件中的用户



   kubectl config set-credentials def-ns-admin --token=<TOKEN> --kubeconfig=/tmp/test.conf

-  创建上下文对象，授权 kaliarch 用户访问名称为 kubernetes 的集群



   kubectl config set-context def-ns-admin@k8s-cluster --cluster=k8s-cluster --user=def-ns-admin --kubeconfig=/tmp/test.conf

-  切换当前使用的上下文，到授权 kaliarch 到 kubernetes 的上下文上



   kubectl config use-context def-ns-admin@k8s-cluster --kubeconfig=/tmp/test.conf

-  复制 /tmp/test.conf 这个文件到 dashboard 中就可以登录了


九 ingress 控制器
=================

如果 k8s 需要提供一个网站，并且这个站点需要以 https 访问，而
iptables/ipvs 工作在 4 层，客户发出的 ssl 请求根本不被解析就被调度到后端
POD了。解决方法有两个：

1. 可以在公有云的负载均衡器上配置上 ssl 证书。

2. 新建一个负载均衡器的 POD ，例如 nignx ，这个 POD
   共享主机的网络命名空间，也就是说可以直接通过 nodeip
   访问到负载均衡器，ssl 证书配置在这个负载均衡器上，对外连接为 https
   而对内的代理为 http 协议到 POD 网络的 POD 上。

-  存在的问题



   - 负载均衡器 POD 使用节点的网络名称空间, 那么它只   能在这个 node 节点上运行一个了,否则就出现端口冲突
   - 负载均衡器是代理 POD 卸载 ssl 证书的关键节点, 它不能只运行一个, 它需要在所有节点运行一个

-  解决方法



   - 负载均衡器使用 DaemonSet 在每个 node 节点运行一个,代理请求至 POD 网络的中的 POD 上
   - 如果集群节点非常的多,其实不必在每个 node 节点都必须运行一个负载均衡器 POD
   - 控制负载均衡器 POD 运行的数量可以通过 lables 指定运行那几个 node 节点上
   - 然后可以在负载均衡器 POD 所在的 node 节点上打上 "污点" 使其他的 POD 不会再被调度上来, 而只有负载均衡器 POD 可以容忍这些 "污点"

-  负载均衡器可选，按照优先级先后排序



   Envoy            # 云原生高性能服务代理,已从cncf毕业
   Traefik          # 为微服务而生的反向代理
   Nginx            # 改造后可以适用于微服务环境
   HAproxy          # 不推荐使用

新建一个 service 将需要代理的不同服务的 pod 分类

新建一个 ingress 资源，从 service 中取得分类结果，映射进 Envoy 中，重载
Envoy 软件。

9.1 ingress.spec 规范
---------------------

-  API 和 kind



   apiVersion: extensions

   kind: ingress

-  ingress.spec



   backend         # 后端有哪些 POD
   rules           # 调度规则
       host        # 虚拟主机
       http        # http 路径

9.2 ingress-nginx 代理
----------------------

-  后端 service 和 pods



   apiVersion: v1
   kind: Service
   metadata:
     name: service-ingress-myapp
     namespace: default
   spec:
     selector:
       app: myapp
       release: canary
     ports:
       - name: http
         port: 80
         targetPort: 80

   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: myapp
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

-  创建 ingress-nginx



   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.21.0/deploy/mandatory.yaml

-  让 ingress-nginx 在集群外部访问



   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.21.0/deploy/provider/baremetal/service-nodeport.yaml

-  创建 ingress 对象，它能将 ingress-nginx 与 service 关联，从而在
   service 后主机发生变动的时候，反应在 ingress-nginx
   这个容器的配置文件中



   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: ingress-deploy-myapp
     namespace: default
     annotations:
       kubernetes.io/ingress.class: "nginx"
   spec:
     rules:
       - host: myapp.kaliarch.com                       # 基于主机名的访问
         http:
           paths:
             - path:                                   # 空的时候代表根，访问根的时候映射到 backend
               backend:                                # 后端的 service 的配置
                 serviceName: service-ingress-myapp    # 关联 service 从而获取到后端主机的变动
                 servicePort: 80                       # 关联 service 的地址

-  查看 ingress-nginx 对外暴露的端口，这里为30080，和 30443 两个



   kubectl get service -n ingress-nginx

-  使用 nodeip + ingress-nginx 暴露端口访问，由于上面创建的 ingress
   为基于主机名称的，所以需要在访问时在 /etc/hosts 做好映射到 node。



   http://myapp.kaliarch.com:30080/index.html  

9.3 ingress-tomcat 代理
-----------------------

-  后端 service 和 pods



   apiVersion: v1
   kind: Service
   metadata:
     name: service-ingress-tomcat
     namespace: default
   spec:
     selector:
       app: tomcat
       release: canary
     ports:
       - name: http
         port: 8080
         targetPort: http
       - name: ajp
         port: 8009
         targetPort: ajp

   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: deploy-tomcat
     namespace: default
   spec:
     replicas: 4
     selector:
       matchLabels:
         app: tomcat
         release: canary
     template:
       metadata:
         labels:
           app: tomcat
           release: canary
       spec:
         containers:
           - name: tomcat
             image: tomcat:8.5.32-jre8-alpine
             ports:
               - name: http
                 containerPort: 8080
               - name: ajp
                 containerPort: 8009

-  制作自签名证书，让 ingress-nginx 带有证书来访问

```bash
   # 生成 key
   openssl genrsa -out tls.key 2048

   # 生成自签证书，CN=域名必须要与自己的域名完全一致
   openssl req -new -x509 -key tls.key -out tls.crt -subj /C=CN/ST=Beijing/L=Beijing/O=DevOps/CN=tomcat.kaliarch.com
```

-  创建 secret 证书对象，它是标准的 k8s 对象



   kubectl create secret tls tomcat-ingress-secret --cert=tls.crt --key=tls.key

-  创建带证书的 ingress 对象，它能将 ingress-tomcat 与 service
   关联，从而在 service 后主机发生变动的时候，反应在 ingress-tomcat
   这个容器的配置文件中



   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: ingress-deploy-tomcat-tls
     namespace: default
     annotations:
       kubernetes.io/ingress.class: "nginx"
   spec:
     tls:
       - hosts:
         - tomcat.kaliarch.com
         secretName: tomcat-ingress-secret
     rules:
       - host: tomcat.kaliarch.com
         http:
           paths:
             - path:
               backend:
                 serviceName: service-ingress-tomcat
                 servicePort: 8080

-  查看 ingress-nginx 对外暴露的端口，这里为30080，和 30443 两个



   kubectl get service -n ingress-nginx

-  使用 nodeip + ingress-nginx 暴露端口访问，由于上面创建的 ingress
   为基于主机名称的，所以需要在访问时在 /etc/hosts 做好映射到 node。



   https://tomcat.kaliarch.com:30443
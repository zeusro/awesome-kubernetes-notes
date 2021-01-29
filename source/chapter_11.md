
# 十一 配置信息容器化


k8s 提供了 configMap、secret 这两种特殊类型的存储卷，多数情况下不是为
POD 提供存储空间，而是为用户提供了从集群外部到 POD
内部注入配置信息的方式。

-  配置信息容器化有哪些方式

1. 自定义命令行参数，例如：command、args，根据 args
   传递不同的参数来将容器运行为不同的特性
2. 直接把配置信息制作为 image
   中，但是这种方式非常不灵活，这个镜像只能适用于一种使用场景，过度耦合
3. 环境变量，Cloud Native 支持通过环境变量来加载配置，或者使用
   ENTRYPOINT 脚本来预处理环境变量为配置信息
4. 存储卷，在容器启动时候挂载一个存储卷，或者专用的配置存储卷，挂载到应用程序的配置文件目录

-  Secret与ConfigMap对比


   相同点：
   -   key / value 的形式 
   -   属于某个特定的 namespace 
   -   可以导出到环境变量 
   -   可以通过目录/文件形式挂载(支持挂载所有key和部分key)

   不同点：
   -   Secret 可以被 ServerAccount 关联(使用) 
   -   Secret 可以存储 register 的鉴权信息，用在 ImagePullSecret 参数中，用于拉取私有仓库的镜像 
   -   Secret 支持 Base64 加密 
   -   Secret 分为 kubernetes.io/Service Account，kubernetes.io/dockerconfigjson，Opaque三种类型, Configmap 不区分类型 
   -   Secret 文件存储在tmpfs文件系统中，Pod 删除后 Secret文件也会对应的删除。

11.1 POD 获取环境变量
---------------------

-  env，详见：kubectl explain pods.spec.containers.env

```
   name              <string>  # 变量名称
   value             <string>  # 变量的值
   valueFrom         <Object>  # 引用值，如：configMap 的某个键、POD 定义中的字段名，如：metadata.labels
   resourceFieldRef  <Object>  # 引用资源限制中的值
   secretKeyRef      <Object>  # 引用 secretKey
```

11.2 configMap
--------------

假如我们现在要启动一个 POD ，这个 POD
启动时候，需要读取不同的配置信息，那么我们有两种方式：

1. 可以将 configMap 资源关联到当前 POD 上，POD 从 configMap
   读取一个数据，传递给 POD
   内部容器的一个变量，变量被注入后，可以重启容器。
2. 可以将 configMap 资源挂载到当前 POD
   上，作为一个文件系统的路径，这个目录正好是应用程序读取配置文件的路径，容器就可以读取到配置信息了，当
   configMap 修改了，那么就会通知 POD ，POD 可以进行重载配置。

在每个 configMap 中所有的配置信息都保存为键值的配置形式。

-  清单格式，详见：kubectl explain configMap



   apiVersion  <string>              # 版本号
   binaryData  <map[string]string>   # 二进制的数据
   data        <map[string]string>   # 键值对的数据
   kind        <string>              # 对象类型
   metadata    <Object>              # 对象元数据

-  命令行方式创建


```
   # 创建名为 my-config 的 configMap，它的数据来自目录中的文件，键为文件名，值为文件内容
   kubectl create configmap my-config --from-file=path/to/dir

   # 创建名为 my-config 的 configMap，它的数据来自文件中的键值对
   kubectl create configmap my-config --from-file=path/to/file

   # 创建名为 my-config 的 configMap，也可以手动指定键的名称
   kubectl create configmap my-config --from-file=key1=/path/to/bar/file1.txt --from-file=key2=/path/to/bar/file2.txt

   # 从字面量中创建
   kubectl create configmap my-config --from-literal=key1=config1 --from-literal=key2=config2

   # 从env文件中命名 my-config
   kubectl create configmap my-config --from-env-file=path/to/bar.env
```

### 11.2.1 注入 POD ENV

-  创建 ConfigMap 并在 POD ENV 中使用

```
   apiVersion: v1
   kind: ConfigMap                                        # 创建 ConfigMap 对象
   metadata:
     name: nginx-config
     namespace: default
   data:
     server_name: myapp.kaliarch.com                       # 键值对数据
     nginx_port: |                                        # 键值对数据，此处为 nginx 配置文件，需要注意换行的写法
       server {
           server_name  myapp.kaliarch.com;
           listen  80;
           root  /data/web/html;
       }

   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-configmap-demo
     namespace: default
     labels:
       app: myapp
       tier: frontend
     annotations:
       kaliarch.com/created-by: "cluster amdin"
   spec:
     containers:
       - name: myapp
         image: ikubernetes/myapp:v1
         ports:
           - name: http
             containerPort: 80
         env:
           - name: NGINX_SERVER_PORT          # 定义容器内变量的名字，容器需要在启动的时候使用 ENTRYPOINT 脚本将环境变量转换为应用的配置文件
             valueFrom:                       # 值来自于 configMap 对象中
               configMapKeyRef:               # 引用 configMap 对象
                 name: nginx-config           # configMap 对象的名字
                 key: nginx_port              # 引用 configMap 中的哪个 key
                 optional: true               # 相对 POD 启动是否为可选，如果 configMap 中不存在这个值，true 则不阻塞 POD 启动
           - name: NGINX_SERVER_NAME          # 定义容器内变量的名字，使用 exec 进入容器会发现变量已经在启动容器前注入容器内部了。
             valueFrom:
               configMapKeyRef:
                 name: nginx-config
                 key: server_name
```

### 11.2.2 挂载为 POD 卷

-  configMap 中的数据可以在容器内挂载为文件，并且当 configMap
   中的数据发生变动的时候，容器内的文件相应也会发生变动，但不会重载容器内的进程。

```
   apiVersion: v1
   kind: ConfigMap                                     # 创建 ConfigMap
   metadata:
     name: nginx-config-volumes
     namespace: default
   data:                                               # ConfigMap 中保存了两个数据，
     index: |                                          # 数据1，它可以在 container 中使用 ENV 注入环境变量，也可以在 container 中使用 volumeMounts 挂载成为文件
       <h1>this is a test page<h1>
     vhost: |                                          # 数据2，它可以在 container 中使用 ENV 注入环境变量，也可以在 container 中使用 volumeMounts 挂载成为文件
       server {                                                                                                                                  
           listen       80;                                                                                                                      
           server_name  localhost;                                                                                                               
                                                                                                                                                 
           location / {                                                                                                                          
               root   /usr/share/nginx/html;                                                                                                     
               index  index.html index.htm;                                                                                                      
           }                                                                                                                                     
                                                                                                                                                 
           error_page   500 502 503 504  /50x.html;                                                                                              
           location = /50x.html {                                                                                                                
               root   /usr/share/nginx/html;                                                                                                     
           }                                                                                                                                     
                                                                                                                                                 
           location = /hostname.html {                                                                                                           
               alias /etc/hostname;                                                                                                              
           }                                                                                                                                     
       } 
       server {
           server_name  myapp.kaliarch.com;
           listen  80;
           root  /data/web/html;
       }

   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-configmap-volumes-demo
     namespace: default
     labels:
       app: myapp
       tier: frontend
     annotations:
       kaliarch.com/created-by: "cluster amdin"
   spec:
     containers:
       - name: myapp
         image: ikubernetes/myapp:v1
         ports:
           - name: http
             containerPort: 80
         volumeMounts:
           - name: nginx-conf
             mountPath: /etc/nginx/conf.d
             readOnly: true
           - name: nginx-page
             mountPath: /data/web/html/
             readOnly: true
     volumes:                                               # 定义卷
       - name: nginx-conf                                   # 定义卷的名字
         configMap:                                         # 该卷的类型为 configMap
           name: nginx-config-volumes                       # 从命名空间中读取哪个名字的 configMap
           items:                                           # 定义 configMap 数据到文件的映射，如果不定义则使用 configMap 中的键为文件名称，值为文件内容
             - key: vhost                                   # 使用 configMap 哪个键
               path: www.conf                               # 将 configMap 中的数据，映射为容器内哪个文件名称
               mode: 0644                                    # 指明文件的权限
       - name: nginx-page
         configMap:
           name: nginx-config-volumes
           items:
             - key: index
               path: index.html
               mode: 0644
```

-  启动后进入容器查看文件是否正常挂载



   kubectl exec -it pod-configmap-volumes-demo -c myapp -- /bin/sh

-  使用 curl 命令验证，是否能够正常使用



   $ curl 10.244.2.104
   Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>

   $ curl -H "Host:myapp.kaliarch.com" 10.244.2.104
   <h1>this is a test page<h1>

## 11.3 secret


configMap 是明文存储数据的，如果需要存储敏感数据，则需要使用 secret
，secret 与 configMap 的作用基本一致，且 secret
中的数据不是明文存放的，而是 base64 编码保存的。

-  secret 类型



   docker-registry    # 创建一个 Docker registry 使用的 secret
   generic            # 从本地文件，目录或字面值创建一个 secret
   tls                # 创建一个 TLS  secret

-  清单格式，详见：kubectl explain secret



   apiVersion  <string>               # API 版本
   data        <map[string]string>    # 以键值对列出数据，值需要经过 base64 加密
   kind        <string>               # 对象类型
   metadata    <Object>               # 元数据
   stringData  <map[string]string>    # 明文的数据
   type        <string>               # 数据类型

### 11.3.1 私有仓库认证1


-  首先通过命令行创建出来 secret



   kubectl create secret docker-registry regsecret --docker-server=registry-vpc.cn-hangzhou.aliyuncs.com --docker-username=admin --docker-password=123456 --docker-email=420123641@qq.com

-  如果想保存为文件可以



   kubectl get secret regsecret -o yaml

-  POD 创建时候，从 docker hub 拉取镜像使用的用户名密码，kubectl explain
   pods.spec 的 imagePullSecrets 字段

```
   apiVersion: v1
   kind: Pod
   metadata:
     name: secret-file-pod
   spec:
     containers:
     - name: mypod
       image: redis
     imagePullSecrets:                         # 获取镜像需要的用户名密码
      - name: regsecret                        # secret 对象
```

### 11.3.2 私有仓库认证2

-  首先通过命令行创建出来 secret

   kubectl create secret docker-registry regsecret --docker-server=registry-vpc.cn-hangzhou.aliyuncs.com --docker-username=admin --docker-password=123456 --docker-email=420123641@qq.com

-  创建自定义的 serviceaccount 对象，在 serviceaccount 对象上定义 image
   pull secrets

.. code:: yaml

   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: admin
     namespace: default
   imagePullSecrets:
   - name: regsecret                       # 指定 secret

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
     serviceAccountName: admin                          # 使用 serviceaccount 进行拉取镜像的认证，这样更加安全

### 11.3.3 创建 TLS 证书

-  首先通过命令行创建出来



   kubectl create secret tls nginx-secret --cert=tls.crt --key=tls.key

-  secret 中的数据可以在容器内挂载为文件，然后在 nginx
   容器内使用证书文件

```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-configmap-volumes-demo
     namespace: default
     labels:
       app: myapp
       tier: frontend
     annotations:
       kaliarch.com/created-by: "cluster amdin"
   spec:
     containers:
       - name: myapp
         image: ikubernetes/myapp:v1
         ports:
           - name: http
             containerPort: 80
         volumeMounts:
           - name: nginx-conf
             mountPath: /etc/nginx/secret
             readOnly: true
     volumes:                                               # 定义卷
       - name: nginx-conf                                   # 定义卷的名字
         configMap:                                         # 该卷的类型为 secret
           name: nginx-secret                               # 从命名空间中读取哪个名字的 secret
           items:                                           # 定义 secret 数据到文件的映射，如果不定义则使用 secret 中的键为文件名称，值为文件内容
             - key: tls.key                                 # 使用 secret 哪个键
               path: www.conf                               # 将 secret 中的数据，映射为容器内哪个文件名称
               mode: 0644                                    # 指明文件的权限
             - key: tls.crt
               path: index.html
               mode: 0644
```
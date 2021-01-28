# 二十五 存储（volume）

严格意义上，env并不算上是 存储的一部分，不过作为可以被声明并使用的资源，env跟其他volume有相似之处，所以归结到这里。

## 25.1 env

###  env

```
          env:
            - name: GIT_REPO
              value: 'ssh://git@127.0.0.1:22/a/b.git'
```

### 嵌套env

```
          env:
            - name: spring.profiles.active
              value: 'product'
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP              
            - name: GOMS_API_HTTP_ADDR
              value: 'http://$(MY_POD_IP):9090'
```

### configMap

**注意一下,修改configmap不会导致容器里的挂载的configmap文件/环境变量发生改变;删除configmap也不会影响到容器内部的环境变量/文件,但是删除configmap之后,被挂载的pod上面会出现一个warnning的事件**

```
Events:
  Type     Reason       Age                 From                                         Message
  ----     ------       ----                ----                                         -------
  Warning  FailedMount  64s (x13 over 11m)  kubelet, cn-shenzhen.i-wz9498k1n1l7sx8bkc50  MountVolume.SetUp failed for volume "nginx" : configmaps "nginx" not found
```

[config map](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-container-environment-variables-using-configmap-data)写的很清楚了,这里恬不知耻得copy一下

**注意,configmap有1M的限制,一般用来挂载小型配置,大量配置建议上配置中心**

### 挂载单一项
```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        # Define the environment variable
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              # The ConfigMap containing the value you want to assign to SPECIAL_LEVEL_KEY
              name: special-config
              # Specify the key associated with the value
              key: special.how
  restartPolicy: Never
```

表示挂载`special-config`这个configmap的`special.how`项


### 挂载整个configmap

```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - configMapRef:
          name: special-config
  restartPolicy: Never
```

参考:

1. [Add nginx.conf to Kubernetes cluster](https://stackoverflow.com/questions/42078080/add-nginx-conf-to-kubernetes-cluster)
2. [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#define-container-environment-variables-using-configmap-data)

###  fieldRef

可以挂载pod的一些属性

```
          env:
          - name: MY_POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP

```

Selects a field of the pod: supports metadata.name, metadata.namespace, metadata.labels, metadata.annotations, spec.nodeName, spec.serviceAccountName, status.hostIP, status.podIP.


### resourceFieldRef

Selects a resource of the container: only resources limits and requests (limits.cpu, limits.memory, limits.ephemeral-storage, requests.cpu, requests.memory and requests.ephemeral-storage) are currently supported.

英文介绍得很明白,用来挂载当前yaml里面container的资源(CPU/内存)限制,用得比较少啦其实.此外还可以结合`downloadAPI`

注意`containerName`不能配错,不然pod状态会变成`CreateContainerConfigError`

```
          env:  
            - name: a
              valueFrom: 
                 resourceFieldRef:
                      containerName: nginx-test2
                      resource: limits.cpu
```



### secretKeyRef

Selects a key of a secret in the pod's namespace

```
        env:
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: username
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
```

参考:
1. [Kubernetes中Secret使用详解](https://blog.csdn.net/yan234280533/article/details/77018640)
2. https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#envvarsource-v1-core


## 25.2 目录/文件类挂载

k8s可以挂载的资源实在是太多,这里挑一些比较有代表性的来讲一下

这一类资源一般要先在spec层级定义`volumes`,然后在`containers`定义`volumeMounts`,有种先声明,再使用的意思

### hostPath(宿主机目录/文件)

1. 既有目录/文件用`Directory`/`File`+nodeSelector
  但是用了`nodeSelector`之后,以后的伸缩都会在匹配的节点上,如果节点只有1个,副本集设置得超出实际节点可承受空间,最终将导致单点问题,这个要注意下
1. 应用启用时读写空文件用`DirectoryOrCreate`或者`FileOrCreate`

以下演示第一种方案


    #给节点打上标签(这里省略)
    kubectl get node --show-labels

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:  
  labels:
    app: nginx-test2
  name: nginx-test2
  namespace: test
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: nginx-test2
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx-test2
    spec:
      containers:
        - image: 'nginx:1.15.4-alpine'
          imagePullPolicy: Always
          name: nginx-test2
          resources: {}
          terminationMessagePolicy: File
          volumeMounts:
            - name: host1
              mountPath: /etc/nginx/sites-enabled
            - name: host2
              mountPath: /etc/nginx/sites-enabled2/a.com.conf            
      nodeSelector: 
        kubernetes.io/hostname: cn-shenzhen.i-wz9aabuytimkomdmjabq        
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
        - name: host1
          hostPath:
            path: /root/site
            type: Directory
        - name: host2
          hostPath:
            path: /root/site/a.com.conf
            type: File            
```

### configMap


#### 单项挂载(第1种)

这种挂载会热更新,更改后大约10秒后能看到变化

```
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level
```

#### 单项挂载(第2种)

这种挂载方式不会热更新

```
          volumeMounts:                  
            - name: nginx
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf                            
      volumes:             
          - name: nginx
            configMap:
              name: amiba-nginx 
```

#### 完全挂载

这种挂载会热更新,更改后大约10秒后能看到变化

```
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
```

### secret

#### 单项挂载

```
  volumes:
  - name: secrets
    secret:
      secretName: mysecret
      items:
      - key: password
        mode: 511
        path: tst/psd
      - key: username
        mode: 511
        path: tst/usr
```


#### 完全挂载

这里用了特定权限去挂载文件,默认好像是777

```
          volumeMounts:
            - name: sshkey
              mountPath: /root/.ssh              
      volumes:
        - name: sshkey
          secret:           
           secretName: pull-gitea
           defaultMode: 0400    
```

```
 kubectl create secret generic pull-gitea  \
--from-file=id_rsa=/Volumes/D/temp/id_rsa  \
--from-file=id_rsa.pub=/Volumes/D/temp/id_rsa.pub  \
--from-file=known_hosts=/Volumes/D/temp/known_hosts \
```
比如这个模式创建出来的secret,容器里面/root/.ssh目录就会有`id_rsa`,`id_rsa.pub`,`known_hosts`3个文件

### downwardAPI


参考链接:
1. [volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
1. [kubernetes-api/v1.12](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#hostpathvolumesource-v1-core)
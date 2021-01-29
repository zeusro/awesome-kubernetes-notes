
二十四 国产容器管理平台KubeSphere实战排错
=========================================

   概述：近期在使用QingCloud的Kubesphere，极好的用户体验，私有化部署，无基础设施依赖，无
   Kubernetes
   依赖，支持跨物理机、虚拟机、云平台部署，可以纳管不同版本、不同厂商的
   Kubernetes
   集群。在k8s上层进行了封装实现了基于角色的权限控制，DevOPS流水线快速实现CI/CD，内置harbor/gitlab/jenkins/sonarqube等常用工具，基于基于
   OpenPitrix
   提供应用的全生命周期管理，包含开发、测试、发布、升级，下架等应用相关操作自己体验还是非常的棒。
   同样作为开源项目，难免存在一些bug，在自己的使用中遇到下排错思路，非常感谢qingcloud社区提供的技术协助，对k8s有兴趣的可以去体验下国产的平台，如丝般顺滑的体验，rancher的用户也可以来对不体验下。

24.1 清理退出状态的容器
-----------------------

在集群运行一段时间后，有些container由于异常状态退出Exited，需要去及时清理释放磁盘，可以将其设置成定时任务执行



   docker rm `docker ps -a | grep Exited |awk '{print $1}'`

24.2 清理异常或被驱逐的 pod
---------------------------

-  清理kubesphere-devops-system的ns下清理



   kubectl delete pods -n kubesphere-devops-system $(kubectl get pods -n kubesphere-devops-system | grep Evicted |awk '{print $1}')
   kubectl delete pods -n kubesphere-devops-system $(kubectl get pods -n kubesphere-devops-system | grep CrashLoopBackOff |awk '{print $1}')

-  为方便清理指定ns清理evicted/crashloopbackoff的pod/清理exited的容器


```
   #!/bin/bash
   clear_evicted_pod() {
     ns=$1
     kubectl delete pods -n ${ns} $(kubectl get pods -n ${ns} | grep Evicted |awk '{print $1}')
   }
   clear_crash_pod() {
     ns=$1
     kubectl delete pods -n ${ns} $(kubectl get pods -n ${ns} | grep CrashLoopBackOff |awk '{print $1}')
   }
   clear_exited_container() {
     docker rm `docker ps -a | grep Exited |awk '{print $1}'`
   }


   echo "1.clear exicted pod"
   echo "2.clear crash pod"
   echo "3.clear exited container"
   read -p "Please input num:" num


   case ${num} in 
   "1")
     read -p "Please input oper namespace:" ns
     clear_evicted_pod ${ns}
     ;;


   "2")
     read -p "Please input oper namespace:" ns
     clear_crash_pod ${ns}
     ;;
   "3")
     clear_exited_container
     ;;
   "*")
     echo "input error"
     ;;
   esac
```

-  清理全部ns中evicted/crashloopbackoff的pod


```
   # 获取所有ns
   kubectl get ns | grep -v "NAME" |awk '{print $1}'

   # 清理驱逐状态的pod
   for ns in `kubectl get ns | grep -v "NAME" | awk '{print $1}'`;do kubectl delete pods -n ${ns} $(kubectl get pods -n ${ns} | grep "Evicted" |awk '{print $1}');done
   # 清理异常pod
   for ns in `kubectl get ns | grep -v "NAME" | awk '{print $1}'`;do kubectl delete pods -n ${ns} $(kubectl get pods -n ${ns} | grep "CrashLoopBackOff" |awk '{print $1}');done
```

24.3 Docker 数据迁移
--------------------

在安装过程中未指定docker数据目录，系统盘50G，随着时间推移磁盘不够用，需要迁移docker数据，使用软连接方式：
首选挂载新磁盘到/data目录



   systemctl stop docker

   mkdir -p /data/docker/  

   rsync -avz /var/lib/docker/ /data/docker/  

   mv /var/lib/docker /data/docker_bak

   ln -s /data/docker /var/lib/

   systemctl daemon-reload

   systemctl start docker

24.4 kubesphere 网络排错
------------------------

-  问题描述：

在kubesphere的node节点或master节点，手动去启动容器，在容器里面无法连通公网，是我的配置哪里不对么，之前默认使用calico，现在改成fluannel也不行，在kubesphere中部署deployment中的pod的容器上可以出公网，在node或master单独手动启动的访问不了公网

查看手动启动的容器网络上走的docker0

```
   root@fd1b8101475d:/# ip a

   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1

       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

       inet 127.0.0.1/8 scope host lo

          valid_lft forever preferred_lft forever

   2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1

       link/ipip 0.0.0.0 brd 0.0.0.0

   105: eth0@if106: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 

       link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0

       inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0

          valid_lft forever preferred_lft forever
```

在pods中的容器网络用的是kube-ipvs0

```
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1

       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

       inet 127.0.0.1/8 scope host lo

          valid_lft forever preferred_lft forever

   2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1

       link/ipip 0.0.0.0 brd 0.0.0.0

   4: eth0@if18: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue

       link/ether c2:27:44:13:df:5d brd ff:ff:ff:ff:ff:ff

       inet 10.233.97.175/32 scope global eth0

          valid_lft forever preferred_lft forever
```

-  解决方案：

查看docker启动配置

![image](images/kubesphere/1.jpeg)

修改文件/etc/systemd/system/docker.service.d/docker-options.conf中去掉参数：–iptables=false
这个参数等于false时会不写iptables



   [Service]
   Environment="DOCKER_OPTS=  --registry-mirror=https://registry.docker-cn.com --data-root=/var/lib/docker --log-opt max-size=10m --log-opt max-file=3 --insecure-registry=harbor.devops.kubesphere.local:30280"

24.5 kubesphere 应用路由异常
----------------------------

在kubesphere中应用路由ingress使用的是nginx，在web界面配置会导致两个host使用同一个ca证书，可以通过注释文件配置

⚠️注意：ingress控制deployment在：

![image](images/kubesphere/2.jpeg)



```
   kind: Ingress
   apiVersion: extensions/v1beta1
   metadata:
     name: prod-app-ingress
     namespace: prod-net-route
     resourceVersion: '8631859'
     labels:
       app: prod-app-ingress
     annotations:
       desc: 生产环境应用路由
       nginx.ingress.kubernetes.io/client-body-buffer-size: 1024m
       nginx.ingress.kubernetes.io/proxy-body-size: 2048m
       nginx.ingress.kubernetes.io/proxy-read-timeout: '3600'
       nginx.ingress.kubernetes.io/proxy-send-timeout: '1800'
       nginx.ingress.kubernetes.io/service-upstream: 'true'
   spec:
     tls:
       - hosts:
           - smartms.tools.anchnet.com
         secretName: smartms-ca
       - hosts:
           - smartsds.tools.anchnet.com
         secretName: smartsds-ca
     rules:
       - host: smartms.tools.anchnet.com
         http:
           paths:
             - path: /
               backend:
                 serviceName: smartms-frontend-svc
                 servicePort: 80
       - host: smartsds.tools.anchnet.com
         http:
           paths:
             - path: /
               backend:
                 serviceName: smartsds-frontend-svc

                 servicePort: 80
```

24.6 Jenkins 的 Agent
---------------------

用户在自己的使用场景当中，可能会使用不同的语言版本活不同的工具版本。这篇文档主要介绍如何替换内置的
agent。

默认base-build镜像中没有sonar-scanner工具，Kubesphere Jenkins 的每一个
agent 都是一个Pod，如果要替换内置的agent，就需要替换 agent 的相应镜像。

构建最新 kubesphere/builder-base:advanced-1.0.0 版本的 agent 镜像

更新为指定的自定义镜像：ccr.ccs.tencentyun.com/testns/base:v1

参考链接：\ https://kubesphere.io/docs/advanced-v2.0/zh-CN/devops/devops-admin-faq/#%E5%8D%87%E7%BA%A7-jenkins-agent-%E7%9A%84%E5%8C%85%E7%89%88%E6%9C%AC

![image](images/kubesphere/3.jpeg)

![image](images/kubesphere/4.jpeg)

在 KubeSphere 修改 jenkins-casc-config 以后，您需要在 Jenkins Dashboard
系统管理下的 configuration-as-code 页面重新加载您更新过的系统配置。

参考：

https://kubesphere.io/docs/advanced-v2.0/zh-CN/devops/jenkins-setting/#%E7%99%BB%E9%99%86-jenkins-%E9%87%8D%E6%96%B0%E5%8A%A0%E8%BD%BD

![image](images/kubesphere/5.jpeg)


jenkins中更新base镜像

![image](images/kubesphere/6.jpeg)


⚠️先修改kubesphere中jenkins的配置，\ `jenkins-casc-config <http://xxxxxxxxx:30800/system-workspace/projects/kubesphere-devops-system/configmaps/jenkins-casc-config>`__

24.7 Devops 中 Mail的发送
-------------------------

参考：\ https://www.cloudbees.com/blog/mail-step-jenkins-workflow

内置变量：

+-----------------------------------+-----------------------------------+
| **变量名**                        | **解释**                          |
+===================================+===================================+
| BUILD_NUMBER                      | The current build number, such as |
|                                   | “153”                             |
+-----------------------------------+-----------------------------------+
| BUILD_ID                          | The current build ID, identical   |
|                                   | to BUILD_NUMBER for builds        |
|                                   | created in 1.597+, but a          |
|                                   | YYYY-MM-DD_hh-mm-ss timestamp for |
|                                   | older builds                      |
+-----------------------------------+-----------------------------------+
| BUILD_DISPLAY_NAME                | The display name of the current   |
|                                   | build, which is something like    |
|                                   | “#153” by default.                |
+-----------------------------------+-----------------------------------+
| JOB_NAME                          | Name of the project of this       |
|                                   | build, such as “foo” or           |
|                                   | “foo/bar”. (To strip off folder   |
|                                   | paths from a Bourne shell script, |
|                                   | try:                              |
|                                   | :math:`{JOB_NAME}) | | BUILD_TAG  |
|                                   | | String of "jenkins-`\ {JOB_NAME |
|                                   | }-${BUILD_NUMBER}".               |
|                                   | Convenient to put into a resource |
|                                   | file, a jar file, etc for easier  |
|                                   | identification.                   |
+-----------------------------------+-----------------------------------+
| EXECUTOR_NUMBER                   | The unique number that identifies |
|                                   | the current executor (among       |
|                                   | executors of the same machine)    |
|                                   | that’s carrying out this build.   |
|                                   | This is the number you see in the |
|                                   | “build executor status”, except   |
|                                   | that the number starts from 0,    |
|                                   | not 1.                            |
+-----------------------------------+-----------------------------------+
| NODE_NAME                         | Name of the slave if the build is |
|                                   | on a slave, or “master” if run on |
|                                   | master                            |
+-----------------------------------+-----------------------------------+
| NODE_LABELS                       | Whitespace-separated list of      |
|                                   | labels that the node is assigned. |
+-----------------------------------+-----------------------------------+
| WORKSPACE                         | The absolute path of the          |
|                                   | directory assigned to the build   |
|                                   | as a workspace.                   |
+-----------------------------------+-----------------------------------+
| JENKINS_HOME                      | The absolute path of the          |
|                                   | directory assigned on the master  |
|                                   | node for Jenkins to store data.   |
+-----------------------------------+-----------------------------------+
| JENKINS_URL                       | Full URL of Jenkins, like         |
|                                   | `http://server:port/jenkins/ <htt |
|                                   | p://server:port/jenkins/>`__      |
|                                   | (note: only available if Jenkins  |
|                                   | URL set in system configuration)  |
+-----------------------------------+-----------------------------------+
| BUILD_URL                         | Full URL of this build, like      |
|                                   | `http://server:port/jenkins/job/f |
|                                   | oo/15/ <http://server:port/jenkin |
|                                   | s/job/foo/15/>`__                 |
|                                   | (Jenkins URL must be set)         |
+-----------------------------------+-----------------------------------+
| SVN_REVISION                      | Subversion revision number that’s |
|                                   | currently checked out to the      |
|                                   | workspace, such as “12345”        |
+-----------------------------------+-----------------------------------+
| SVN_URL                           | Subversion URL that’s currently   |
|                                   | checked out to the workspace.     |
+-----------------------------------+-----------------------------------+
| JOB_URL                           | Full URL of this job, like        |
|                                   | `http://server:port/jenkins/job/f |
|                                   | oo/ <http://server:port/jenkins/j |
|                                   | ob/foo/>`__                       |
|                                   | (Jenkins URL must be set)         |
+-----------------------------------+-----------------------------------+

最终自己写了适应自己业务的模版，可以直接使用



   mail to: 'xuel@net.com',
             charset:'UTF-8', // or GBK/GB18030
             mimeType:'text/plain', // or text/html
             subject: "Kubesphere ${env.JOB_NAME} [${env.BUILD_NUMBER}] 发布正常Running Pipeline: ${currentBuild.fullDisplayName}",
             body: """
             ---------Anchnet Devops Kubesphere Pipeline job--------------------


             项目名称 : ${env.JOB_NAME}
             构建次数 : ${env.BUILD_NUMBER}
             扫描信息 : 地址:${SONAR_HOST}
             镜像地址 : ${REGISTRY}/${QHUB_NAMESPACE}/${APP_NAME}:${IMAGE_TAG}
             构建详情：SUCCESSFUL: Job ${env.JOB_NAME} [${env.BUILD_NUMBER}]
             构建状态 : ${env.JOB_NAME} jenkins 发布运行正常
             构建URL : ${env.BUILD_URL}"""

![image](images/kubesphere/7.jpeg)

![image](images/kubesphere/8.jpeg)
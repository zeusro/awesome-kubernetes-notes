三 集群部署与维护
===========

为简单上手体验功能，可以先利用kubeadm安装测试，生产环境建议二进制或者一些成熟的集群高可用安装方式，Kubeadm
是 K8S 官方提供的快速部署工具，它提供了 kubeadm init 以及 kubeadm join
这两个命令作为快速创建 kubernetes 集群的最佳实践，本章节说明了使用
kubeadm 来部署 K8S 集群的过程。

-   集群组织结构

> 项目说明 -----------------------------------------------------
> 集群规模
> Master、node1、node2 系统 CentOS 7.3 网络规划
> POD：10.244.0.0/16、Service：10.96.0.0/12

3.1 部署前准备
--------------

> 本小节的所有的操作，在所有的节点上进行

### 3.1.1 关闭 firewalld 和 selinux

``` bash
setenforce 0
sed -i '/^SELINUX=/cSELINUX=disabled' /etc/selinux/config

systemctl stop firewalld
systemctl disable firewalld
```

### 3.1.2 加载 ipvs 内核模块

-   安装 IPVS 模块

``` bash
yum -y install ipvsadm ipset sysstat conntrack libseccomp
```

-   设置开机加载配置文件

``` bash
cat >>/etc/modules-load.d/ipvs.conf<<EOF
ip_vs_dh
ip_vs_ftp
ip_vs
ip_vs_lblc
ip_vs_lblcr
ip_vs_lc
ip_vs_nq
ip_vs_pe_sip
ip_vs_rr
ip_vs_sed
ip_vs_sh
ip_vs_wlc
ip_vs_wrr
nf_conntrack_ipv4
EOF
```

-   设置开机加载 IPVS 模块

``` bash
systemctl enable systemd-modules-load.service   # 设置开机加载内核模块
lsmod | grep -e ip_vs -e nf_conntrack_ipv4      # 重启后检查 ipvs 模块是否加载
```

-   如果集群已经部署在了 iptables 模式下，可以通过下面命令修改，修改
    mode 为 ipvs 重启集群即可。

``` bash
kubectl edit -n kube-system configmap kube-proxy
```

### 3.1.3 下载 Docker 和 K8S

-   设置 docker 源

``` bash
curl -o /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

-   设置 k8s 源

``` bash
cat >>/etc/yum.repos.d/kuberetes.repo<<EOF
[kuberneres]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
enabled=1
EOF
```

-   安装 docker-ce 和 kubernetes

``` bash
yum install docker-ce kubelet kubectl kubeadm -y
```

``` bash
systemctl start docker
systemctl enable docker
systemctl enable kubelet
```

### 3.1.4 设置内核及 K8S 参数

-   设置内核参数

``` bash
cat >>/etc/sysctl.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

-   设置 kubelet 忽略 swap，使用 ipvs

``` bash
cat >/etc/sysconfig/kubelet<<EOF
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
KUBE_PROXY_MODE=ipvs
EOF
```

3.2 部署 Master
---------------

> 本小节的所有的操作，只在 Master 节点上进行

### 3.2.1 提前拉取镜像

宿主机最好能访问国外资源，在kubeadm init 在初始化的时候会到谷歌的 docker
hub 拉取镜像，如果宿主机测试无法访问 k8s.gcr.io
可以在服务器所以我们要提前部署好代理软件，本例中监听个本机 9666
进行部署。

如果条件不允许可以参考:
<https://blog.csdn.net/jinguangliu/article/details/82792617>
来解决镜像问题。

-   配置 Docker 拉取镜像时候的代理地址，vim
    /usr/lib/systemd/system/docker.service。

``` bash
[Service]
Environment="HTTPS_PROXY=127.0.0.1:9666"
Environment="NO_PROXY=127.0.0.0/8,172.16.0.0/16"
```

-   提前拉取初始化需要的镜像

``` bash
kubeadm config images pull
```

-   使用其他源镜像

``` bash
docker pull mirrorgooglecontainers/kube-apiserver:v1.14.2
docker pull mirrorgooglecontainers/kube-controller-manager:v1.14.2
docker pull mirrorgooglecontainers/kube-scheduler:v1.14.2
docker pull mirrorgooglecontainers/kube-proxy:v1.14.2
docker pull mirrorgooglecontainers/pause:3.1
docker pull mirrorgooglecontainers/etcd:3.3.10
docker pull coredns/coredns:1.3.1


利用`kubeadm config images list` 查看需要的docker image name

k8s.gcr.io/kube-apiserver:v1.14.2
k8s.gcr.io/kube-controller-manager:v1.14.2
k8s.gcr.io/kube-scheduler:v1.14.2
k8s.gcr.io/kube-proxy:v1.14.2
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1

# 修改tag

docker tag docker.io/mirrorgooglecontainers/kube-apiserver:v1.14.2 k8s.gcr.io/kube-apiserver:v1.14.2
docker tag docker.io/mirrorgooglecontainers/kube-scheduler:v1.14.2 k8s.gcr.io/kube-scheduler:v1.14.2
docker tag docker.io/mirrorgooglecontainers/kube-proxy:v1.14.2 k8s.gcr.io/kube-proxy:v1.14.2
docker tag docker.io/mirrorgooglecontainers/kube-controller-manager:v1.14.2 k8s.gcr.io/kube-controller-manager:v1.14.2
docker tag docker.io/mirrorgooglecontainers/etcd:3.3.10  k8s.gcr.io/etcd:3.3.10
docker tag docker.io/mirrorgooglecontainers/pause:3.1  k8s.gcr.io/pause:3.1
docker tag docker.io/coredns/coredns:1.3.1  k8s.gcr.io/coredns:1.3.1

docker rmi `docker images |grep docker.io/ |awk '{print $1":"$2}'`
```

### 3.2.2 初始化Master

-   使用 kubeadm 初始化 k8s 集群

``` bash
kubeadm init --kubernetes-version=v1.14.0 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap
```

-   如果有报错使用下面命令查看

``` bash
journalctl -xeu kubelet
```

-   如果初始化过程被中断可以使用下面命令来恢复

``` bash
kubeadm reset
```

-   下面是最后执行成功显示的结果，需要保存这个执行结果，以让 node
    节点加入集群

``` bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.100.9:6443 --token 2dyd69.hrfsjkkxs4stim7n \
    --discovery-token-ca-cert-hash sha256:4e30c1f41aefb177b708a404ccb7e818e31647c7dbdd2d42f6c5c9894b6f41e7
```

-   最好以普通用户的身份运行下面的命令

``` bash
# 在当前用户家目录下创建.kube目录并配置访问集群的config 文件
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

-   部署 flannel 网络插件

``` bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

-   查看 kube-system 命名空间中运行的 pods

``` bash
kubectl get pods -n kube-system
```

-   查看 k8s 集群组件的状态

``` bash
kubectl get ComponentStatus
```

-   配置命令补全

``` bash
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

3.3 部署 Node
-------------

> 本小节的所有的操作，只在 Node 节点上进行。

### 3.3.1 加入集群

-   加入集群，注意在命令尾部加上 --ignore-preflight-errors=Swap ，以忽略
    k8s 对主机 swap 的检查（k8s为了性能所以要求进制 swap ）

``` bash
kubeadm join 172.16.100.9:6443 --token 2dyd69.hrfsjkkxs4stim7n \
    --discovery-token-ca-cert-hash sha256:4e30c1f41aefb177b708a404ccb7e818e31647c7dbdd2d42f6c5c9894b6f41e7 --ignore-preflight-errors=Swap
```

-   返回结果，表示加入集群成功

``` bash
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

### 3.3.2 查看进度

当 node 节点加入 K8S 集群中后，Master 会调度到 Node
节点上一些组件，用于处理集群事务，这些组件没有下载完成之前 Node
节点在集群中还是未就绪状态

-   在 node 执行下面命令，可以查看镜像的下载进度，下面是最终结果显示

``` bash
$ docker image ls
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy    v1.14.0             5cd54e388aba        6 weeks ago         82.1MB
quay.io/coreos/flannel   v0.11.0-amd64       ff281650a721        3 months ago        52.6MB
k8s.gcr.io/pause         3.1                 da86e6ba6ca1        16 months ago       742kB
```

-   可以在 Master 上使用下面命令来查看新加入的节点状态

``` bash
$ kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
master   Ready    master   3d21h   v1.14.1
node1    Ready    <none>   3d21h   v1.14.1
node2    Ready    <none>   3d21h   v1.14.1
```

-   查看集群状态

``` bash
[root@master ~]# kubectl cluster-info 
Kubernetes master is running at https://10.234.2.204:6443
KubeDNS is running at https://10.234.2.204:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://10.234.2.204:6443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[root@master ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
```

如果嫌网络pull镜像慢可以在一台上面将镜像打包发送至其他node节点

    拷贝到node节点
    for i in /tmp/*.tar; do scp -i $i root@172.16.0.15:/root/;done


    node节点还原
    for i in *.tar ;do docker load -i $i;done

-   查看 kube-system 这个 k8s
    命名空间中有哪些组件，分别运行在哪个节点，-o wide 是以详细方式显示。

```bash
$ kubectl get pods -n kube-system -o wide

NAME                                 READY   STATUS    RESTARTS   AGE     IP              NODE         NOMINATED NODE   READINESS GATES
coredns-fb8b8dccf-cp24r              1/1     Running   0          26m     10.244.0.2      i-xeahpl98   <none>           <none>
coredns-fb8b8dccf-ljswp              1/1     Running   0          26m     10.244.0.3      i-xeahpl98   <none>           <none>
etcd-i-xeahpl98                      1/1     Running   0          25m     172.16.100.9    i-xeahpl98   <none>           <none>
kube-apiserver-i-xeahpl98            1/1     Running   0          25m     172.16.100.9    i-xeahpl98   <none>           <none>
kube-controller-manager-i-xeahpl98   1/1     Running   0          25m     172.16.100.9    i-xeahpl98   <none>           <none>
kube-flannel-ds-amd64-crft8          1/1     Running   3          16m     172.16.100.6    i-me87b6gw   <none>           <none>
kube-flannel-ds-amd64-nckw4          1/1     Running   0          6m41s   172.16.100.10   i-qhcc2owe   <none>           <none>
kube-flannel-ds-amd64-zb7sg          1/1     Running   0          23m     172.16.100.9    i-xeahpl98   <none>           <none>
kube-proxy-7kjkf                     1/1     Running   0          6m41s   172.16.100.10   i-qhcc2owe   <none>           <none>
kube-proxy-c5xs2                     1/1     Running   2          16m     172.16.100.6    i-me87b6gw   <none>           <none>
kube-proxy-rdzq2                     1/1     Running   0          26m     172.16.100.9    i-xeahpl98   <none>           <none>
kube-scheduler-i-xeahpl98            1/1     Running   0          25m     172.16.100.9    i-xeahpl98   <none>           <none>
```

### 3.3.3 镜像下载太慢

node 节点需要翻墙下载镜像太慢，建议使用 docker 镜像的导入导出功能
先将master的三个镜像打包发送到node节点，load后再jion

-   导出

``` bash
docker image save -o /tmp/kube-proxy.tar k8s.gcr.io/kube-proxy
docker image save -o /tmp/flannel.tar quay.io/coreos/flannel
docker image save -o /tmp/pause.tar k8s.gcr.io/pause
```

-   导入

``` bash
docker image load -i /tmp/kube-proxy.tar
docker image load -i /tmp/pause.tar
docker image load -i /tmp/flannel.tar
```
## 3.4 利用Azure中国搭建Kubernetes 1.14.2集群


### 基础设施

centos 7.6 64位

内核版本:5.1.3-1.el7.elrepo.x86_64(手动升级,可免)

kubeadm

kubelet

node*3


### 初始准备

#### repo镜像

```bash
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```

#### 升级内核

```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
yum --enablerepo=elrepo-kernel install -y kernel-ml
# 改引导
# awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg
sed -i 's/GRUB_DEFAULT=saved/GRUB_DEFAULT=0/g' /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot
uname -sr
```

#### 系统设置

```bash
# 禁用交换区
swapoff -a
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
# 开启forward
# Docker从1.13版本开始调整了默认的防火墙规则
# 禁用了iptables filter表中FOWARD链
# 这样会引起Kubernetes集群中跨Node的Pod无法通信
iptables -P FORWARD ACCEPT
```

#### 启用IPVS


```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

# https://github.com/easzlab/kubeasz/issues/374
# 4.18内核将nf_conntrack_ipv4更名为nf_conntrack
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
ipvs_modules="ip_vs ip_vs_lc ip_vs_wlc ip_vs_rr ip_vs_wrr ip_vs_lblc ip_vs_lblcr ip_vs_dh ip_vs_sh ip_vs_fo ip_vs_nq ip_vs_sed ip_vs_ftp nf_conntrack"
for kernel_module in \${ipvs_modules}; do
    /sbin/modinfo -F filename \${kernel_module} > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        /sbin/modprobe \${kernel_module}
    fi
done
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules 
lsmod | grep ip_vs
```




####  装18.06.2的docker

按照[kubernetes源代码](https://github.com/kubernetes/kops/blob/master/nodeup/pkg/model/docker.go#L57-L485)安装特定docker版本

```bash
# http://mirror.azure.cn/docker-ce/linux/centos/7/x86_64/stable/
sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=http://mirror.azure.cn/docker-ce/linux/centos/7/x86_64/stable/
enabled=1 
gpgcheck=1
gpgkey=http://mirror.azure.cn/docker-ce/linux/centos/gpg
EOF

yum install -y docker-ce-18.06.2.ce-3.el7.x86_64

# 配置docker加速
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://vhc6pxhv.mirror.aliyuncs.com"]
}
EOF

sudo systemctl start docker
systemctl enable docker.service
```

#### 安装其他依赖


```bash
yum -y install yum nfs-utils wget nano yum-utils device-mapper-persistent-data lvm2 git docker-compose ipvsadm net-tools telnet
yum update -y
```



#### install-kubeadm

配置k8s的镜像

```bash
sudo tee /etc/yum.repos.d/kubernetes.repo <<-'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
yum install -y  kubeadm kubectl --disableexcludes=kubernetes

systemctl enable kubelet
# && systemctl start kubelet
```

### 安装集群

接下来要根据实际情况选择单master还是奇数台master了

kubeadm的默认配置文件"藏"在`kubeadm config print init-defaults`和`kubeadm config print join-defaults`中,这里要根据中国特色社会主义的实际情况进行修改.


```
kubeadm config print join-defaults --component-configs KubeProxyConfiguration //JoinConfiguration KubeProxyConfiguration
kubeadm config print join-defaults --component-configs KubeletConfiguration // JoinConfiguration KubeletConfiguration
```

一般来说`serviceSubnet`范围要比`podSubnet`小

`podSubnet: 10.66.0.0/16`注定了最多只能有65534个pod,serviceSubnet同理.

#### 高可用型(生产用)

高可用性的特点在于N个etcd,kube-apiserver,kube-scheduler,kube-controller-manager,以组件的冗余作为高可用的基础.


api-server以负载均衡作为对外的入口.

[设置master](https://kubernetes.io/docs/setup/independent/setup-ha-etcd-with-kubeadm/)

```
# https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta1#ClusterConfiguration



export HOST0=172.18.221.35
export HOST1=172.18.243.72
export HOST2=172.18.243.77
mkdir -p ${HOST0}/ ${HOST1}/ ${HOST2}/
ETCDHOSTS=(${HOST0} ${HOST1} ${HOST2})
NAMES=("infra0" "infra1" "infra2")


for i in "${!ETCDHOSTS[@]}"; do
HOST=${ETCDHOSTS[$i]}
NAME=${NAMES[$i]}
cat << EOF > ${HOST}/kubeadm-config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
apiVersion: "kubeadm.k8s.io/v1beta1"
kind: ClusterConfiguration
# kubernetesVersion: stable
kubernetesVersion: v1.14.2
imageRepository: gcr.azk8s.cn/google_containers
# 这里配置了一个阿里云内网负载均衡作为入口,如果没有的话请自行忽略
# controlPlaneEndpoint: "172.18.221.7:6443"
networking:
# 规划pod CIDR
  podSubnet: 10.66.0.0/16
  serviceSubnet: 10.88.0.0/16
etcd:
    local:
        serverCertSANs:
        - "${HOST}"
        peerCertSANs:
        - "${HOST}"
        extraArgs:
            initial-cluster: ${NAMES[0]}=https://${ETCDHOSTS[0]}:2380,${NAMES[1]}=https://${ETCDHOSTS[1]}:2380,${NAMES[2]}=https://${ETCDHOSTS[2]}:2380
            initial-cluster-state: new
            name: ${NAME}
            listen-peer-urls: https://${HOST}:2380
            listen-client-urls: https://${HOST}:2379
            advertise-client-urls: https://${HOST}:2379
            initial-advertise-peer-urls: https://${HOST}:2380
EOF
done





kubeadm config images pull --config=kubeadm-config.yaml
sudo kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs

sudo kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs --ignore-preflight-errors=all

# todo:
kubeadm join 172.18.221.35:6443 --token l0ei3n.rqqqseno29oo564z \
    --discovery-token-ca-cert-hash sha256:9752be9ff3b619f5b6baadc98ed184e3e1dc2ff02b080aea2457b8f89496de2f
    
```


#### 单master型(实验用)

```
sudo tee kubeadm-config.yaml <<-'EOF'
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
# kubernetesVersion: stable
kubernetesVersion: v1.14.2
# 这里配置了一个阿里云内网负载均衡作为入口,如果没有的话请自行忽略
# controlPlaneEndpoint: "172.18.221.7:6443"
imageRepository: gcr.azk8s.cn/google_containers
networking:
# 规划pod CIDR
  podSubnet: 10.66.0.0/16
  serviceSubnet: 10.88.0.0/16
EOF

kubeadm config images pull --config=kubeadm-config.yaml
sudo kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs
```

#### 配置kubelet客户端

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### [配置网络插件](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network)

这里我选择`quay.io/coreos/flannel:v0.11.0-amd64`,因为架构比较齐全

#### 引入其他master节点(高可用版)

在`kubeadm init`的输出中,有一行是

```
[upload-certs] Using certificate key: 05ae8e3c139a960c6e4e01aebf26869ce5f9abd9fa5cf4ce347e8308b9c276f9
```

复制起来,在别的master上面运行命令

```
kubeadm join 172.18.221.35:6443 \
--token 29ciq3.5mtkr4nzc11mlzd6 \
--discovery-token-ca-cert-hash sha256:03b46745a1f3887417270e33fc9b3fb5ddd82599f0d0ec789ed4edf2c310faae \
--experimental-control-plane \
--certificate-key 05ae8e3c139a960c6e4e01aebf26869ce5f9abd9fa5cf4ce347e8308b9c276f9
```





### 工作节点加入集群

```bash
kubeadm join 172.18.221.35:6443 --token c63abt.45sn8bhyxxo2lh0r \
    --discovery-token-ca-cert-hash sha256:891e41e798c29f7235078479ca3e0622594c91db08160bea620f60fffcd558f5
```




### 收尾工作


```
  ipvsadm -l
```


### 其他参考

#### [kubelet-check] Initial timeout of 40s passed.

```
systemctl status kubelet
journalctl -xeu kubelet
```
通过以上任意一个命令看到,kubernetes服务虽然启动中,但是提示节点找不到.

```
May 20 14:55:22 xxx kubelet[3457]: E0520 14:55:22.095536    3457 kubelet.go:2244] node "xxx" not found
```

最后发现是一开始指定了负载均衡,负载均衡连接不上导致超时

 --ignore-preflight-errors=all


#### 修改driver之后的注意事项

如果docker是之前安装的,改一下配置然后重启服务即可

改成systemd要在kubelet的服务上要加多一个参数,不然服务无法启动


```bash
vi /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

```
# Restart docker.
systemctl daemon-reload
systemctl restart docker
systemctl restart kubelet
```

#### master参与调度

```bash
# 去掉master污点,让其参与调度
kubectl taint $node --all node-role.kubernetes.io/master-
```

#### 重置


```bash
kubeadm reset
ipvsadm --clear
```

#### 重新计算discovery-token-ca-cert-hash

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```


### todo

https://github.com/kubernetes/kubeadm/issues/1331

[证书轮换](https://kubernetes.io/docs/tasks/tls/certificate-rotation/)

[setup-ha-etcd-with-kubeadm](https://kubernetes.io/docs/setup/independent/setup-ha-etcd-with-kubeadm/)

### 参考链接

1. [Overview of kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/)
1. [阿里云镜像仓库](https://opsx.alibaba.com/mirror/search?q=kubelet&lang=zh-CN)
2. [官方安装指南](https://kubernetes.io/docs/setup/independent/install-kubeadm/)
1. [使用kubeadm安装kubernetes](http://bazingafeng.com/2017/11/20/using-kubeadm-install-kubernetes/)
2. [centos7安装kubeadm](http://www.maogx.win/posts/15/)
3. [centos7使用kubeadm安装k8s-1.11版本多主高可用](http://www.maogx.win/posts/33/)
4. [centos7使用kubeadm安装k8s集群](http://www.maogx.win/posts/16/)
5. [kubernetes集群的安装异常汇总](https://juejin.im/post/5bbf7dd05188255c652d62fe)
6. [kubeadm 设置工具参考指南](https://k8smeetup.github.io/docs/admin/kubeadm/)
7. [ipvs](https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/ipvs)

## 3.5 更新kubernetes大版本需要注意的问题


### 最大的坑是 deprecated apiVersion

`Kubernetes` 的 `apiVersion` 是会过期的

以 1.16来说,`DaemonSet`, `Deployment`, `StatefulSet`, `ReplicaSet` 全部统一使用 `apps/v1`

`NetworkPolicy`  使用 `networking.k8s.io/v1`

`PodSecurityPolicy` 使用 `networking.k8s.io/v1`

所以,在 `1.16` 中使用 `apps/v1beta2`, `extensions/v1beta1` 等废弃API都会出错

### 拥抱变化


#### 检查受影响资源

```bash
kubectl get NetworkPolicy,PodSecurityPolicy,DaemonSet,Deployment,ReplicaSet \
--all-namespaces \
-o 'jsonpath={range .items[*]}{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}{"\n"}{end}' | grep '"apiVersion":"extensions/v1beta1"'

kubectl get DaemonSet,Deployment,StatefulSet,ReplicaSet \
--all-namespaces \
-o 'jsonpath={range .items[*]}{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}{"\n"}{end}' | grep '"apiVersion":"apps/v1beta'

kubectl get --raw="/metrics" | grep apiserver_request_count | grep 'group="extensions"' | grep 'version="v1beta1"' | grep -v ingresses | grep -v 'client="hyperkube' | grep -v 'client="kubectl' | grep -v 'client="dashboard'

kubectl get --raw="/metrics" | grep apiserver_request_count | grep 'group="apps"' | grep 'version="v1beta' | grep -v 'client="hyperkube' | grep -v 'client="kubectl' | grep -v 'client="dashboard'

```

#### recreate

是的，你没有听错，只能删除后重建。

我的建议是，在业务低峰期，建同label deploy 覆盖旧的`resource`，旧的`resource`缩容至0,并加上`deprecated:true`的`label`观察一段时间后,再彻底删除.

### 后记

apiVersion 变动的频繁,在某种程度上也可以证明 `Kubernetes` 在容器调度方面的霸权——毕竟，如果你跟女朋友分手了，也不会想给她买新衣服，对吧？

![](/img/sticker/云原生开发.gif)

### 参考链接

1. [error: At least one of apiVersion, kind and name was changed](https://stackoverflow.com/questions/56386647/error-at-least-one-of-apiversion-kind-and-name-was-changed)
1. [Kubernetes Version 1.16 Removes Deprecated APIs](https://www.ibm.com/cloud/blog/announcements/kubernetes-version-1-16-removes-deprecated-apis)
1. [Kubernetes v1.17 版本解读](https://yq.aliyun.com/articles/739120)
1. [API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
1. [Kubernetes Deprecation Policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)
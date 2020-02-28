二十一 Kubernetes 生态系统
=====================

## 21.1 [kube-eventer](https://github.com/AliyunContainerService/kube-eventer)

Kubernetes 的事件导出工具,可以把 Kubernetes 的 Event 推送到多个数据源,比如钉钉,mysql,微信群等。

## 21.2 [kt-connect](https://github.com/alibaba/kt-connect)

`kt-connect`  是一个本地开发环境连接远端 Kubernetes 的工具。

在容器上 `Kubernetes` 的过程中，我们经常会遇到需要服务互访问的问题。比如 pod a 要访问 service b，但是 pod a 对应的程序还在开发中，如果可以在本地的程序直接访问`b`,那么就会节省很多时间（不需要部署 service b对应的环境，直接使用 virtual service name）。节约了部署时间，大大提高了开发排错效率。

`kt-connect` 目前有2个模式：
1. vpn模式
1. socks5

vpn模式是基于sshuttle，会把dns请求转到集群，如果集群能上外网的话也不会有影响。
mac/linux的执行ktctl connect以后就已经在vpn下了。

由于 Windows 不支持 sshuttle,所以只能用 socks5,socks5是走的ssh端口转发

中文教程:
https://alibaba.github.io/kt-connect/#/zh-cn/quickstart

`kt-connect` 目前 Istio 相关的调试工具正在开发中，

[相关项目](https://github.com/alibaba/virtual-environment)

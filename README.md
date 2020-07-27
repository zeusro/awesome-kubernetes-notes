# awesome-kubernetes-notes

[![Documentation Status](https://readthedocs.org/projects/awesome-kubernetes-notes/badge/?version=latest)](https://zeusro-awesome-kubernetes-notes.readthedocs.io/zh_CN/latest/?badge=latest)  ![](https://img.shields.io/badge/sphinx-python-blue.svg)  ![](https://img.shields.io/badge/python-3.6-green.svg)

为方便更多k8s爱好者更系统性的学习文档，利用`sphinx`将笔记整理构建程[在线文档](https://github.com/redhatxl/awesome-kubernetes-notes)，方便学习交流


贡献者：

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->

<table>
  <tr>
<td align="center"><a href="https://github.com/redhatxl">    <img src="https://avatars.githubusercontent.com/u/24467514?v=3" width="100px;"        alt="" /><br /><sub><b>redhatxl</b></sub></a><br /><a href="https://juejin.im/user/5c36033fe51d456e4138b473/posts" title="掘金">💬</a><a href="https://www.imooc.com/u/1260704"  title="慕课网">📖</a><a` href="https://github.com/zeusro/awesome-kubernetes-notes/pulls?q=is%3Apr+reviewed-by%3Akentcdodds"    title="Reviewed Pull Requests">👀</a>    <a href="#talk-kentcdodds" title="Talks">📢</a></td>
           <td align="center"><a href="https://www.zeusro.com/">    <img src="https://avatars.githubusercontent.com/u/5803609?v=3" width="100px;"        alt="" /><br /><sub><b>Zeusro</b></sub></a><br /><a href="" title="Answering Questions">💬</a><a href="https://github.com/zeusro"    title="Documentation">📖</a>                    <a href="https://github.com/zeusro/awesome-kubernetes-notes/pulls?q=is%3Apr+reviewed-by%3Akentcdodds"    title="Reviewed Pull Requests">👀</a> <a href="#talk-kentcdodds" title="Talks">📢</a></td>
  </tr>
</table>

<!-- markdownlint-enable -->
<!-- prettier-ignore-end -->
<!-- ALL-CONTRIBUTORS-LIST:END -->

## 参考资料

### 建议学习路线

#### [docker](https://yeasy.gitbooks.io/docker_practice/introduction/what.html)

理解docker快速分发,分层镜像,环境隔离,端口/目录映射,网络模式等重要特性.

#### [docker-compose/swam](https://docs.docker.com/compose/)

建议日常使用docker-compose替代docker命令启动镜像
  
#### swarm

swarm已是弃子，可忽略

#### kubernetes

理解基础概念后，通过minikube/kubeadm(配置建议4核8G以上)搭建集群，最后应用于生产中。生产环境一般使用3 master+N worker部署。

如果觉得自己部署很麻烦，可以试试阿里云的kubernetes托管版。master节点由阿里云负责运维。

### 官方文档(不要选中文)

https://kubernetes.io/docs/concepts/

API Reference(配置资源时用到):

https://kubernetes.io/docs/reference/

不同版本配置有一些细微差别，自己酌情

[kubectl旧版指南](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)

[kubectl新版指南](https://kubectl.docs.kubernetes.io/)

### kubernetes 相关镜像

#### [官方源](https://console.cloud.google.com/gcr/images/google-containers/GLOBAL)

#### [Azure加速](http://mirror.azure.cn/help/gcr-proxy-cache.html)

#### 阿里云(不是很全,有些没有)

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd6:v1.15.0-alpha.0 
```

拉取下来后用`docker tag`改名即可

需要说明的是,阿里云加速器只对 `hub.docker.com` 加速,套娃镜像得用海外机器搬运,或者使用阿里云的免费海外机器构建镜像服务。
地址为[容器镜像服务](https://cr.console.aliyun.com/)

### 其他文档

* [Kubernetes官网教程](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
* [Kubernetes中文社区](https://www.kubernetes.org.cn/k8s)
* [从Kubernetes到Cloud Native](https://jimmysong.io/kubernetes-handbook/cloud-native/from-kubernetes-to-cloud-native.html)
* [Kubernetes Handbook](https://www.bookstack.cn/read/feiskyer-kubernetes-handbook/appendix-ecosystem.md)
* [Kubernetes从入门到实战](https://www.kancloud.cn/huyipow/kubernetes/722822)
* [Kubernetes指南](https://kubernetes.feisky.xyz/)
* [awesome-kubernetes](https://ramitsurana.github.io/awesome-kubernetes/)
* [从Docker到Kubernetes进阶](https://www.qikqiak.com/k8s-book/)
* [python微服务实战](https://www.qikqiak.com/tdd-book/)
* [云原生之路](https://jimmysong.io/kubernetes-handbook/cloud-native/from-kubernetes-to-cloud-native.html)
* [CNCF Cloud Native Interactive Landscape](https://landscape.cncf.io/)

## 学习社群

### qq群

点击链接加入群聊【Kubernetes&Docker技术交流】（为了防机器人加了点付费模式）：https://jq.qq.com/?_wv=1027&k=27PyHReU

<img src="source/images/readme/qrcode_1595817926180.jpg" alt="QQ群" width='30%' height='30%'/>

### 阿里云官方 kubernetes 千人钉钉群

![](source/images/readme/aliyun-kubernetes.png)

常见问题，@K8s答疑 就能收获答案。

## TODO

- [ ] 日志收集
- [ ] CI/CD的DevOPS相关
- [ ] 完善 Kubernetes 生态系统

如果此笔记对你没有帮助，欢迎打钱🎉

![](source/images/readme/zeusro.jpg)
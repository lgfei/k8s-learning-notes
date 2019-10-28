# k8s-learning-notes
## k8s集群部署
- [kubeadm部署多master高可用集群](https://github.com/lgfei/k8s-learning-notes/blob/master/kubeadm/README.md)

## 我眼中的k8s
### 为什么要容器化
1. 节省服务器资源
2. 自动伸缩扩容
3. 方便迁移，一次构建到处部署
4. 
### 什么是Docker
1. Docker并不等于容器，Docker只是基于容器技术的一个产品。相比于其他容器产品Docker最大的优势和创新是镜像(image)
2. 通过docker run启动一个容器，是通过Linux Namespace、Linux Cgroups 和 rootfs 三种技术构建出来的进程的隔离环境，实际它只是运行在主机上的一个特殊的进程
3. 
### 什么是k8s，为什么需要k8s
1. 如果说Docker只是安装应用的另外一种形式，而k8s就是用于基于容器的操作系统。为Docker化的应用提供路由网关、水平扩展、监控、备份、灾难恢复等一系列运维能力。有了k8s才能真正走入DevOps的世界
2. 
### k8s的架构
### k8s的网络原理
### k8s的日志采集方案
### k8s监控方案Prometheus


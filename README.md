# k8s-learning-notes
## k8s集群部署
- [kubeadm部署多master高可用集群](https://github.com/lgfei/k8s-learning-notes/blob/master/kubeadm/README.md)

## 我眼中的k8s
### 为什么要容器化
1. 节省服务器资源
2. 自动伸缩扩容
3. 环境一致性
4. 方便迁移，一次构建到处部署
### 什么是Docker
1. Docker并不等于容器，Docker只是基于容器技术的一个产品。相比于其他容器产品，Docker最大的优势和创新是镜像(image)
2. 通过docker run启动一个容器，是通过Linux Namespace、Linux Cgroups 和 rootfs 三种技术构建出来的进程的隔离环境，实际它只是运行在主机上的一个特殊的进程
### 什么是k8s，为什么需要k8s
1. 如果说Docker只是安装应用的另外一种形式，那么k8s就是管理容器应用的操作系统，为Docker化的应用提供路由网关、水平扩展、监控、备份、灾难恢复等一系列运维能力。认识了k8s才能真正走入容器化的世界
2. k8s最重要的概念是Pod，Pod是k8s项目的原子调度单位，一个Pod可以包含一个或多个容器应用（通常只放一个）。所以你可以将一个pod看成我们传统的一台虚拟机。
### k8s的架构
1. 全局架构<br>
- ApiServer: k8s访问入口，所有通过kubectl执行的命令都是调用ApiServer实现的
- Scheduler: 调度室，决定一个Pod应该运行在哪个Node。（Pod运行的节点一般通过Node的label指定）
- Controller Manager: 总控室，监控集群状态，管理集群资源。例如：例如某一个应用设置的副本是2，其中一个意外停止，则Controller Manager负责重新创建一个Pod，保证应用副本个数是2。
- Etcd: key-value的数据库，负责持久化集群中各资源对象的信息
- kubelet: 主要负责和Docker交互
- kube-proxy: 负责处理外部请求应该访问到那个pod，nginx的反向代理<br>
![k8s-cluster](https://github.com/lgfei/k8s-learning-notes/raw/master/images/k8s-cluster.png)
2. 集群对象关系<br>
- Pod: 一个或多个紧密协作的容器应用组成的逻辑对象，每个Pod会分配一个虚拟的PodIP(主机模式用的是主机IP)，一个Pod内的容器共享Pod的IP和网络配置，用于同外界通信。
- Replica Set: Pod的子类，简称RC。一个RC可以管理多个Pod。
- Deployment: RC的子类，可以看成高版本的RC。提供了更丰富管理Pod的功能，例如：健康检查，滚动升级等。
- Service: 一组Pod的访问入口，并负责pod的负载均衡。一个Service会分配一个Cluster IP，并指定与主机和Pod的通信端口。
- Ingress: 一般和Service一起使用，可以自定义配置Service的负载均衡。
- ConfigMap/Secret: 都属于一种特殊的volume，负责存放一些环境相关的配置，方便多环境配置调整，只是Secret是加密的。
- DaemonSet: 会在每个或指定范围内的Node都运行一个Pod，且新增节点后会自动部署。例如：网络插件flannel
- StatefulSet: 有状态的应用。一般用来部署中间件，例如：mysql
- Job: 一次性任务
- CronJob: 定时任务
- Horizontal PodAutoscaler: 水平自动伸缩控制器<br>
![k8s-pod](https://github.com/lgfei/k8s-learning-notes/raw/master/images/k8s-pod.png)
### k8s的网络原理
k8s网络实则容器和容器的通信<br>
1. docker容器怎么和主机通信
宿主机安装完docker后，创建一个docker0网桥，执行ifconfig会看到有如下信息<br>
***注：其中192.168.5.1可以通过/etc/docker/daemon.json中bip配置项自行指定***
<pre>
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.5.1  netmask 255.255.255.0  broadcast 192.168.5.255
        inet6 fe80::42:ecff:fee9:1ed3  prefixlen 64  scopeid 0x20<link>
        ether 02:42:ec:e9:1e:d3  txqueuelen 0  (Ethernet)
        RX packets 47074  bytes 14036792 (13.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 47814  bytes 12944523 (12.3 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
.
.
.
vetha0087a6: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::ccf2:faff:fed1:cf71  prefixlen 64  scopeid 0x20<link>
        ether ce:f2:fa:d1:cf:71  txqueuelen 0  (Ethernet)
        RX packets 47074  bytes 14695828 (14.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 47822  bytes 12945171 (12.3 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
</pre>
在docker0网桥上有一个Veth Pair的虚拟网卡设备，正是通过这个虚拟设备容器可以和docker0通信，然后docker0则可以和主机直接通信<br>
至于docker0怎么和主机通信，我想应该和iptables技术有关<br>
<br>
2. A主机的容器怎么和B主机的容器通信
两个主机网络是连通的，但是两台主机上的docker0是不互通的，所以我们要通过软件的方式为两台主机构建一个虚拟网络Overlay Network。<br>
有了这个虚拟网络，两台主机上的容器通信就和单机类似了。<br>
这就是为什么k8s集群必须安装网络插件的原因，执行ifconfig会看到如下信息
<pre>
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 100.244.0.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::b07b:70ff:feae:1fe8  prefixlen 64  scopeid 0x20<link>
        ether b2:7b:70:ae:1f:e8  txqueuelen 0  (Ethernet)
        RX packets 21112  bytes 10048496 (9.5 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 23197  bytes 2119926 (2.0 MiB)
        TX errors 0  dropped 8 overruns 0  carrier 0  collisions 0
</pre>
3. 集群外访问集群内服务的过程<br>
Browser->Nginx->Service->Pod<br>
用户请求经过Nginx分发，根据Service的端口映射，找到对应的ClusterIP，在通过Service找到其关联的Pods，再通过Ingress分发给具体的Pod
### k8s日志采集方案
1. EFK
### k8s监控方案
2. Prometheus


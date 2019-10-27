# kubeadm安装高可用k8s集群过程
## 机器准备 
<pre> 
系统内核3.10以上，建议至少2核2G  
如果只是试验，可以只用1台mater,1台node。生产环境一般需要多master（至少3台），实现高可用   
此次我试验的架构如下：  
Kubernetes: v1.15.3  
Docker-ce: 18.06.1 
Keepalived保证apiserever服务器的IP高可用  
Haproxy实现apiserver的负载均衡 
</pre> 
节点名称|角色|IP|安装的软件
--|:--:|--:|--:
负载VIP|VIP|192.168.1.200|不是一台真实的机器，是一个与master同网段未被占用的虚拟IP
master-01|master|192.168.1.101|kubeadm、kubelet、kubectl、etcd、docker、haproxy、keepalived、ipvsadm
master-02|master|192.168.1.102|kubeadm、kubelet、kubectl、etcd、docker、haproxy、keepalived、ipvsadm
master-03|master|192.168.1.103|kubeadm、kubelet、kubectl、etcd、docker、haproxy、keepalived、ipvsadm
node-01|node|192.168.1.104|kubeadm、kubelet、kubectl、docker、ipvsadm
node-02|node|192.168.1.105|kubeadm、kubelet、kubectl、docker、ipvsadm

## 机器初始化配置
***注: 没有特别说明的，表示每台机器都要执行***
1. 关闭防火墙
```
systemctl disable firewalld
systemctl stop firewalld
```
2. 禁用selinux
```
sed -ri 's#(SELINUX=).*#\1disabled#' /etc/selinux/config
setenforce 0
```
3. 关闭swap
```
swapoff -a
```
4. 添加hosts
```
cat >>/etc/hosts<<EOF
192.168.1.101 master-01
192.168.1.102 master-02
192.168.1.103 master-03
192.168.1.104 node-01
192.168.1.105 node-02
EOF
```
5. 在master-01上创建ssh秘钥，并分发给其他节点，方便从master上复制文件到其他节点
```
cd ~
ssh-keygen -t rsa
ssh-copy-id master-02
ssh-copy-id master-03
...
```
6. 配置内核参数
```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
EOF
sysctl --system
```
7. 加载ipvs模块
```
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

## 部署keepalived和haproxy
***注: 只要在3台master节点部署***
1. 安装keepalived和haproxy
2. 修改keepalived配置
3. 修改haproxy配置
4. 启动服务

## 安装docker
***注: 每台机器都安装***
1. 添加yum源
```
cd /etc/yum.repos.d/
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
1. 安装docker
```
yum list docker-ce.x86_64 --showduplicates | sort -r
yum -y install docker-ce-18.06.1.ce-3.el7
```
2. 查看或修改/etc/docker/daemon.json

3. 启动
```
systemctl enable docker && systemctl start docker
docker -version
```

## 部署kubernetes
1. 添加yum源
```
cat << EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
2. 每台机都安装kubelet，kubectl，kubeadm
```
yum install -y kubelet-1.15.3 kubeadm-1.15.3 kubectl-1.15.3
```

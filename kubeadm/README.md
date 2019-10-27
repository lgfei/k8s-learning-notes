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
node-03|node|192.168.1.106|kubeadm、kubelet、kubectl、docker、ipvsadm

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
`yum install -y keepalived haproxy`
2. 修改keepalived配置/etc/keepalived/keepalived.conf
***注: master-01的priority 100，master-02的priority 99 ，master-03的priority 98***
```
global_defs {
   router_id csapiserver
   #vrrp_skip_check_adv_addr
   #vrrp_strict
   #vrrp_garp_interval 0
   #vrrp_gna_interval 0
}

vrrp_script chk_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    <font color="red">priority 100</font>
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        <font color="red">192.168.1.200</font>
    }
    track_script {
        chk_haproxy
    }
}
```
3. 修改haproxy配置/etc/haproxy/haproxy.cfg
```
global
    daemon
    nbproc    4
    user      haproxy
    group     haproxy
    maxconn   50000
    pidfile   /var/run/haproxy.pid
    log       127.0.0.1   local0
    chroot    /var/lib/haproxy

defaults
    log       global
    log       127.0.0.1   local0
    maxconn   50000
    retries   3
    balance   roundrobin
    option    httplog
    option    dontlognull
    option    httpclose
    option    abortonclose
    timeout   http-request 10s
    timeout   connect 10s
    timeout   server 1m
    timeout   client 1m
    timeout   queue 1m
    timeout   check 5s

listen stats :1234
    stats     enable
    mode      http
    option    httplog
    log       global
    maxconn   10
    stats     refresh 30s
    stats     uri /
    stats     hide-version
    stats     realm HAproxy
    stats     auth admin:admin@haproxy
    stats     admin if TRUE

listen kube-api-lb
    <font color="red">bind      0.0.0.0:8443</font>
    balance   roundrobin
    mode      tcp
    option    tcplog
    <font color="red">server    master-01 192.168.1.101:6443 weight 1 maxconn 10000 check inter 10s rise 2 fall 3</font>
    <font color="red">server    master-02 192.168.1.102:6443 weight 1 maxconn 10000 check inter 10s rise 2 fall 3</font>
    <font color="red">server    master-03 192.168.1.103:6443 weight 1 maxconn 10000 check inter 10s rise 2 fall 3</font>
```
4. 启动服务
```
systemctl enable keepalived && systemctl start keepalived 
systemctl enable haproxy && systemctl start haproxy 
```

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
2. 根据自己的情况修改/etc/docker/daemon.json
没有这个文件则新建，也可以没有这个文件，则按默认配置启动docker
```
{
  "registry-mirrors": ["https://iljr3exx.mirror.aliyuncs.com"],
  "insecure-registries":["registry.topmall.com:5000","hub.wonhigh.cn"],
  "disable-legacy-registry": true,
  "bip":"192.168.5.1/24"
}
```

3. 启动
```
systemctl enable docker && systemctl start docker
docker version
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
3. 编辑kubeadm初始化yaml文件kubeadm-init.yaml
***注: 红色字体部分根据实际情况修改***
```
apiVersion: kubeadm.k8s.io/v1beta1
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: <font color="red">192.168.1.101</font>
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: <font color="red">master-01</font>
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token  
  ttl: 72h0m0s
  usages:
  - signing
  - authentication

---
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: <font color="red">v1.15.3</font>
apiServer:
  extraArgs:
    service-node-port-range: <font color="red">3000-39999</font>
  CertSANs:
  - <font color="red">192.168.1.101</font>
  - <font color="red">192.168.1.102</font>
  - <font color="red">192.168.1.103</font>
  - <font color="red">master-01</font>
  - <font color="red">master-02</font>
  - <font color="red">master-03</font>
controlPlaneEndpoint: <font color="red">192.168.1.200:8443</font>
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
networking:
  dnsDomain: cluster.local
  serviceSubnet: <font color="red">100.96.0.0/12</font>
  podSubnet: <font color="red">100.244.0.0/16</font>
  
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs
```
4. 预下载镜像
```
kubeadm config images pull --config kubeadm-init.yaml
```
5. 初始化
```
kubeadm init --config kubeadm-init.yaml
```
初始化成功会输出如下日志信息，注意保存，红色字体部分很重要
<pre>
[init] Using Kubernetes version: v1.15.3
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [master-01 localhost] and IPs [192.168.1.101 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [master-01 localhost] and IPs [192.168.1.101 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [master-01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [100.96.0.1 10.234.9.126 192.168.1.200]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "admin.conf" kubeconfig file
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[apiclient] All control plane components are healthy after 40.509663 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.15" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node suzhou-tsc-k8s-master-10-234-9-126-vm.belle.lan as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node suzhou-tsc-k8s-master-10-234-9-126-vm.belle.lan as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: jtkhrx.w9w6u0s8stpaianz
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

<font color="red">
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
</font>

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities 
and service account keys on each node and then running the following as root:

<font color="red">
  kubeadm join 192.168.1.200:8443 --token jtkhrx.w9w6u0s8stpaianz \
    --discovery-token-ca-cert-hash sha256:11902c4de08e89cd7d2da1d7543e086720061ce48acf5ce48fec1f825c8aef44 \
    --control-plane
</font>	

Then you can join any number of worker nodes by running the following on each as root:

<font color="red">
kubeadm join 192.168.1.200:8443 --token jtkhrx.w9w6u0s8stpaianz \
    --discovery-token-ca-cert-hash sha256:11902c4de08e89cd7d2da1d7543e086720061ce48acf5ce48fec1f825c8aef44
</font>
</pre>
- [init]：指定版本进行初始化操作
- [preflight] ：初始化前的检查和下载所需要的Docker镜像文件
- [kubelet-start] ：生成kubelet的配置文件”/var/lib/kubelet/config.yaml”，没有这个文件kubelet无法启动，所以初始化之前的kubelet实际上启动失败。
- [certificates]：生成Kubernetes使用的证书，存放在/etc/kubernetes/pki目录中。
- [kubeconfig] ：生成 KubeConfig 文件，存放在/etc/kubernetes目录中，组件之间通信需要使用对应文件。
- [control-plane]：使用/etc/kubernetes/manifest目录下的YAML文件，安装 Master 组件。
- [etcd]：使用/etc/kubernetes/manifest/etcd.yaml安装Etcd服务。
- [wait-control-plane]：等待control-plan部署的Master组件启动。
- [apiclient]：检查Master组件服务状态。
- [uploadconfig]：更新配置
- [kubelet]：使用configMap配置kubelet。
- [patchnode]：更新CNI信息到Node上，通过注释的方式记录。
- [mark-control-plane]：为当前节点打标签，打了角色Master，和不可调度标签，这样默认就不会使用Master节点来运行Pod。
- [bootstrap-token]：生成token记录下来，后边使用kubeadm join往集群中添加节点时会用到
- [addons]：安装附加组件CoreDNS和kube-proxy
6. 为kubectl准备Kubeconfig文件
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
7. 添加master节点
- 将master-01的k8s证书复制到master-02，master-03
```
USER=root
CONTROL_PLANE_IPS="master-02 master-03"
for host in ${CONTROL_PLANE_IPS}; do
    ssh "${USER}"@$host "mkdir -p /etc/kubernetes/pki/etcd"
    scp /etc/kubernetes/pki/ca.* "${USER}"@$host:/etc/kubernetes/pki/
    scp /etc/kubernetes/pki/sa.* "${USER}"@$host:/etc/kubernetes/pki/
    scp /etc/kubernetes/pki/front-proxy-ca.* "${USER}"@$host:/etc/kubernetes/pki/
    scp /etc/kubernetes/pki/etcd/ca.* "${USER}"@$host:/etc/kubernetes/pki/etcd/
    scp /etc/kubernetes/admin.conf "${USER}"@$host:/etc/kubernetes/
done
```
- 加入集群
```
kubeadm join 192.168.1.200:8443 --token jtkhrx.w9w6u0s8stpaianz \
  --discovery-token-ca-cert-hash sha256:11902c4de08e89cd7d2da1d7543e086720061ce48acf5ce48fec1f825c8aef44 \
  --control-plane
```
8. 添加node节点(node-01,node-02,node-03)
***注: 和master节点的区别在于 --control-plane ***
kubeadm join 192.168.1.200:8443 --token jtkhrx.w9w6u0s8stpaianz \
    --discovery-token-ca-cert-hash sha256:11902c4de08e89cd7d2da1d7543e086720061ce48acf5ce48fec1f825c8aef44
9. 查看集群状态  
```
kubectl version
kubectl get cs
```
<pre>
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"} 
</pre>
`kubectl get nodes`
<pre>
NAME        STATUS      ROLES    AGE     VERSION
master-01   NotReady    master   2d18h   v1.15.3
master-02   NotReady    master   2d17h   v1.15.3
master-03   NotReady    master   2d5h    v1.15.3
node-01     NotReady    node     2d5h    v1.15.3
node-01     NotReady    node     2d17h   v1.15.3
node-01     NotReady    node     2d17h   v1.15.3
</pre>
10. 部署fannel或者calico 
从上一步看到节点的状态是NotReady，是因为还没部署网络插件
***注: 部署任何组件，一定不要直接用网上下载的yaml文件部署 ***  
***类似于 kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml ***  
*** 一定要下载下来仔细对比，修改相应配置项 ***  
kube-flannel.yml文件内容如下：
```yaml
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.flannel.unprivileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
    apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
spec:
  privileged: false
  volumes:
    - configMap
    - secret
    - emptyDir
    - hostPath
  allowedHostPaths:
    - pathPrefix: "/etc/cni/net.d"
    - pathPrefix: "/etc/kube-flannel"
    - pathPrefix: "/run/flannel"
  readOnlyRootFilesystem: false
  # Users and groups
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  # Privilege Escalation
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  # Capabilities
  allowedCapabilities: ['NET_ADMIN']
  defaultAddCapabilities: []
  requiredDropCapabilities: []
  # Host namespaces
  hostPID: false
  hostIPC: false
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  # SELinux
  seLinux:
    # SELinux is unsed in CaaSP
    rule: 'RunAsAny'
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
rules:
  - apiGroups: ['extensions']
    resources: ['podsecuritypolicies']
    verbs: ['use']
    resourceNames: ['psp.flannel.unprivileged']
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes/status
    verbs:
      - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "100.244.0.0/16", <font color="red"># 和kubeadm-init.yml中设置的podSubnet一致</font>
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-amd64
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - amd64
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.11.0-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-arm64
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - arm64
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.11.0-arm64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-arm64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-arm
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - arm
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.11.0-arm
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-arm
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-ppc64le
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - ppc64le
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.11.0-ppc64le
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-ppc64le
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-s390x
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - s390x
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.11.0-s390x
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-s390x
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
```
flannel插件部署成功后，所有节点的状态会依次变成Ready
11. kubectl常用命令
- 标签管理
```
kubectl label nodes node-01 node-role.kubernetes.io/node=            // 标记为node节点
kubectl label nodes node-01 k8s.lgfei.com/namespace=dev              // 添加标签
kubectl label nodes node-01 k8s.lgfei.com/namespace=prd --overwrite  // 修改标签
kubectl label nodes node-01 k8s.lgfei.com/namespace-                 // 删除标签
```
- 组件管理
```
kubectl get cs
kubectl get nodes --show-labels
kubectl get ns
kubectl get pods --all-namespaces
kubectl get svc
kubectl get deployment
kubectl get configmaps
kubectl describe pod xxx
kubectl delete pod xxx
...
```

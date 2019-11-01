# K8S软件包管理工具Helm安装部署
Helm是K8S生态系统中的一个软件包管理工具，就相当于Linux的yum和apt工具。简化K8S安装，升级，删除，回滚应用等操作过程。<br>
Helm2.x版本的安装包括客户端helm和服务端Tiler（本文版本v2.15.0）<br>
Helm3.x版本则不再需要Tiler

## 安装 Helm 客户端
[下载地址](https://github.com/helm/helm/releases/tag/v2.15.0)<br>
```
mkdir -p /opt/helm/src
cd /opt/helm/src
tar -zxvf helm-v2.15.0-linux-amd64.tar.gz
cp linux-amd64/helm /usr/local/bin/
```

## 安装 Helm 服务器端 Tiller
1. Tiller 是以 Deployment 方式部署在 Kubernetes 集群中，使用阿里云镜像安装并把默认仓库设置为阿里云上的镜像仓库。<br>
```
helm init --upgrade --tiller-image registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.15.0 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```
2. 创建服务帐号和绑定角色
```
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
kubectl get deploy --namespace kube-system   tiller-deploy  --output yaml|grep  serviceAccount
```
3. 验证是否安装成功
```
kubectl -n kube-system get deployment | grep tiller
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
tiller-deploy   1/1     1            1           7d17h

kubectl -n kube-system get pods | grep tiller
NAME                               READY   STATUS    RESTARTS   AGE
tiller-deploy-84fc6949cd-k2x9b     1/1     Running   0          7d17h

helm version
Client: &version.Version{SemVer:"v2.15.0", GitCommit:"c2440264ca6c078a06e088a838b0476d2fc14750", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.15.0", GitCommit:"c2440264ca6c078a06e088a838b0476d2fc14750", GitTreeState:"clean"}
```
4. 卸载Tiller
```
helm reset
```

## 参考文献
- [Helm入门指南](https://www.hi-linux.com/posts/21466.html)

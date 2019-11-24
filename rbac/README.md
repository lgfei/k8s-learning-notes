# K8S集群RBAC权限控制
本文示例创建pod日志查看用户k8sloger

## 安装证书生成工具cfssl
```
mkdir -p /opt/cfssl
cd /opt/cfssl

wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64

chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64

mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

## 准备ca配置json文件
```
mkdir -p /opt/k8s/rbac/ssl
cd /opt/k8s/rbac/ssl

cat >  ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "expiry": "87600h",
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ]
      }
    }
  }
}
EOF
```

## 创建用户的证书签署请求配置json文件
CN即comman name，后面用户证书认证时使用的用户名
```
cd /opt/k8s/rbac/ssl

cat >  k8sloger-csr.json <<EOF
{
  "CN": "k8sloger",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

## 生成k8sloger的证书
```
cfssl gencert -ca=/etc/kubernetes/pki/ca.crt -ca-key=/etc/kubernetes/pki/ca.key -config=./ca-config.json -profile=kubernetes ./k8sloger-csr.json | cfssljson -bare k8sloger
```
会生成下面3个文件
<pre>
k8sloger-key.pem  k8sloger.pem  k8sloger.csr
</pre>

## 生成用户的专属配置文件
完整的配置文件包含3块，分别是cluster/context/user部分，包含相应的内容，分3个步骤生成<br>
1. cluster部分<br>
复制/etc/kubernetes/admin.conf的cluster部分到/opt/k8s/rbac/ssl/k8sloger.kubeconfig
```
cd /opt/k8s/rbac/ssl/

cat >  k8sloger.kubeconfig <<EOF
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: [CERT DATA 此处信息不公开]
    server: https://192.168.1.200:6443
  name: kubernetes
EOF
```
2. context部分
```
kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=k8sloger \
--namespace=default \
--kubeconfig=k8sloger.kubeconfig
```
3. user认证部分
```
kubectl config set-credentials k8sloger \
--client-certificate=./k8sloger.pem \
--client-key=./k8sloger-key.pem \
--embed-certs=true \
--kubeconfig=k8sloger.kubeconfig
```
最终生成的k8sloger.kubeconfig文件内容即k8s集群认证信息

## 为用户绑定角色
1. 创建角色
```
cd /opt/k8s/rbac

cat > k8sloger-role.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: my-namespace
  name: k8sloger-role
rules:
- apiGroups: [""]
  resources: ["pods","pods/log","configmaps","services"]
  verbs: ["get", "list","logs","describe"]
EOF

kubectl apply -f k8sloger-role.yaml
```
2. 绑定角色
```
cat > k8sloger-role-binding.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: my-namespace
  name: k8sloger-role-binding
subjects:
- kind: User
  name: k8sloger
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: k8sloger-role
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f k8sloger-role-binding.yaml
```

## 验证
1. 用k8sloger.kubeconfig替换现有的config
```
cd /opt/k8s/rbac/ssl/
mv ~/.kube/config ~/.kube/config.bak
cp k8sloger.kubeconfig ~/.kube/config
```
此时会发现kubectl get pods 正常返回，kubectl get nodes 会提示
<pre>
Error from server (Forbidden): nodes is forbidden: User "k8sloger" cannot list nodes at the cluster scope
</pre>
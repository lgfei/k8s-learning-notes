# 二进制方式安装高可用etcd集群
## 机器准备
节点名称|IP
--|:--:|
etcd01|192.168.1.101
etcd02|192.168.1.102
etcd03|192.168.1.103
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
## 生成key
- 创建目录
```
mkdir -p /opt/ssl/etcd
cd /opt/ssl/etcd
```
- ca-config.json
```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
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
- ca-csr.json
```
cat > ca-csr.json <<EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF
```
- server-csr.json
```
cat > server-csr.json <<EOF
{
    "CN": "etcd",
    "hosts": [
    "192.168.1.101",
    "192.168.1.102",
    "192.168.1.103"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF
```
- 生成证书(ca-key.pem  ca.pem  server-key.pem  server.pem)
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
```
## 部署etcd 
***3个节点都操作***
1. 下载 [下载地址](https://github.com/etcd-io/etcd/releases/tag/v3.3.17)
```
mkdir -p /opt/etcd/src
cd /opt/etcd/src
tar -zvxf etcd-v3.3.17-linux-amd64.tar.gz
mkdir -p /opt/etcd/{bin,cfg,ssl}
mv etcd-v3.3.17-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
```

2. 添加etcd配置文件
***3台机器ETCD_NAME名字不一样，本机IP不一样***
```
cd /opt/etcd/cfg
cat > etcd <<EOF
#[Member]
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.1.101:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.1.101:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.1.101:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.101.1:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://192.168.1.101:2380,etcd02=https://192.168.1.102:2380,etcd03=https://192.168.1.103:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```
- ETCD_NAME 节点名称
- ETCD_DATA_DIR 数据目录
- ETCD_LISTEN_PEER_URLS 集群通信监听地址
- ETCD_LISTEN_CLIENT_URLS 客户端访问监听地址
- ETCD_INITIAL_ADVERTISE_PEER_URLS 集群通告地址
- ETCD_ADVERTISE_CLIENT_URLS 客户端通告地址
- ETCD_INITIAL_CLUSTER 集群节点地址
- ETCD_INITIAL_CLUSTER_TOKEN 集群Token
- ETCD_INITIAL_CLUSTER_STATE 加入集群的当前状态，new是新集群，existing表示加入已有集群

3. 添加etcd.service
```
cd /usr/lib/systemd/system/
cat > etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd
ExecStart=/opt/etcd/bin/etcd \
--name=${ETCD_NAME} \
--data-dir=${ETCD_DATA_DIR} \
--listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
--listen-client-urls=${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
--advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
--initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
--initial-cluster=${ETCD_INITIAL_CLUSTER} \
--initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
--initial-cluster-state=new \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
4. 复制证书文件
```
cp /opt/ssl/etcd/ca*pem  /opt/etcd/ssl/
cp /opt/ssl/etcd/server*pem  /opt/etcd/ssl/

scp  /opt/ssl/etcd/ca*pem  root@192.168.1.102:/opt/etcd/ssl/
scp  /opt/ssl/etcd/ca*pem  root@192.168.1.102:/opt/etcd/ssl/
scp  /opt/ssl/etcd/server*pem  root@192.168.1.103:/opt/etcd/ssl/
scp  /opt/ssl/etcd/server*pem  root@192.168.1.103:/opt/etcd/ssl/
```
5. 启动
```
systemctl enable etcd
systemctl start etcd
```
6. 检查
```
cd /opt/ssl/etcd/
/opt/etcd/bin/etcdctl --ca-file=ca.pem --cert-file=server.pem --key-file=server-key.pem --endpoints="https://192.168.1.101:2379,https://192.168.1.102:2379,https://192.168.1.103:2379" cluster-health
```
出现如下信息，则安装成功
<pre>
member 65f49728d3d54972 is healthy: got healthy result from https://192.168.1.101:2379
member d94ba21c17c75ffb is healthy: got healthy result from https://192.168.1.102:2379
member dc51f874259f7894 is healthy: got healthy result from https://192.168.1.103:2379
cluster is healthy
</pre>
***如果有问题第一步先看日志：/var/log/message 或 journalctl -u etcd***
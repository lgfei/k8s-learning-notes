# 镜像仓库Harbor部署
存放docker镜像的仓库，也可以存放helm安装包。跟[Docker Hub](https://hub.docker.com)是一样的东西

# 环境准备
机器ip：192.168.2.101<br/>
预先安装docker，安装过程略
```
docker version
```

## Docker Compose安装
***通过docker-compose.yml修改端口***<br>
```
yum install -y docker-compose
docker-compose --version
```
如果使用http的方式配置harbor需要为所有Docker添加信任配置。"insecure-registries" : ["http://192.168.2.101"]
```
vim /etc/docker/daemon.json
{

"registry-mirrors": ["https://tdimi5q1.mirror.aliyuncs.com"],

"insecure-registries" : ["http://192.168.2.101"]

}
systemctl restart docker
```

## 下载Harbor 
- [下载地址](https://github.com/vmware/harbor/releases)<br>
建议下载offline的压缩包，里面包含了harbor启动所用的所有docker镜像，可以快速的部署harbor<br>
```
mkdir -p /opt/harbor/src
cd /opt/harbor/src
wget https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-offline-installer-v1.8.1.tgz
mkdir -p /opt/harbor/1.8.0
cp harbor-offline-installer-v1.8.1.tgz /opt/harbor/1.8.0
cd /opt/harbor/1.8.0
tar zxf harbor-offline-installer-v1.8.1.tgz
```

## 安装Harbor
***通过harbor.yml修改hostname***<br>
```
cd /opt/harbor/1.8.0
#如果要支持上传helm Chart 则执行 ./install.sh   --with-clair --with-chartmuseum
./install.sh
docker-compose ps
```

## 使用
<pre>
使用Harbor管理Registry 
web登录：http://192.168.2.101  默认用户名密码  admin/Harbor12345
后台登录：docker loginhttp://192.168.2.101

提交镜像到Registry
docker tag centos:latest 192.168.2.101/system/centos:latest
docker push 192.168.2.101/system/centos:latest
</pre>

## 重置&重启
```
# 停止
cd /opt/harbor/1.8.0
docker-compose down
# 修改harbor.yml重新生成配置
./prepare
# 重启
docker-compose up –d
# 推倒重来
docker-compose down
rm -rf database registry
./install.sh
```

## 参考文献
- [m.unixhot.com](http://m.unixhot.com/docker/registry.html)

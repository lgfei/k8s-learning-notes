# 镜像仓库Harbor部署
机器ip：192.168.2.101
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
## 下载Harbor [下载地址](https://github.com/vmware/harbor/releases)
建议下载offline的压缩包，里面包含了harbor启动所用的所有docker镜像，可以快速的部署harbor<br>
```
mkdir -p /opt/harbor/src
cd /opt/harbor/src
wget https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-offline-installer-v1.8.1.tgz
tar zxf harbor-offline-installer-v1.8.1.tgz
```
## 安装Harbor
***通过harbor.yml修改hostname***<br>
```
cd /usr/local/src/harbor/
./install.sh
docker-compose ps
```
## 使用
<pre>
使用Harbor管理Registry 
web登录：http://192.168.10.1  默认用户名密码  admin/Harbor12345
后台登录：docker login http://192.168.10.1

提交镜像到Registry
docker tag centos:latest 192.168.10.1/system/centos:latest
docker push 192.168.10.1/system/centos:latest
</pre>
## 参考文献
- [m.unixhot.com](http://m.unixhot.com/docker/registry.html)
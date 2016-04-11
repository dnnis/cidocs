# cidocs
jenkins+docker+ansible

# 基于 Jenkins+Docker+Ansible的maven项目集成
目前可以在svn提交之后自动build 编译并build docker镜像 发布 并用ansible部署到测试环境
#TODO
*  etcd
*  confd
*  nginx dynamic  upstream
*  CD

所用版本:
*  Centos 6.5 X64
*  Jenkins 1.6.5
*  Docker 1.7.1
*  MAVEN 3.2.5
*  Ansible 1.9.3

服务器两台
|IP|用途|
|------|------|
|192.168.0.151|Jenkins+Docker HuB|
|192.168.0.150|Apps|

安装docker:
docker需要内核 3.10支持 centos os 6.5的默认内核版本为 2.6.32 故需要升级内核
安装elrepo和epel源
``` shell
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-6-6.el6.elrepo.noarch.rpm
yum --enablerepo=elrepo-kernel install kernel-lt -y
yum install -y epel-release
```
安装完毕以后记得更改/etc/grub.conf 选择默认的启动内核为刚安装的内核版本
重启服务器

安装docker
``` shell
yum install -y docker-io
```

由于我的磁盘分区都挂在在/data目录下 
更改docker配置

安装jenkins

 docker run -d -p 5000:5000 --restart=always --name registry \
  -v /data/AEData/repositories:/var/lib/registry \
  registry:2
  
  
  other_args="-g /data/AEData/docker --insecure-registry 192.168.0.151:5000"
  
  
  docker rmi repo
  docker tag -f src dest

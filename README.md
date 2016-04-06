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

 docker run -d -p 5000:5000 --restart=always --name registry \
  -v /data/AEData/repositories:/var/lib/registry \
  registry:2
  
  
  other_args="-g /data/AEData/docker --insecure-registry 192.168.0.151:5000"
  
  
  

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
*  JDK 1.8

服务器两台

| IP | 用途 |
| ------ | ------ |
| 192.168.0.151 | Jenkins+Docker registry |
| 192.168.0.150 | Apps |

在上面两台服务器上安装docker:
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
``` BASH
yum install -y docker-io
```

由于我的磁盘分区都挂在在/data目录下 
更改docker配置
``` BASH
# /etc/sysconfig/docker 更改 other_args 为
>> other_args="-g /data/AEData/docker --insecure-registry 192.168.0.151:5000"
```
启动docker
``` BASH
 serverce docker start
 chkconfig docker on
 
```

在 `192.168.0.151` 上运行docker registry
``` BASH
 docker run -d -p 5000:5000 --restart=always --name registry \
-v /data/AEData/repositories:/var/lib/registry \
 registry:2
```
在 `192.168.0.151` 上安装 jenkins maven  
``` BASH
rpm -Uvh http://pkg.jenkins-ci.org/redhat-stable/jenkins-1.642.4-1.1.noarch.rpm
vi /etc/sysconfig/jenkins #修改java 路径 以及jenkins home路径 以及jenkins 用户 本环境使用root运行jenkins
| ...
| JENKINS_HOME="/data/AEData/jenkins"
| JENKINS_JAVA_CMD="/usr/local/jdk1.8.0_73/bin/java"
| JENKINS_USER="root"
| ...
#启动jenkins
service jenkins start
chkconfig jenkins on
```
安装maven 并配置maven私服 ，本环境使用了maven 3.2.5 setting.xml大致配置如下
``` XML
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <pluginGroups>
	<pluginGroup>org.mortbay.jetty</pluginGroup>
  </pluginGroups>
  <proxies>
  </proxies>
  <servers>
      <server>
      <id>snapshots</id>
      <username>admin</username>
      <password>admin</password>
    </server>
     <server>
      <id>releases</id>
      <username>admin</username>
      <password>admin</password>
     </server>
  </servers>
  <mirrors>
 <mirror>
		<id>Nexus</id>
		<name>Nexus Public Mirror</name>
		<url>http://192.168.0.34:8081/nexus/content/groups/public</url>
		<mirrorOf>central</mirrorOf>
	</mirror>
  </mirrors>
  <profiles>
 	  <profile>  
			<id>dev</id>  
			<repositories>  
				  <repository>  
					<id>local-nexus</id>  
					<url>http://192.168.0.34:8081/nexus/content/groups/public/</url>  
					<releases>  
					  <enabled>true</enabled>  
					</releases>  
					<snapshots>  
					  <enabled>true</enabled>  
					</snapshots>  
				  </repository>  
			</repositories>  
	  </profile> 

	  <profile>  
			<id>releases</id>  
			<repositories>  
				  <repository>  
					<id>Releases</id>  
					<url>http://192.168.0.34:8081/nexus/content/repositories/releases/</url>  
					<releases>  
					  <enabled>true</enabled>  
					</releases>  
					<snapshots>  
					  <enabled>true</enabled>  
					</snapshots>  
				  </repository>  
			</repositories>  
	  </profile> 
  </profiles>
 <activeProfiles>
        <activeProfile>dev</activeProfile>
    </activeProfiles>
</settings>

```

安装 Ansible 和docker-py
``` Bash
yum install ansible -y
pip install docker-py==1.4.0
#注意 这里docker-py需要和docker版本相对照 否则会报错. 
#安装完以后在实际运行中会报 DEFAULT_DOCKER_API_VERSION 未定义的错误 修改
vi /usr/lib/python2.6/site-packages/ansible/modules/core/cloud/docker/docker.py #在 大约355行增加如下内容

DEFAULT_DOCKER_API_VERSION = None 

```

在 `192.168.0.150` 上安装dokcer-py 安装方法参照 上文

设置 `192.168.0.151` ssh免密码登录 `192.168.0.150` 

在 `192.168.0.151` 上设置下Ansible的hosts 和增加一个playbook 用于部署
``` BASH
vi /etc/ansible/hosts 
#add hosts
192.168.0.150
#end
cat > task-playbook.yml<<EOF
---

- hosts: "{{host}}"
  tasks:
    - name: stop docker images
      docker:
        name: "{{POM_ARTIFACTID}}_{{InsID}}"
        registry: "192.168.0.151:5000"
        image: "bojoy/{{POM_ARTIFACTID}}"
        state: absent
    - name: pull the latest version
      docker:
        pull: "always"
        image: "192.168.0.151:5000/bojoy/{{POM_ARTIFACTID}}"
        name: "{{POM_ARTIFACTID}}_{{InsID}}"
        state: started
        restart_policy: always
        ports: "{{Ports}}"
        volumes:
          - /data/tomcatlogs/{{POM_ARTIFACTID}}_{{InsID}}:/usr/local/tomcat-6.0.45/logs:rw
 EOF

#因为我们需要把tomcat的日志保留 所以映射了一个本地目录到容器
```

制作自己的docker基础镜像
```
yum install febootstrap -y
febootstrap -i centos-release -i vim -i lrzsz  -i tar -i coreutils -i yum -i shadow-utils -i initscripts   centos6x /root/centos6  http://mirrors.163.com/centos/6/os/x86_64/
#生成docker镜像
tar -c .|docker import – 192.168.0.151:5000/centos6-base
docker push  192.168.0.151:5000/centos6-base
```

制作tomcat+java环境镜像
``` BASH
FROM centos6-base
MAINTAINER Haifeng <smhchf@gmail.com>
COPY jdk1.6.0_20 /usr/local/jdk1.6.0_20
COPY tomcat-6.0.45 /usr/local/tomcat-6.0.45
RUN useradd tomcat
RUN chown -R tomcat.tomcat /usr/local/tomcat-6.0.45
ENV JAVA_HOME /usr/local/jdk1.6.0_20
USER tomcat
EXPOSE 8080
WORKDIR /usr/local/tomcat-6.0.45/bin
CMD ./startup.sh && tail -f ../logs/catalina.out

```
>  注意 CMD ./startup.sh && tail -f ../logs/catalina.out  也可以用 CMD ['catalina.sh','run'] 但是其日志都会输出到标准输出，不方便收集
>  未深入研究

docker  build -t  192.168.0.151:5000/bojoy/centos6-tomcat .

docker push 192.168.0.151:5000/bojoy/centos6-tomcat

安装Jenkins 
到Jenkins官网下载最新的安装包 redhat系列rpm安装包地址 http://pkg.jenkins-ci.org/redhat-stable/
rpm -Uvh http://pkg.jenkins-ci.org/redhat-stable/jenkins-1.651.1-1.1.noarch.rpm
注意 jenkins需要 jdk 1.7以上支持
安装完成以后
service jenkins start
chkconfig jenkins on

配置jenkins,主要配置以下内容
1.插件的安装
2.JDK环境
3.maven环境
4.添加第一个项目

虽然构建使用了docker环境 但是未使用相关的docker插件 使用执行shell命令实现了docker的管理


1. 插件的安装
jenkins 有着丰富强大的插件库，安装插件通过 页面左侧的 "系统管理" --> "管理插件" ---> "可选插件" 进行安装
我这边安装了以下插件:
1.Ansible
2.Parameterized Trigger plugin

2. 环境配置
通过页面左侧的 "系统管理" --> "系统设置" 进行配置
主要配置添加jdk ansible maven环境 配置如下

3.添加第一个项目 
  我们的项目均为java 使用Maven构建
  通过页面左侧的 "系统管理" --> "新建" ---> "构建一个maven项目" 并为这个项目命名 如"3in1_task"
  点击OK J进入项目配置界面.
  
  我们主要配置 项目的scm的地址 maven 构建参数 脚本设置





  docker rmi repo
  docker tag -f src dest

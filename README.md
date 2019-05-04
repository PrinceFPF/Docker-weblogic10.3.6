# Docker-Weblogic 10.3.6 #
- 基础环境准备
- 安装weblogic
- 生成weblogic基础镜像
- 编制 Dockerfile 定制镜像
## 基础环境准备 ##
**1. 拉取并保存centos镜像**
```
$ sudo docker pull centos
$ sudo docker save -o centos-7.2.tar centos:latest
```
**2. 下载jdk rpm包**
[rpm:jdk-7u80-64bit](https://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase7-521261.html#jdk-7u80-oth-JPR) 

**3. 下载weblogic的zip版安装包**
[ZIP: weblogic-10.3.6-develop-zip](https://www.oracle.com/technetwork/middleware/weblogic/downloads/wls-for-dev-1703574.html) 

**4. 安装包准备**
```
# 新建文件夹 CVE-Docker
$ mkdir CVE-Docker

# 将weblogic安装包及jdk安装包拷贝至 CVE-Docker
$ cp *.* /home/hacker/CVE-Docker

# 解压wls1036_dev.zip,解压后的顶层文件目录是wls10360
$ sudo unzip wls1036_dev.zip -d wls10360
```
**5. 导入centos基础镜像**
```
$ sudo docker load -i centos-7.2.tar
```
## 安装weblogic ##
```
# step1 运行基础centos容器,把安装包目录映射到容器的home目录中
$ sudo docker run -itd -P -v /home/hacker/CVE-Docker:/tmp --name "install_weblogic" centos:latest /bin/bash

# step2 进入容器
$ sudo docker exec -it install_weblogic /bin/bash

# step3 设置root口令
[root@e7063152f8de /]# passwd root
root
root

# step4 创建weblogic用户
[root@e7063152f8de home]# useradd weblogic -p weblogic123

# step5 安装jdk
[root@e7063152f8de home]# cd /tmp
[root@e7063152f8de home]# rpm -ivh jdk-7u80-linux-x64.rpm

# step6 寻找jdk安装目录，并复制
[root@e7063152f8de jdk1.7.0_80]# find / -name "*jdk*"

# step7 设置weblogic安装目录
[root@e7063152f8de jdk1.7.0_80]# mkdir -p /opt/Oracle/weblogic/wls10360


# step8 设置环境变量，在profile尾部添加如下配置
[root@e7063152f8de jdk1.7.0_80]# vi /etc/profile

export JAVA_HOME=/usr/java/jdk1.7.0_80
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export MW_HOME=/opt/Oracle/weblogic/wls10360

# step9 使环境变量生效
[root@e7063152f8de jdk1.7.0_80]# source /etc/profile

# step10  将weblogic安装包文件拷贝至安装目录
[root@e7063152f8de wls10360]# cd /tmp/wls10360
[root@e7063152f8de wls10360]# cp -R ./* /opt/Oracle/weblogic/wls10360/

# step11 将weblogic安装目录及文件所有者修改为weblogic用户
[root@e7063152f8de weblogic]# cd /opt
[root@e7063152f8de opt]# chown -R weblogic:weblogic Oracle/

# step12 切换至weblogic用户
[root@e7063152f8de wls10360]# su - weblogic

# step12 檢查當前環境是否符合weblogic安裝要求
[weblogic@e7063152f8de ~]$ cd /opt/Oracle/weblogic/wls10360/
[weblogic@e7063152f8de wls10360]$ ./configure.sh

# step13 配置weblogic初始化參數
[weblogic@e7063152f8de wls10360]$ wlserver/server/bin/setWLSEnv.sh

#step14 创建weblogic域(domain)
[weblogic@e7063152f8de wls10360]$ wlserver/common/bin/config.sh

#step15 选1,创建新域
#step16 选1, 选择WebLogic平台组件 
#step17 默认回车, Basic WebLogic Server Domain - 10.3.6.0 [wlserver]
#step18 默认回车,使用base_domain作为域名
#step19 默认鬼扯,使用/opt/Oracle/weblogic/wls10360/user_projects/domains作为域的安装路径
#step20 选1,设置登录管理员用户名,设置为weblogic回车
#step21 选2,设置登录管理员用户口令,设置为weblogic123回车
#step22 选3,确认登录管理员用户口令,输入weblogic123回车
#step23 回车进入下一步,选择1,开发环境
#step24 选1,选择我们安装的jdk作为java环境
#step25 选1,因为我们只安装单节点,所以选1,如果要部署集群则选择2
#step26 默认回车,确认weblogic 管理服务信息，回车进行确认

# step27 启动weblogic,测试安装成效
[weblogic@e7063152f8de wls10360]$ user_projects/domains/base_domain/startWebLogic.sh

#step28 切换到root用户,安装net-tools软件包,查看服务
[root@e7063152f8de /]# yum install net-tools
[root@e7063152f8de /]# netstat -ntlp
[root@e7063152f8de /]# curl http://127.0.0.1:7001/console
```
## 生成weblogic基础镜像 ##
**1. 生成weblogic基础镜像**
```
$ sudo docker commit install_weblogic weblogic:10.3.6
```
## 编制 Dockerfile 定制镜像 ##
**1. 编制Dockerfile文件**
```
# Version 1.0

# base image
FROM weblogic:10.3.6

# Author informations
MAINTAINER 1159739336@qq.com

# 将启动后的目录切换到 /home/weblogic目录
WORKDIR  /home/weblogic

# Add the locate file to container
ADD start.sh /home/weblogic

# 使用root用户来执行后续命令
USER root

# Running some commonds
RUN source /etc/profile
RUN chown weblogic:weblogic /home/weblogic/start.sh
RUN chmod +x /home/weblogic/start.sh

# 使用weblogic用户来执行后续命令
USER weblogic

# Expose the port 7001
EXPOSE 7001

# The commond running after container started
CMD ["/home/weblogic/start.sh"]
```
**2.编制附加文件start.sh**
```
#!/bin/bash
source /etc/profile
/opt/Oracle/weblogic/wls10360/user_projects/domains/base_domain/startWebLogic.sh
```
**3.制作最终镜像**
```
$ sudo docker build -t weblogic:10.3.6 .
```


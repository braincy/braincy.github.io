---
layout: post
title: 一些常用软件的环境配置
date: 2018-06-06 12:11:38
tags: [CentOS, 环境配置]
categories: 环境配置
mathjax: true
---

* content
{:toc}

本文记录了一些常用软件的环境配置，包括 *Git、MongoDB、Redis等*，大部分是在CentOS上的配置方法，方便查阅。




## Git源码编译安装
Git是一款免费、开源的分布式版本控制系统，用于敏捷高效地处理任何或小或大的项目。

很多yum源上自动安装的git版本为1.7.1，而Github及其他使用Coding管理项目时需要的Git版本最低都不能低于1.7.2 。所以我们一般不用 `yum -y install git` 这种方法，而是下载git源码编译安装比较新的版本。如下是在CentOS7上编译安装步骤：

### 删除已有的git
```bash
[root@localhost ~]# git --version  //查看当前版本号
git version 1.8.3.1
[root@localhost ~]# yum remove git  //删除原有版本
[root@localhost ~]# git --version  //再次查看版本号确认已删除成功
-bash: /usr/bin/git: No such file or directory
```
### 下载git源码并解压
```bash
[root@localhost ~]# wget https://www.kernel.org/pub/software/scm/git/git-2.9.3.tar.gz  //下载git源码
[root@localhost ~]# tar xf git-2.9.3.tar.gz   //解压git安装包
[root@localhost ~]# mv git-2.9.3 /usr/src   //移动到/usr/src目录下
```
### 编译安装
源码的安装一般由3个步骤组成：配置(configure)、编译(make)、安装(make install)。
```bash
[root@localhost git-2.9.3]# make configure  

    GEN configure
./configure prefix=/usr/local/git/
[root@localhost git-2.9.3]# ./configure prefix=/usr/local/git/  //配置git安装路径
[root@localhost git-2.9.3]# make && make install  //编译并且安装
```
### 将git指令添加到bash中
```bash
[root@localhost ~]# vi /etc/profile  //打开文件
按下字母i键进入编辑模式，在最后一行加入如下命令，加入后按Esc键退出编辑模式，并用:wq命令保存。
　　export PATH=$PATH:/usr/local/git/bin
[root@localhost ~]# source /etc/profile  //让profile配置文件立即生效
```
### 查看新版本
```bash
[root@localhost ~]# git --version  //查看版本号，安装成功
git version 2.9.3
```
*注：安装前需要使用 `yum` 安装 `gettext-devel` 和 `autoconf`*

## CentOS7安装Python3.6
### 安装IUS软件源
```bash
#安装EPEL依赖
sudo yum install epel-release

#安装IUS软件源
sudo yum install https://centos7.iuscommunity.org/ius-release.rpm
```
### 安装Python3.6
```bash
sudo yum install python36u
```
安装Python3完成后的shell命令为python3.6，为了使用方便，创建一个到python3的符号链接。
```bash
sudo ln -s /bin/python3.6 /bin/python3
```
### 安装pip3
安装完成python36u并没有安装pip，安装pip。
```bash
sudo yum install python36u-pip
```
安装pip完成后的shell命令为pip3.6，为了使用方便，创建一个到pip3的符号链接。
```bash
sudo ln -s /bin/pip3.6 /bin/pip3
```

## Centos7安装MongoDB
### 配置yum
用 vim 创建文件：
```shell
vi /etc/yum.repos.d/mongodb-org-3.4.repo
```
填入内容：
```
[mongodb-org-3.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.4/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.4.asc
```
### 安装MongoDB
通过以下命令安装：
```shell
yum install -y mongodb-org
```
安装过程中出现验证错误的情况，则取消 `gpgcheck` 的验证。修改 `mongodb-org-3.4.repo` 文件后重新安装：
```
[mongodb-org-3.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.4/x86_64/
gpgcheck=0
enabled=1
```
### 相关文件位置
> 安装完后配置文件在：`/etc/mongod.conf`
> 数据文件在：`/var/lib/mongo`
> 日志文件在：`/var/log/mongodb`

### 启动服务
可以使用相关命令来启动或者关闭 MongoDB:
```shell
// 启动 
systemctl start mongod.service

// 检查是否启动
systemctl status mongod.service

// 关闭 
systemctl stop mongod.service

// 重启服务
systemctl restart mongod.service

// 设置开机启动
systemctl enable mongod.service
```
### 访问权限控制
MongoDB服务默认只绑定在本机IP上，即只有本机才能访问MongoDB，我们可以修改访问权限控制让外网也能访问。
修改配置文件 `/etc/mongod.conf` 将其中的 `bindip:127.0.0.1` 注释即可。

## CentOS7安装Redis
### 更改yum源
备份你的原镜像文件，保证出错后可以恢复：
```shell
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
```
将CentOS的yum源更换为国内的阿里云源，下载新的 `CentOS-Base.repo` 到 `/etc/yum.repos.d/`：
```shell
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```
### 安装Redis
```shell
yum install redis
```
### 设置Redis开机启动
```shell
systemctl enable redis.service
```
### 设置Redis密码
打开文件 `/etc/redis.conf`，找到其中的 `# requirepass foobared`，去掉前面的 `#`，并把 `foobared` 改成你的密码。

## CentOS7下Elastic Stack环境配置
### 安装jdk1.8
* 查看yum库中的java安装包：`yum -y list java*`
* 安装需要的jdk版本的所有java程序：`yum -y install java-1.8.0-openjdk*`

此时输入 `java -version`，显示如下则安装成功。
```shell
openjdk version "1.8.0_171"
OpenJDK Runtime Environment (build 1.8.0_171-b10)
OpenJDK 64-Bit Server VM (build 25.171-b10, mixed mode)
```
### 安装Elasticsearch
*Elasticsearch官方下载地址：https://www.elastic.co/downloads/elasticsearch*

#### 在官网下载Elasticsearch的rpm包
```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.3.0.rpm
```
#### 安装rpm包
```
rpm -ivh elasticsearch-6.3.0.rpm
```
#### 开启Elasticsearch服务
```
service elasticsearch start
```
#### 测试Elasticsearch是否安装成功
```
curl http://localhost:9200
```
如果出现以下内容则说明安装成功。
```
[root@VM_0_11_centos software]# curl http://localhost:9200
{
  "name" : "QhrY0bW",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "BWJyzf8QSyujU8X4OouYbw",
  "version" : {
    "number" : "6.3.0",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "424e937",
    "build_date" : "2018-06-11T23:38:03.357887Z",
    "build_snapshot" : false,
    "lucene_version" : "7.3.1",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

### 安装Kibana
*Elasticsearch官方下载地址：https://www.elastic.co/cn/downloads/kibana*

#### 在官网下载Kibana的rpm包
```
wget https://artifacts.elastic.co/downloads/kibana/kibana-6.3.0-x86_64.rpm
```
#### 安装rpm包
```
rpm -ivh kibana-6.3.0-x86_64.rpm
```
#### 开启Kibana服务
```
service kibana start
```
#### 测试Kibana是否安装成功
```
curl http://localhost:5601
```

## CentOS7下Nginx安装及配置

### 配置yum源
新建 `/etc/yum.repos.d/nginx.repo` 文件，并在其中写入以下代码：
```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/mainline/centos/7/$basearch/
gpgcheck=0
enabled=1
```
### 安装Nginx
```
yum install nginx
```
### 测试Nginx安装是否成功
输入 `nginx -v`，显示如下则安装成功。
```
nginx version: nginx/1.15.0
```
### 启动Nginx
```
systemctl restart nginx.service
```
---
layout: post
title: vps基础环境搭建
categories: Development
description: aliyun ECS environment
keywords: CentOS 7, java, maven, nginx, git, node
---

根据实际vps搭建情况逐渐更新，因为是vps，资源紧张，基于docker安装有点没必要，直接裸装

## 阿里云服务器基础环境准备
### 检查基础环境变量
#### 登录服务器后，检查基础配置。
阿里云的机器默认做了很多优化，省了很多事。
```Bash
su
hostnamectl
yum update && yum upgrade
# 检查locale是否为en_US.UTF-8
locale
# 检查open files
ulimit -a
# 关于其他检查比如防火墙的设置，禁止root用户远程登录，只允许公钥登录等属于基础配置，不进行赘述。
yum -y groupinstall "Development Tools"
```
### 设置Java等环境
#### 下载安装
```Bash
cd /usr/local
# 去https://jdk.java.net/下载JDK并解压，我这里使用的阿里的开源jdk,对应的openJDK8，更新的可以去自行下载安装
wget "https://github.com/alibaba/dragonwell8/releases/download/8.0-preview/Alibaba_Dragonwell8_Linux_x64_8.0-preview.tar.gz"
tar xzf Alibaba_Dragonwell8_Linux_x64_8.0-preview.tar.gz 
ln -s j2sdk-image jdk
rm Alibaba_Dragonwell8_Linux_x64_8.0-preview.tar.gz
```
#### 配置环境变量
```Bash
touch /etc/profile.d/java.sh
vi /etc/profile.d/java.sh
```
`java.sh`内容
```Bash
#set java environment
JAVA_HOME=/usr/local/jdk
CLASSPATH=.:$JAVA_HOME/lib
PATH=$JAVA_HOME/bin:$PATH
export JAVA_HOME CLASSPATH PATH
```
```Bash
source /etc/profile
```
### 设置Maven环境
#### 下载安装
```Bash
cd /usr/local
wget "http://mirror.bit.edu.cn/apache/maven/maven-3/3.6.1/binaries/apache-maven-3.6.1-bin.tar.gz"
tar xzf apache-maven-3.6.1-bin.tar.gz
ln -s apache-maven-3.6.1 apache-maven
rm apache-maven-3.6.1-bin.tar.gz
```
#### 配置环境变量
```Bash
touch /etc/profile.d/maven.sh
vi /etc/profile.d/maven.sh
```
`maven.sh`内容
```Bash
#set maven environment
M2_HOME=/usr/local/apache-maven
PATH=$PATH:$M2_HOME/bin
export M2_HOME PATH
```
```Bash
source /etc/profile
```
### 安装nginx
#### 下载安装
```Bash
# 如果对本机器openssl的版本不满意，可以去官网下载源码自行编译安装
yum -y install zlib zlib-devel openssl openssl-devel pcre pcre-devel pcre-lib
cd /usr/local/src
wget -c https://github.com/nginx/nginx/archive/release-1.16.0.tar.gz -O nginx-release-1.16.0.tar.gz
tar xzf nginx-release-1.16.0.tar.gz
cd nginx-release-1.16.0
groupadd nginx
useradd nginx -g nginx -s /sbin/nologin -M
./configure --user=nginx --group=nginx --prefix=/usr/local/nginx --sbin-path=/usr/sbin/nginx --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/lock/subsys/nginx --with-http_gzip_static_module --with-http_realip_module --with-http_stub_status_module --with-http_ssl_module --with-http_addition_module --with-stream --with-stream_ssl_module --with-http_v2_module --with-threads
make
make install # 安装完成后 nginx 启动，查看有无错误消息
```
#### 包装成Systemd服务
```Bash
cd /usr/lib/systemd/system
# systemd的服务有system和user两种，这里的nginx随着机器启动，当然是系统服务
# 这里的配置基于我自己系统
wget https://raw.githubusercontent.com/yvonxiao/systemd-config/master/nginx.service
systemctl enable nginx # 可用systemctl is-enabled nginx.service来查看nginx服务
```
### 安装git
阿里云的CentOS 7自带的git很旧，需要自行编译安装
```Bash
yum install -y curl-devel zlib zlib-devel asciidoc xmlto perl perl-devel perl-CPAN cpio expat-devel gettext-devel autoconf tk perl-ExtUtils-MakeMaker
yum -y remove git
cd /usr/local/src
wget https://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.16.tar.gz
tar xzf libiconv-1.16.tar.gz
cd libiconv-1.16
./configure --prefix=/usr/local
make
make install
libtool --finish /usr/local/lib
ln -s /usr/local/lib/libiconv.so.2 /usr/lib/libiconv.so.2
ldconfig
cd /usr/local
wget "https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.21.0.tar.gz"
tar xzf git-2.21.0.tar.gz
cd git-2.21.0
./configure --prefix=/usr/local/git
make all doc
make install install-doc install-html
ln -s /usr/local/lib/libcharset.so.1 /usr/lib/libcharset.so.1
ldconfig
touch /etc/profile.d/git.sh
vi /etc/profile.d/git.sh
```
`git.sh`内容
```Bash
#set git environment
export PATH=$PATH:/usr/local/git/bin
```
```Bash
source /etc/profile
```
Continue...
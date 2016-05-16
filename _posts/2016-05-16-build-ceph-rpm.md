---
layout: post
title: build ceph RPM from source code
date: 2016-05-16 21:39:40
categories: ceph
tag: ceph
excerpt: 本文介绍如何从ceph 源码编译ceph RPM
---

## 前言
本文分成两个部分，第一部分是如何从ceph的源码出发，build出ceph RPM。 第二部分介绍如何搭建Jenkins，创建jobs，通过Jenkins来build ceph RPM


## build RPM from ceph code

ceph官网 [build ceph](http://docs.ceph.com/docs/master/install/build-ceph/)已经介绍了如何build ceph，基本够用。我们简单介绍下RHEL7下 RPM的build 情况

1. 安装依赖

  ceph提供了 ./install-dep.sh，可以用它来安装依赖

2. 下拉代码

  注意，ceph有一些submodule，下拉的时候，需要将子模块也下拉下来。

  ```
  git clone --recursive https://github.com/ceph/ceph.git
  ```
  
  注意，ceph有很多的分支，要checkout到你希望编译的分支上。
  
3. 将源码压缩成bz2格式

  ```
  ./autogen.sh 
  ./configure 
  make dist-bzip2
  ```
  
  执行完毕后，会在代码顶层目录看到 ceph-0.87.2.tar.bz2这种类型的源码压缩包
4. 准备rpmbuild 目录

 ```
 mkdir ~/rpmbuild/
 mkdir ~/rpmbuild/BUILD
 mkdir ~/rpmbuild/BUILDROOT
 mkdir ~/rpmbuild/RPMS
 mkdir ~/rpmbuild/SOURCES
 mkdir ~/rpmbuild/SPECS
 mkdir ~/rpmbuild/SRPMS
 ```


5. 准备构建文件

 ```
  cp ceph/ceph-0.87.2.tar.bz2  ~/rpmbuild/SOURCES/
  cp ceph/rpm/init-ceph.in-fedora.patch ~/rpmbuild/SOURCES/
  cp ceph/ceph.spec ~/rpmbuild/SPECS
  
 ```
  
6. build

  ```
  rpmbuild -ba rpmbuild/SPECS/ceph.spec
  ```

build结束之后，会在~/rpmbuild/RPMS/x86_64/下看到编译出来的RPMS


## install jenkins in RHEL7


* 安装依赖

```
yum install java-1.8.0-openjdk-headless dejavu-sans-fonts fontconfig
```

* 安装jenkins repo

```
cd /etc/yum.repos.d/
curl -O http://pkg.jenkins-ci.org/redhat-stable/jenkins.repo
rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key
```

* 安装jenkins

```
yum install jenkins
```

* 修改jenkins home目录,将家目录设置成/home/jenkins/

usermod -m -d /home/jenkins jenkins

vim  /etc/sysconfig/jenkins 

![](/assets/LINUX/jenkins_home.png)

* 设置开机自启动和启动jenkins 服务

```
systemctl enable jenkins.service
systemctl start jenkins.service 
```

* 设置firewalld

```
firewall-cmd --zone=public --add-port=8080/tcp --permanent
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --reload
firewall-cmd --list-all
```

* 通过 http://ip:8080来访问Jenkins服务。


## 部署jenkins job来编译ceph

有了编译ceph RPM的方法，又有了jenkins，可以在jenkins上创建job来编译ceph。

创建一个自由风格的项目，设置好git repo的地址和branch，jenkins就可以帮助我们下拉最新的代码，在Execute shell设置好要执行的命令，就可以将编译自动话。不再赘述。




---
layout: post
title: 在一个Linux下使用两个github账号
date: 2016-03-07 13:12:40
categories: ceph
---

前言
-----
有时候Linux下会有多个github的账号，昨天我新申请一个github账号的时候，发现添加ssh-key到github时，有报错信息。：

```
    public_key.verification_failure – imac - f0:25:97:f1:56:39:88:15:a1:72:0c:2d:a8:07:82:59
```

 后来想到，同一个Linux多个github，可能会有问题，所以搜索了下。下面把解决步骤记录下来。


解决步骤
--------

1 用ssh-keygen生成一个新的密钥：


```
➜  ~ ssh-keygen -t rsa -C "beanli.coder@gmail.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/home/manu/.ssh/id_rsa): /home/manu/.ssh/**id_rsa_me**    
➜  ~ 
```

2 ssh会读取id_rsa，为了让SSH能够识别该新增key，需要将其添加入SSH agent

```
ssh-add ~/.ssh/id_rsa_second
```

单独执行该命令，可能给会有如下错误：

```
Could not open a connection to your authentication agent.
```

```
➜  ~ eval `ssh-agent -s`
Agent pid 4582
➜  ~ ssh-add  /home/manu/.ssh/id_rsa_me
Identity added: /home/manu/.ssh/id_rsa_me (/home/manu/.ssh/id_rsa_me)
```

3 修改~/.ssh/config文件

```
	➜  ~ cat ~/.ssh/config
	Host github.com
	HostName github.com
	PreferredAuthentications publickey
	IdentityFile ~/.ssh/id_rsa
	
	Host **me.github.com**
	HostName github.com
	PreferredAuthentications publickey
	IdentityFile ~/.ssh/id_rsa_me 

```

注意，此处id_rsa_me密钥对应的Host为me.github.com，后面git clone 或者 git remote add origin需要使用
git@me.github.com这种地址，详情见第五步

4 将生成的新的密钥文件放入github的账号中

```
	➜  BLOG cat ~/.ssh/id_rsa_me.pub 
```

5 clone github上的repo到本地

```

	➜  BLOG git clone  git@me.github.com:Bean-Li/Bean-Li.github.io.git
	Cloning into 'Bean-Li.github.io'...
	remote: Counting objects: 77, done.
	remote: Compressing objects: 100% (60/60), done.
	remote: Total 77 (delta 23), reused 60 (delta 7), pack-reused 0
	Receiving objects: 100% (77/77), 65.22 KiB | 26 KiB/s, done.
	Resolving deltas: 100% (23/23), done.
	➜  BLOG

``` 

6 对于当前repo设置email 和name 

```
    git config  user.email "xxxx@xx.com"
    git config  user.name "xxxx"
```

从此之后可以正常地修改和使用第二个账号，而第一个账号不受影响

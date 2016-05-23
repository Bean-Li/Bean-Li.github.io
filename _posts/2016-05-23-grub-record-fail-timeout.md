---
layout: post
title:  异常启动后，不要停在grub页面的方法
date: 2016-05-23 10:26:40
categories: linux
tag: linux
excerpt: 如果上次启动失败或者进入了Recovery mode，下次启动会停留在grub选择页面，等待用户选择。
---

## 前言
机器在客户的机房中，当然最好是要有IPMI，但是IPMI并不是总是available的，重启的时候可能会卡在Grub menu那个页面，等待用户选择，但是有没有IPMI，无法选择，这就变成了死循环，必须打电话给客户解决，这带来了很多的不必要的麻烦。

根据[Ubuntu官方文档](https://help.ubuntu.com/community/Grub2)，有两种情况可能会进入Grub menu选择页面

1. 上一次启动失败了
2. 用户上一次选择了恢复模式

无论哪一种情况，就会停在Grub Menu，等待用户输入，这种情况下，对于自己的机房或有IPMI的情况，不是什么大事，但是如果机房在客户机房并且没有IPMI，这无疑增加了很多的跨公司的沟通成本。无论发生什么事情，不要无限等待就非常必要了。


## 解决方法



第一步：/etc/default/grub中，可以添加一个GRUB\_RECORDFAIL_TIMEOUT参数，该参数：

* GRUB\_RECORDFAIL_TIMEOUT ＝ -1 并不倒计时数秒，GRUB menu会出现
* GRUB\_RECORDFAIL_TIMEOUT ＝ 0  grub menu不会出现
* GRUB\_RECORDFAIL_TIMEOUT >= 1  grub menu 会展现 设置的秒数

我们将GRUB_RECORDFAIL_TIMEOUT等待时间设置为6秒。

如果仅仅修改/etc/default/grub然后 update-grub进程可能会起不到效果，同时需要修改 /etc/grub.d/00_header

第二步：修改/etc/grub.d/00_header

在/etc/grub.d/00_header下可能有如下内容：

```
232 make_timeout ()
233 {
234     cat << EOF
235 if [ "\${recordfail}" = 1 ]; then
236   set timeout=-1
237 else
238   set timeout=${2}
239 fi
240 EOF
241 }
```
将236行的set timeout=-1 改成：

```
  set timeout=6
```

注意，我查看最新的Ubuntu 12.04版本对应的1.99-21ubuntu3.19，

```
ii  grub2-common                     1.99-21ubuntu3.19                   GRand Unified Bootloader (common files for version 2)
```
发现00_header的内容已经发生了变化：

```
232 make_timeout ()
233 {
234     cat << EOF
235 if [ "\${recordfail}" = 1 ] ; then
236   set timeout=${GRUB_RECORDFAIL_TIMEOUT:-30}
237 else
238 EOF
```

从逻辑上分析，如果设置了GRUB\_RECORDFAIL\_TIMEOUT，那么timeout即为GRUB\_RECORDFAIL_TIMEOUT的值，否则为30秒。
这个改进是为了修复[Bug 1443735] Re: recordfail false positive causes
headless servers to hang on boot by default

```

Version: 1.99-21ubuntu3.18	2015-07-08 05:07:08 UTC
  grub2 (1.99-21ubuntu3.18) precise; urgency=medium

  * Do not hang headless servers indefinitely on boot after edge case power 
    failure timing (LP: #1443735). Instead, time out after 30 seconds and boot 
    anyway, including on non-headless systems.

 -- Robie Basak Tue, 19 May 2015 12:22:34 +0100
```

如果你的GRUB比这个版本要新，引入了这个修复，那么第二步就不需要了，直接设置GRUB\_RECORDFAIL_TIMEOUT即可执行第三步了。

第三部：备份/etc/boot/grub.cfg 并执行 update-grub

执行完毕后，不妨diff比较下区别。

![](/assets/LINUX/diff_grub_cfg.jpg)


## 脚本

下面是一段python 代码，当然do_cmd是我们内部封装的函数。关键是其中的shell命令部分。

```
def patch_grub_conf():
    default_grub_conf="/etc/default/grub"
    grub_00_header="/etc/grub.d/00_header"

    do_cmd("grep -q '^GRUB_RECORDFAIL_TIMEOUT=' {0} && " \
           "sed -i 's/^GRUB_RECORDFAIL_TIMEOUT=.*/GRUB_RECORDFAIL_TIMEOUT=6/' {0} || " \
           "sed -i '$ a\GRUB_RECORDFAIL_TIMEOUT=6' {0}".format(default_grub_conf))
    do_cmd("sed -i '236s/timeout=.*/timeout=6/' {}".format(grub_00_header))
    do_cmd("update-grub")
```


## 结束语
其实如果升级到1.99-21ubuntu3.18 及以上版本，就不会出现这个无限制等待的问题，最简单的方法当然是升级grub2-common。
如果升级觉得风险大的话，可以考虑patch grub的配置文件。



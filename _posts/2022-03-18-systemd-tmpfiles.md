---
layout: post
title: systemd tmpfiles相关的服务
date: 2022-03-18 10:29
categories: Linux
tag: systmed
excerpt: systemd tmpfiles相关的机制
---
# 前言

前些日子在排查一个httpd session的问题，发现虚机尽管重启了，但是Firefox浏览器并不logout，机器重启后，Firefox的发送的请求，httpd后台依然会处理。

一般来讲，httpd请求会为每一个client维护一个session，session相关的信息存放位置

* CentOS：/tmp/systemd-private-5257097ec71448b0aac566695b533a84-httpd.service-dMlTTI/sessions 这种类似的目录

```bash
CentOS 版本：
---------------
[root@node-a tmp]# tree systemd-private-5257097ec71448b0aac566695b533a84-httpd.service-dMlTTI
systemd-private-5257097ec71448b0aac566695b533a84-httpd.service-dMlTTI
└── tmp
    └── sessions
        └── 6d474c39c1606792c92f72973c0f135dc07b6682
2 directories, 1 file
```

注意为什么CentOS版本的session存放在一个systemd-priviate-开头的奇怪目录下。原因是写在httpd的systemd启动脚本中：

```bash
[root@node-a system]# cat httpd.service 
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
# We want systemd to give httpd some time to finish gracefully, but still want
# it to kill httpd after TimeoutStopSec if something went wrong during the
# graceful stop. Normally, Systemd sends SIGTERM signal right after the
# ExecStop, which would kill httpd. We are sending useless SIGCONT here to give
# httpd time to finish.
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

注意上面的PrivateTmp=true，这个选项是一个systemd选项，表示该服务会有一个独立的/tmp目录作为自己的/tmp目录，对于httpd而言：

```bash
drwx------  3 root root    4096 Mar 18 13:22 systemd-private-5257097ec71448b0aac566695b533a84-httpd.service-dMlTTI
drwx------  3 root root    4096 Mar 18 12:44 systemd-private-5257097ec71448b0aac566695b533a84-ntpd.service-YDlnOF
```

很多服务都有类似的行为，比如上面的ntpd的service，毫不意外，ntpd的service脚本中也有：

```bash
[Unit]
Description=Network Time Service
After=syslog.target ntpdate.service sntp.service

[Service]
Type=forking
EnvironmentFile=-/etc/sysconfig/ntpd
ExecStart=/usr/sbin/ntpd -u ntp:ntp $OPTIONS
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

很奇怪的现象是，CentOS的session文件，有以下行为模式：

* reboot -f时，sessoin并不会被清理 。reboot -f，然后重建集群，发现很多很奇怪的请求到来，并且被后端正常处理。
* reboot的时候，本次的session文件会被清理，但是历史垃圾的session信息不会被清理。


# systemd-tmpfiles 相关的服务

Linux操作系统，会有一些临时存放文件的区域，最典型的就应该是/tmp/目录下，该目录存放的文件，一般为临时存放区，不重要，可损失的文件，正式因为这个区域不那么严肃，所以很多进程或者人，会堆放一些文件在该目录下，如果没有任何服务负责管理，很可能会累积非常多的垃圾文件。

Linux的systemd提供了tmpfiles相关的服务：

```bash
[root@node-a system]# systemctl status systemd-tmpfiles-*
● systemd-tmpfiles-clean.service - Cleanup of Temporary Directories
   Loaded: loaded (/usr/lib/systemd/system/systemd-tmpfiles-clean.service; static; vendor preset: disabled)
   Active: inactive (dead) since Fri 2022-03-18 13:00:01 CST; 1h 14min ago
     Docs: man:tmpfiles.d(5)
           man:systemd-tmpfiles(8)
  Process: 12340 ExecStart=/usr/bin/systemd-tmpfiles --clean (code=exited, status=0/SUCCESS)
 Main PID: 12340 (code=exited, status=0/SUCCESS)

Mar 18 13:00:01 node-a systemd[1]: Starting Cleanup of Temporary Directories...
Mar 18 13:00:01 node-a systemd[1]: Started Cleanup of Temporary Directories.

● systemd-tmpfiles-clean.timer - Daily Cleanup of Temporary Directories
   Loaded: loaded (/usr/lib/systemd/system/systemd-tmpfiles-clean.timer; static; vendor preset: disabled)
   Active: active (waiting) since Fri 2022-03-18 12:44:40 CST; 1h 29min ago
     Docs: man:tmpfiles.d(5)
           man:systemd-tmpfiles(8)

Mar 18 12:44:40 node-a systemd[1]: Started Daily Cleanup of Temporary Directories.

● systemd-tmpfiles-setup-dev.service - Create Static Device Nodes in /dev
   Loaded: loaded (/usr/lib/systemd/system/systemd-tmpfiles-setup-dev.service; static; vendor preset: disabled)
   Active: active (exited) since Fri 2022-03-18 12:44:35 CST; 1h 29min ago
     Docs: man:tmpfiles.d(5)
           man:systemd-tmpfiles(8)
  Process: 556 ExecStart=/usr/bin/systemd-tmpfiles --prefix=/dev --create --boot (code=exited, status=0/SUCCESS)
 Main PID: 556 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/systemd-tmpfiles-setup-dev.service

Mar 18 12:44:35 node-a systemd[1]: Starting Create Static Device Nodes in /dev...
Mar 18 12:44:35 node-a systemd[1]: Started Create Static Device Nodes in /dev.

● systemd-tmpfiles-setup.service - Create Volatile Files and Directories
   Loaded: loaded (/usr/lib/systemd/system/systemd-tmpfiles-setup.service; static; vendor preset: disabled)
   Active: active (exited) since Fri 2022-03-18 12:44:38 CST; 1h 29min ago
     Docs: man:tmpfiles.d(5)
           man:systemd-tmpfiles(8)
  Process: 805 ExecStart=/usr/bin/systemd-tmpfiles --create --remove --boot --exclude-prefix=/dev (code=exited, status=0/SUCCESS)
 Main PID: 805 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/systemd-tmpfiles-setup.service

Mar 18 12:44:38 node-a systemd[1]: Starting Create Volatile Files and Directories...
Mar 18 12:44:38 node-a systemd[1]: Started Create Volatile Files and Directories.
[root@node-a system]# vim systemd-tmpfiles-clean.service 
```

清理工作一般分成两类:

* 开机启动时的清理
  * systemd-tmpfiles-setup.service 
  * systemd-tmpfiles-setup-dev.service
* Linux正常运行期间的清理
  * systemd-tmpfiles-clean.timer 

开机清理，这个是比较容易理解，开机的时候，可能所有服务都没有启动，上一轮机器运行阶段，可能产生了不少垃圾文件，或者没来得及清理的文件（比如异常掉电或者reboot -f），需要对具体的目录做一些清理动作。

另外一个就是长时间运行状态下，也要有服务负责清理，因为Linux服务器一般不喜欢关机，我们也有很多机器线上运行时间1000天以上，完全指望开机清理，可能势必垃圾文件堆积成山。

我们一起看下sytemd-tmpfiles-setup.service配置文件的内容：

```bash
[Unit]
Description=Create Volatile Files and Directories
Documentation=man:tmpfiles.d(5) man:systemd-tmpfiles(8)
DefaultDependencies=no
Conflicts=shutdown.target
After=systemd-readahead-collect.service systemd-readahead-replay.service local-fs.target systemd-sysusers.service
Before=sysinit.target shutdown.target
RefuseManualStop=yes

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/systemd-tmpfiles --create --remove --boot --exclude-prefix=/dev

```

该文件可以看到，主要是调用一个名称为/usr/bin/systemd-tmpfiles的可执行文件，传了一些参数。sytemd-tmpfiles有一些参数值得我们注意：

* create：

  * ```bash
    If this option is passed, all files and directories marked with f, F, w, d, D, v, p, L, c, b, m in the configuration files are created
               or written to. Files and directories marked with z, Z, t, T, a, and A have their ownership, access mode and security labels set.
    ```
    
* remove 

  * ```bash
    If this option is passed, the contents of directories marked with D or R, and files or directories themselves marked with r or R are removed.
    ```

* clean 

  * ```bash
    If this option is passed, all files and directories with an age parameter configured will be cleaned up.
    ```

* boot 

  * ```bash
    Also execute lines with an exclamation mark. (带感叹号的行， --boot才会执行)
    ```

boot最好理解，正常运行期间和重启刚开机是又分别的，有些目录，开机阶段可以粗暴删除，但是正常运行期间不能太粗暴，因为你可能没有办法确定，该文件或者目录是否还在被使用。因此后面配置文件会有区分的手段，剧透一下就是 action后面加感叹号，就是只有boot阶段才应该执行，正常运行期间，本条配置可以忽略。

# systemd-tmpfiles 的可配置性

我们讲了很多了，但是到底如何告知systemd那些文件可以被清理，可被清理的文件又遵循什么样的策略呢？这个地方就到了配置文件部分了。

配置文件的名称必须符合 package.conf` 或 `package-part.conf格式。 当需要明确的将某部分(part)配置提取出来，以方便用户专门针对这部分进行修改的时候， 应该使用第二种命名格式。

对于不同目录下的同名配置文件，仅以优先级最高的目录中的那一个为准。具体说来就是： `/etc/tmpfiles.d` 的优先级最高、 `/run/tmpfiles.d` 的优先级居中、 `/usr/lib/tmpfiles.d` 的优先级最低。 软件包应该将自带的配置文件安装在 `/usr/lib/tmpfiles.d` 目录中， 而 `/etc/tmpfiles.d` 目录仅供系统管理员使用。 所有的配置文件，无论其位于哪个目录中，都统一按照文件名的字典顺序处理。 如果在多个配置文件中设置了同一个路径(文件或目录)，那么仅以文件名最靠前(字典顺序)的那一个为准， 其他针对同一个路径的配置项将会作为警告信息记录到错误日志中。 如果有两行的路径互为前后缀，那么始终是先创建前缀行、再创建后缀行， 如果还需要删除，那么顺序正好相反，始终是先删除后缀行、再删除前缀行。 所有带有shell风格通配符的行，都在所有不带通配符的行之后处理。如果有多个操作符应用于同一个文件(例如 ACL, xattr, 文件属性调整)，那么将始终按固定的顺序操作。除上述清空之外，对于其他情况， 文件与目录总是按照它们在配置文件中出现的顺序处理。

```bash
[root@node-a system]# cd /usr/lib/tmpfiles.d/
[root@node-a tmpfiles.d]# ll
total 132
-rw-r--r--. 1 root root   29 Mar 16 04:33 ceph-common.conf
-rw-r--r--. 1 root root   35 Apr  1  2020 cryptsetup.conf
-rw-r--r--. 1 root root   26 Mar 16 05:02 ctdb.conf
-rw-r--r--. 1 root root   67 Dec 11 08:32 elasticsearch.conf
-rw-r--r--. 1 root root  464 Nov 17  2020 etc.conf
-rw-r--r--. 1 root root   77 Nov 16  2020 httpd.conf
-rw-r--r--. 1 root root   39 Nov 17  2020 initscripts.conf
-rw-r--r--. 1 root root   75 Dec 15  2020 iscsi.conf
-rw-r--r--. 1 root root 1181 Nov 17  2020 legacy.conf
-rw-r--r--. 1 root root   34 Apr  1  2020 libselinux.conf
-r--r--r--. 1 root root   61 Dec 16  2020 lvm2.conf
-rw-r--r--. 1 root root   34 Oct  1  2020 mdadm.conf
-rw-r--r--. 1 root root   35 Jan 22  2021 net-snmp.conf
-rw-r--r--. 1 root root   27 Sep 30  2020 nscd.conf
-rw-r--r--. 1 root root  104 Sep 30  2020 nss-pam-ldapd.conf
-rw-r--r--. 1 root root   85 Oct  1  2020 openldap.conf
-rw-r--r--. 1 root root  110 Apr  1  2020 pam.conf
-rw-r--r--. 1 root root   15 Apr 10  2017 proftpd.conf
-rw-r--r--. 1 root root   16 Nov 17  2020 python.conf
-rw-r--r--. 1 root root   42 Nov 17  2020 resource-agents.conf
-rw-r--r--. 1 root root   87 Apr  1  2020 rpcbind.conf
-rw-r--r--. 1 root root   22 Oct  1  2020 rpm.conf
-rw-r--r--. 1 root root   60 Mar 16 05:02 samba.conf
-rw-r--r--. 1 root root  228 Nov 17  2020 sap.conf
-rw-r--r--. 1 root root   72 Oct  1  2020 screen.conf
-rw-r--r--. 1 root root  137 Nov 17  2020 selinux-policy.conf
-rw-r--r--. 1 root root  305 Jan 27  2021 sudo.conf
-rw-r--r--. 1 root root 1662 Nov 17  2020 systemd.conf
-rw-r--r--. 1 root root  496 Nov 17  2020 systemd-nologin.conf
-rw-r--r--. 1 root root  638 Nov 17  2020 tmp.conf
-rw-r--r--. 1 root root   56 Mar 22  2019 tuned.conf
-rw-r--r--. 1 root root  563 Nov 17  2020 var.conf
-rw-r--r--. 1 root root  623 Nov 17  2020 x11.conf
[root@node-a tmpfiles.d]# 
[root@node-a tmpfiles.d]# cd /etc/tmpfiles.d/
[root@node-a tmpfiles.d]# ll
total 0
[root@node-a tmpfiles.d]# 
```

主要的配置文件都被存放在/etc/tmpfiles.d和 /usr/lib/tmpfiles.d/下。其中/usr/lib/tmpfiles.d下可以看到有形形色色的配置文件。Ubuntu版本的产品比较神器，可以自动清理/tmp/sessions下的文件，甚至整个/tmp/sessions都被删除，我们看下配置文件：

```bash
root@CVM01:/usr/lib/tmpfiles.d# cat tmp.conf 
D /tmp 1777 root root -
#q /var/tmp 1777 root root 30d

# Exclude namespace mountpoints created with PrivateTmp=yes
x /tmp/systemd-private-%b-*
X /tmp/systemd-private-%b-*/tmp
x /var/tmp/systemd-private-%b-*
X /var/tmp/systemd-private-%b-*/tmp

```

这里除了我们比较熟悉的路径意外，有写D x X之类的奇奇怪怪的字母，这些何意？

# systemd-tmpfiles 配置语法

`tmpfiles.d` 配置文件定义了一套临时文件管理机制： 创建 文件、目录、管道、设备节点， 调整访问模式、所有者、属性、限额、内容， 删除过期文件。 主要用于管理易变的临时文件与目录，例如 `/run`, `/tmp`, `/var/tmp`, `/sys`, `/proc`, 以及 `/var` 下面的某些目录

配置文件的格式是每行对应一个路径，包含如下字段： 类型, 路径, 权限, 属主, 属组, 寿命, 参数

```
#Type Path        Mode User Group Age Argument
d     /run/user   0755 root root  10d -
L     /tmp/foobar -    -    -     -   /dev/null
```

字段值可以用引号界定，并可以包含C风格的转义字符

然后就到了奇奇怪怪的字母部分了，因为它支持的Type太多了，我就一一罗列的，否则也记不住，我重点介绍常用的，如果需要了解细节的，可以阅读 [金步国的文章](http://www.jinbuguo.com/systemd/tmpfiles.d.html)。

* d   创建指定的目录并赋于指定的User/Group与权限。如果指定的目录已经存在，那么仅调整User/Group与权限。 如果指定了"寿命"字段，那么该目录中的内容将遵守基于时间的清理策略。
* D  与 `d` 类似， 但是如果使用了 `--remove` 选项，那么将会清空目录中的所有内容。

注意此处， 我们看下Ubuntu的tmp.conf配置文件里面的一行：

```
D /tmp 1777 root root -
```

这一行毁天灭地，如果有印象的话，systemd-tmpfiles-setup.service执行的是：

```
ExecStart=/usr/bin/systemd-tmpfiles --create --remove --boot --exclude-prefix=/dev
```

这会将/tmp/下的所有东西一扫而空。但是Ubuntu版本有些人会说，没有，启动之后/tmp/目录下还有文件。这些文件是重启之后创建出来的。因此我们就明白了为什么Ubuntu下，/tmp/sessions会被清理，因为实时上不仅它会被清理，整个/tmp/都会被清理。


* x   根据"寿命"字段清理过期文件时， 忽略指定的路径及该路径下的所有内容。 可以在"路径"字段中使用shell风格的通配符。 注意， 这个保护措施对 `r` 与 `R` 无效。
* R  在根据"寿命"字段清理过期文件时， 仅忽略指定的路径自身而不包括该路径下的其他内容。 可以在"路径"字段中使用shell风格的通配符。 注意， 这个保护措施对 `r` 与 `R` 无效。

这两个是额外的保护，即按照时间线来清理的时候，免死金牌，不被清理。但是这个保护措施对后面提到的r和R无效

* r  若指定的文件或目录存在， 则删除它。 不可用于非空目录。 可以在"路径"字段中使用shell风格的通配符。 不追踪软连接。
* R 若指定的文件或目录存在，则递归的删除它。 可用于非空目录。 可以在"路径"字段中使用shell风格的通配符。 不追踪软连接。

这两个删除指令，R可递归，r不可。支持通配符。我们欣赏下Ubuntu 20.04版本里的systemd-tmp.conf 

```bash
# Exclude namespace mountpoints created with PrivateTmp=yes
x /tmp/systemd-private-%b-*
X /tmp/systemd-private-%b-*/tmp
x /var/tmp/systemd-private-%b-*
X /var/tmp/systemd-private-%b-*/tmp

# Remove top-level private temporary directories on each boot
R! /tmp/systemd-private-*
R! /var/tmp/systemd-private-*
```

上面赦免不用说了，正常清理过程中，不要按照时间清理/tmp/systemd-private/打头的文件。但是后面R！表示机器重启期间，要将上述文件彻底删除，防止留下垃圾。

使用了感叹号(!)标记的行，仅可在系统启动过程中执行， 而不能用于运行中的系统(会破坏系统的正常运行)。 未使用感叹号(!)标记的行， 可以在任意时间安全的执行(例如升级软件包的时候)。 **systemd-tmpfiles** 仅在明确使用了 `--boot` 选项的时候 才会执行使用了感叹号(!)标记的行。

# 分析CentOS版本的行为

```bash
# Clear tmp directories separately, to make them easier to override
v /tmp 1777 root root 10d
v /var/tmp 1777 root root 30d

# Exclude namespace mountpoints created with PrivateTmp=yes
x /tmp/systemd-private-%b-*
X /tmp/systemd-private-%b-*/tmp
x /var/tmp/systemd-private-%b-*
X /var/tmp/systemd-private-%b-*/tmp
```

啥也不说了。根本就不清理 /tmp/system-private-文件。


# 如何调试systemd-tmpfiles

```
env SYSTEMD_LOG_LEVEL=debug systemd-tmpfiles --clean
```

这种方式比较详细，会把中间的判断过程也打印出来，如果清理行为不理解的话，可以用这个方法，看下为什么行为和自己理解的不一样。

```
[root@node-a tmpfiles.d]# env SYSTEMD_LOG_LEVEL=debug systemd-tmpfiles --clean
Reading config file "/usr/lib/tmpfiles.d/ceph-common.conf".
Reading config file "/usr/lib/tmpfiles.d/cryptsetup.conf".
Reading config file "/usr/lib/tmpfiles.d/ctdb.conf".
Reading config file "/usr/lib/tmpfiles.d/elasticsearch.conf".
Reading config file "/usr/lib/tmpfiles.d/etc.conf".
Reading config file "/usr/lib/tmpfiles.d/httpd.conf".
Reading config file "/usr/lib/tmpfiles.d/initscripts.conf".
Reading config file "/usr/lib/tmpfiles.d/iscsi.conf".
Reading config file "/run/tmpfiles.d/kmod.conf".
Ignoring entry c! "/dev/fuse" because --boot is not specified.
Ignoring entry c! "/dev/cuse" because --boot is not specified.
Ignoring entry c! "/dev/btrfs-control" because --boot is not specified.
Ignoring entry c! "/dev/loop-control" because --boot is not specified.
Ignoring entry c! "/dev/net/tun" because --boot is not specified.
Ignoring entry c! "/dev/ppp" because --boot is not specified.
Ignoring entry c! "/dev/uinput" because --boot is not specified.
Ignoring entry c! "/dev/mapper/control" because --boot is not specified.
Ignoring entry c! "/dev/uhid" because --boot is not specified.
Ignoring entry c! "/dev/vfio/vfio" because --boot is not specified.
Ignoring entry c! "/dev/vhci" because --boot is not specified.
Ignoring entry c! "/dev/vhost-net" because --boot is not specified.
....
```

如果带上参数 --boot 和--remove，模拟的基本就是开机启动时的清理逻辑：

```
env SYSTEMD_LOG_LEVEL=debug systemd-tmpfiles --clean

```

# 参考文献：
*	[金步国的文章](http://www.jinbuguo.com/systemd/tmpfiles.d.html)
*  man systemd-tmpfiles

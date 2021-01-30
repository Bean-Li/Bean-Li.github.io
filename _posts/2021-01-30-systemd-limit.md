---
layout: post
title: systemd修改服务的资源使用
date: 2021-01-30 10:29
categories: linux
tag: systemd
excerpt: 本文介绍如何使用systemd启动脚本修改服务的资源限制
---

# 前言

有些时候，我们需要修改某些进程或者某个服务的资源，比如：

* 允许进程打开的最大文件数
* 修改线程的默认栈大小

Linux提供了ulimit 指令来查看和修改默认资源限制：

```bash
root@subhealth1:/lib/systemd# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 379704
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 4096
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 379704
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

对于systmed来讲，如果修改配置文件，限定或者修改资源的使用呢？

# systemd和ulimit的映射

systemd基本上可以修改所有资源的限制,它和ulimit的映射关系如下：

```
Directive        ulimit equivalent     Unit
LimitCPU=        ulimit -t             Seconds      
LimitFSIZE=      ulimit -f             Bytes
LimitDATA=       ulimit -d             Bytes
LimitSTACK=      ulimit -s             Bytes
LimitCORE=       ulimit -c             Bytes
LimitRSS=        ulimit -m             Bytes
LimitNOFILE=     ulimit -n             Number of File Descriptors 
LimitAS=         ulimit -v             Bytes
LimitNPROC=      ulimit -u             Number of Processes 
LimitMEMLOCK=    ulimit -l             Bytes
LimitLOCKS=      ulimit -x             Number of Locks 
LimitSIGPENDING= ulimit -i             Number of Queued Signals 
LimitMSGQUEUE=   ulimit -q             Bytes
LimitNICE=       ulimit -e             Nice Level 
LimitRTPRIO=     ulimit -r             Realtime Priority  
LimitRTTIME=     No equivalent
```



我们以OSD为例：

```bash
[Unit]
Description=Ceph object storage daemon osd.%i
After=network-online.target local-fs.target time-sync.target ceph-mon.target
Wants=network-online.target local-fs.target time-sync.target
PartOf=ceph-osd.target

[Service]
LimitNOFILE=1048576
LimitNPROC=1048576
EnvironmentFile=-/etc/default/ceph
Environment=CLUSTER=ceph
ExecStart=/usr/bin/ceph-osd -f --cluster ${CLUSTER} --id %i --setuser root --setgroup root
ExecStartPre=/usr/lib/ceph/ceph-osd-prestart.sh --cluster ${CLUSTER} --id %i
ExecReload=/bin/kill -HUP $MAINPID
ProtectHome=true
ProtectSystem=full
PrivateTmp=true
TasksMax=infinity
Restart=on-failure
StartLimitInterval=30min
StartLimitBurst=30
RestartSec=20s

[Install]
WantedBy=ceph-osd.target
```

注意，Linux默认允许打开的文件数非常有限，而ceph-osd可能需要打开更多的文件，所以通过

```
LimitNOFILE=1048576
```

将允许该进程打开的文件数放大到了1M。

如果我们需要修改默认stacksize，那么，我们需要增加如下行：

```bash
LimitSTACK=2097152
```

我们将ceph-osd的栈大小从默认的8M改成了2M。

修改完毕之后需要执行：

```bash
systemctl daemon-reload
```

然后重启相关的服务，即可生效。

# 检查修改是否生效

如果我们修改了相关的资源限制，如何查看是否生效呢？

```bash
cat /proc/[PID]/limits
```

我们以一个测试环境为例：

```bash
root@CVM01:/lib/systemd/system# pidof ceph-osd
978185 928119
root@CVM01:/lib/systemd/system# cat /proc/928119/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            2097152              2097152              bytes
Max core file size        0                    unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             1048576              1048576              processes
Max open files            1048576              1048576              files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       257494               257494               signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
root@CVM01:/lib/systemd/system#
```

我们可以看到我们修改的stacksize已经生效了，从默认的8M降低为2M。


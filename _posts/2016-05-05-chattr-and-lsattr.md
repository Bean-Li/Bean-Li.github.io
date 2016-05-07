---
layout: post
title: lsattr
date: 2016-05-07 14:44:40
categories: linux
tag: linux
excerpt: 本文介绍chattr和lsattr
---


## 前言

前面有篇文章介绍了EXT4_INODE_TOPDIR标志位可以让目录下的文件或目录遵循Spread out的分布策略，提到了chattr +T可以给目录打上该标志位，但是如何检查是否已经生效呢。

lsattr属于e2fsprogs这个package，它提供了查看ext4的标志的作用。

```
root@node186:~# which lsattr
/usr/bin/lsattr
root@node186:~# dpkg -S /usr/bin/lsattr
e2fsprogs: /usr/bin/lsattr
```


## lsattr ::T

在博文[EXT4 Inode 分配策略](https://bean-li.github.io/EXT4-Inode-orlov/)曾经提到，顶层目录需要spread out的策略，除此以外，/home目录下的多个目录，一般对应不同用户，而不同用户彼此的资料是无关的，而且一般用户A基本不会去访问用户B的资料，因此，这些/home/下的目录也应该采用spread out的策略，即应该给/home目录打上EXT4_INODE_TOPDIR标志味。

```
    chattr +T /home
```

如何确认是否生效呢？这时候，lsattr既可以上场了。

lsattr是chattr的好基友，chattr负责改变，lsattr负责查看改变是否生效。

```
root@node186:/# ls -alid */
2752513 drwxr-xr-x   2 root     root      4096 Apr 27 21:49 bin/
 917505 drwxr-xr-x   4 root     root      4096 Apr 27 22:05 boot/
4718593 drwxr-xr-x   3 root     root      4096 Apr 27 18:33 build/
2883585 drwxr-xr-x   9 root     root      4096 May  5 21:30 data/
   1025 drwxr-xr-x  16 root     root      4700 May  5 21:54 dev/
4194305 drwxr-xr-x 116 root     root     12288 May  7 11:44 etc/
 393217 drwxr-xr-x   2 root     root      4096 Apr 19  2012 home/
3276801 drwxr-xr-x   3 admin    admin     4096 Apr 27 20:11 hotfix/
 262145 drwxr-xr-x  19 root     root      4096 Apr 27 21:49 lib/
1835009 drwxr-xr-x   2 root     root      4096 Apr 27 21:48 lib64/
     11 drwx------   2 root     root     16384 Apr 27 18:27 lost+found/
2359297 drwxr-xr-x   4 root     root      4096 Apr 27 18:28 media/
5505025 drwxr-xr-x   3 root     root      4096 Apr 27 22:13 mnt/
 524289 drwxr-xr-x   2 memcache memcache  4096 Apr 27 18:36 nonexistent/
5767169 drwxr-xr-x   3 root     root      4096 Apr 27 18:34 opt/
      1 dr-xr-xr-x 691 root     root         0 May  5 21:27 proc/
5636097 drwx------   5 root     root      4096 May  6 22:10 root/
  13325 drwxr-xr-x  25 root     root      1340 May  5 21:54 run/
 786433 drwxr-xr-x   2 root     root     12288 Apr 27 21:49 sbin/
4980737 drwxr-xr-x   2 root     root      4096 Mar  6  2012 selinux/
 131073 drwxr-xr-x   2 root     root      4096 Apr 27 18:28 srv/
      1 dr-xr-xr-x  12 root     root         0 May  6 10:08 sys/
1966081 drwxrwxrwt  12 root     root     20480 May  7 12:17 tmp/
3407873 drwxr-xr-x  12 root     root      4096 Apr 27 21:49 usr/
1572865 drwxr-xr-x  14 root     root      4096 Apr 27 18:57 var/
2621441 drwxrwxrwx   8 root     root      4096 May  5 16:10 vol/
root@node186:/#
```

我们注意看，根目录下的目录，他们的inode是散开的，彼此并不毗邻，以/etc下属的目录，紧紧团结在etc对应的inode附近。

```
root@node186:/etc# ls -lid  */
4594859 drwxr-xr-x  2 root root   4096 May  4 21:45 ImageMagick/
4194767 drwxr-xr-x  7 root root   4096 May  4 21:45 X11/
4458830 drwxr-xr-x  3 root root   4096 Apr 27 18:30 acpi/
4194408 drwxr-xr-x  2 root root   4096 May  4 21:45 alternatives/
4328210 drwxr-xr-x  7 root root   4096 Apr 27 18:36 apache2/
4196904 drwxr-xr-x  3 root root   4096 Apr 27 18:30 apm/
4194589 drwxr-xr-x  8 root root   4096 Apr 27 18:34 apparmor.d/
4194586 drwxr-xr-x  3 root root   4096 Apr 27 18:30 apparmor/
4194412 drwxr-xr-x  6 root root   4096 Apr 27 22:05 apt/
4593180 drwxr-xr-x  3 root root   4096 Apr 27 18:36 avahi/
4194339 drwxr-xr-x  3 root root  12288 Apr 27 21:52 bash_completion.d/
4457995 drwxr-xr-x  3 root root   4096 Apr 27 18:30 ca-certificates/
4196895 drwxr-xr-x  2 root root   4096 Apr 27 18:30 calendar/
4328229 drwxr-xr-x  2 root root   4096 May  5 17:06 ceph/
4458454 drwxr-s---  2 root dip    4096 Apr 27 18:30 chatscripts/
4590030 drwxr-xr-x  2 root root   4096 Apr 27 18:36 cluster/
4194642 drwxr-xr-x  2 root root   4096 Apr 27 18:28 console-setup/
4197540 drwxr-xr-x  4 root root   4096 Apr 27 18:34 corosync/
4194593 drwxr-xr-x  2 root root   4096 Apr 27 18:37 cron.d/
4194410 drwxr-xr-x  2 root root   4096 Apr 27 21:52 cron.daily/
4194637 drwxr-xr-x  2 root root   4096 Apr 27 18:38 cron.hourly/
4194629 drwxr-xr-x  2 root root   4096 Apr 27 18:28 cron.monthly/
4194631 drwxr-xr-x  2 root root   4096 Apr 27 18:30 cron.weekly/

```

当我们执行chattr +T /home之后，我们掉用lsattr可以看到/home目录有了T标志位：

```
root@node186:/# lsattr /
-------------e- /build
-------------e- /lost+found
-------------e- /lib64
...
------------Te- /home
...
```

其中的T表示的即EXT4_INODE_TOPDIR标志位，/home下的各个目录之间，其inode应该散开，而不是靠拢。

```
root@node186:/home# for i in {alice,big,bean,chandler,bruce}; do useradd -m -d /home/${i} -s /bin/bash  ${i} ; done
root@node186:/home# ls -lid  */
 655361 drwxr-xr-x 2 alice    alice    4096 May  7 12:28 alice/
3014657 drwxr-xr-x 2 bean     bean     4096 May  7 12:28 bean/
2490369 drwxr-xr-x 2 big      big      4096 May  7 12:28 big/
1179649 drwxr-xr-x 2 bruce    bruce    4096 May  7 12:28 bruce/
5898241 drwxr-xr-x 2 chandler chandler 4096 May  7 12:28 chandler/
```

不难看出，/home/下的目录，设置了EXT4_INODE_TOPDIR标志后，采用散开的策略。

##lsattr :: I

前面曾经介绍过[dir_index属性](https://bean-li.github.io/EXT4_DIR_INDEX/)，当目录下的条目比较少时，一个block（默认4KB）能够存放下所有的条目时，采用线性方法存放条目。但是随着目录下的条目越来越多，一个4K页面已经存放不下的时候，EXT4会采用hash tree的方式存放。如何判断当前目录时采用了linear 还是 hash tree的方式存放目录的呢？

```
root@node186:/home# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0   1.8T  0 disk
|-sda1   8:1    0  30.5M  0 part
|-sda2   8:2    0 488.3M  0 part
|-sda3   8:3    0  93.1G  0 part /
|-sda4   8:4    0   128G  0 part [SWAP]
`-sda5   8:5    0   1.6T  0 part
sdb      8:16   0   7.3T  0 disk
`-sdb1   8:17   0   7.3T  0 part /data/osd.0
sdc      8:32   0   7.3T  0 disk
`-sdc1   8:33   0   7.3T  0 part /data/osd.1
sdd      8:48   0 372.6G  0 disk
|-sdd1   8:49   0    32G  0 part
|-sdd2   8:50   0    32G  0 part
`-sdd3   8:51   0   300G  0 part
sde      8:64   0 372.6G  0 disk
|-sde1   8:65   0    32G  0 part
|-sde2   8:66   0    32G  0 part
`-sde3   8:67   0   300G  0 part
root@node186:/home# debugfs  /dev/sda3  -R "htree home/"
debugfs 1.42 (29-Nov-2011)
htree: Not a hash-indexed directory

root@node186:/home# ll
total 28
drwxr-xr-x  7 root     root     4096 May  7 12:28 ./
drwxr-xr-x 28 root     root     4096 May  7 12:57 ../
drwxr-xr-x  2 alice    alice    4096 May  7 12:28 alice/
drwxr-xr-x  2 bean     bean     4096 May  7 12:28 bean/
drwxr-xr-x  2 big      big      4096 May  7 12:28 big/
drwxr-xr-x  2 bruce    bruce    4096 May  7 12:28 bruce/
drwxr-xr-x  2 chandler chandler 4096 May  7 12:28 chandler/
```

由于/home目录下只有有限的几个条目，因此，采用的线性方式存放条目。

因为/etc/下有大量的文件和目录，条目无法放到一个4K的页面内，因此采用了hash tree的存放方式。

```
Root node dump:
         Reserved zero: 0
         Hash Version: 1
         Info length: 8
         Indirect levels: 0
         Flags: 0
Number of entries (count): 2
Number of entries (limit): 508
Entry #0: Hash 0x00000000, block 1
Entry #1: Hash 0x910f4882, block 2

Entry #0: Hash 0x00000000, block 1
4194307 0x53f8172e-16270e57 (12) udev
4194313 0x56ec8672-286d2982 (16) network
4194315 0x78e7527e-9eb18f0a (16) hostname
4194316 0x8c256a20-4f8db1f7 (16) hosts
4194339 0x08b263c8-0c74b184 (28) bash_completion.d
4194364 0x33519812-fb5ca2df (20) insserv.conf
4194365 0x22688064-88f21fcf (24) insserv.conf.d
4194369 0x4fdcadc4-20fcf113 (16) pam.conf
4194372 0x12594e24-cf6cfd53 (16) security
4194388 0x4025667a-f9d9fdec (12) dpkg
...

Entry #1: Hash 0x910f4882, block 2
4194468 0x910f4882-d0ec24e6 (20) python2.7
4194480 0x92081afa-13971cad (16) rc1.d
4199540 0x930dac62-9218048a (16) gtk-2.0
4196904 0x9310f70c-14b73caf (12) apm   4328376 0x96ed9e30-85d36869 (12) zfs
4458830 0x97f9261a-26a813d1 (12) acpi   4461046 0x9bef8b50-decaebf7 (12) ha.d
4194408 0x9cc61de2-7c65bbb0 (20) alternatives
4194595 0x9da4066a-6d3822bd (16) crontab
4194759 0xa44f0998-a53301df (12) vim
4197728 0xa572c360-e1ce1f1c (24) cachefilesd.conf
...
4196870 0xffdce130-59ce6dbb (2228) bash_completion
---------------------
(END)
```


但是debugfs这种方式查看目录使用了那种方式，太重了，耗时太久。毕竟我们只是需要看看下目录是否带上了EXT4_INODE_INDEX标志：

```
root@node186:~# debugfs /dev/sda3
debugfs 1.42 (29-Nov-2011)
debugfs:  stat etc
Inode: 4194305   Type: directory    Mode:  0755   Flags: 0x81000
Generation: 674607911    Version: 0x00000000:000011a1
User:     0   Group:     0   Size: 12288
File ACL: 0    Directory ACL: 0
Links: 116   Blockcount: 24
Fragment:  Address: 0    Number: 0    Size: 0
 ctime: 0x572d6ee3:ecde5944 -- Sat May  7 12:28:19 2016
 atime: 0x572d7845:6c1a3798 -- Sat May  7 13:08:21 2016
 mtime: 0x572d6ee3:ecde5944 -- Sat May  7 12:28:19 2016
crtime: 0x57209426:30c49e98 -- Wed Apr 27 18:27:50 2016
Size of extra inode fields: 28
EXTENTS:
(0):16785440, (1-2):16786501-16786502
(END)
```

注意，Flags的0x81000中的1，偏移量时12，表示该目录使用了hash tree的方式组织目录内容。

```
EXT4_INODE_INDEX    = 12,   /* hash-indexed directory */ 
```

出了上述比较重的方法意外，lsattr也可以查看,如果目录有I标志，表示采用了hash tree的方式组织了目录。比如下面的/sbin/和/etc

```
root@node186:~# lsattr /
...
----------I--e- /sbin
-------------e- /mnt
----------I--e- /etc
...
```

很有意思的是，一旦目录采用了hash tree的组织形式，如果后来目录下的条目（文件或目录）被删除，哪怕所有内容可以存放到一个4KB的页面中，EXT4也不会从hash tree的组织形式回退到线性的最执行时。

同时I标志，并不可以通过chattr设置，完全是EXT4文件系统的行为：

```
    The 'I' attribute is used by the htree code to indicate that a directory 
    is being indexed using hashed trees.  It may not be set or  reset  using
    chattr(1), although it can be displayed by lsattr(1).
```


## lsattr ::e

e的含义是说file使用extents来定位文件内容在磁盘上的位置。

```
    The 'e' attribute indicates that the file is using extents for mapping 
    the blocks on disk.  It may not be removed using chattr(1).
```

下面这个图是经典的文件存放位置定位的图片：

![](/assets/EXT4/ext3_tripple.gif)

事实上，这种算法早就已经过时了。为什么过时，其实比较明显，如果数据存放在磁盘上的位置明明是连续的（很多个连续的4K的块），可是还是不得不根据映射，一级一级地找一个4K的块，这太坑了。

EXT4开始引入了一种extent tree的技术，来解决这种低效的文件内容定位问题。


![](/assets/EXT4/ext4_extent_tree.png)

这种技术的本质和三级指针的技术大同小异，不过它考虑到了很可能有相当多的块是连续的，与其浪费指针指向一个块，不如设计数据结构，指向这多个连续的块。

```
/*
 * This is the extent on-disk structure.
 * It's used at the bottom of the tree.
 */
struct ext4_extent {
	__le32	ee_block;	/* first logical block extent covers */
	__le16	ee_len;		/* number of blocks covered by extent */
	__le16	ee_start_hi;	/* high 16 bits of physical block */
	__le32	ee_start_lo;	/* low 32 bits of physical block */
};
```

根据上述数据结构，不难看出对于叶子节点是如何描述文件对应位置的多个连续块的磁盘所在位置。

extent tree并非本文主旨，因此，不打算展开。可以相见，EXT4所有的文件也好都会有e标志，因为EXT4默认使用extent tree来组织文件的内容。




## 结尾

lsattr还有很多其他的标志，其实很多我也并不熟悉，但是并不妨碍我们将该部分内容列在此处。希望后面有需要的人可以研究之后分享下这些标志的含义。


```

    The letters `acdeijstuACDST' select the new attributes for the files: 
    append only (a), compressed (c), no dump (d), extent format (e),  
    immutable (i),  data  journalling  (j),  secure deletion (s), no 
    tail-merging (t), undeletable (u), no atime updates (A), no copy on 
    write (C), synchronous directory updates (D), synchronous updates (S), 
    and top of directory hierarchy (T).

    The following attributes are read-only, and may be listed by lsattr(1) 
    but not modified by chattr: huge file (h), compression error (E), 
    indexed directory (I), compression raw access (X), and compressed dirty 
    file (Z).
```



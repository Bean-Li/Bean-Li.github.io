---
layout: post
title: EXT4 的 dir_index 特性
date: 2016-03-25 14:26:40
categories: linux
tag: EXT4, linux,ceph
excerpt: EXT4的dir_index feature，是为了加快目录查找
---

引言
-----
这个点子是前辈博文，[EXT4 Optimization for Filestore I/O optimization](https://www.aevoo.fr/2016/02/14/ceph-ext4-optimisation-for-filestore/)  中提出来的点子，原理也并不复杂，我看这篇博文的基础，顺便看了内核的EXT4的一些资料和代码，收获颇丰。

感谢前辈，光荣属于前辈。

EXT4的dir_index功能
---------
严格来说，dir_index并非EXT4时引入的，EXT3就已经有这个功能了。
对于EXT4而言该feature是默认打开的。

```
root@node2:/data/osd.3# tune2fs -l /dev/sdc2  |grep features
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype needs_recovery extent flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize

```
从/etc/mke2fs.conf中也可以看出，这个是格式化文件系统的默认选项：

```
[defaults]
    base_features = sparse_super,filetype,resize_inode,dir_index,ext_attr
    default_mntopts = acl,user_xattr
    enable_periodic_fsck = 0
    blocksize = 4096
    inode_size = 256
    inode_ratio = 16384

[fs_types]
    ext3 = {
        features = has_journal
    }
    ext4 = {
        features = has_journal,extent,huge_file,flex_bg,uninit_bg,dir_nlink,extra_isize
        auto_64-bit_support = 1
        inode_size = 256
    }

```

这个功能到底是什么作用呢？其实这个功能开发的目的是为了应对单个目录下有海量的子文件或子文件夹。
目录本质也是一种文件，但是这种目录文件的内容，是其下文件或目录的inode与文件名信息。

```
inode1  file1
inode2  file2
inode3  file3

```
当然了，这只是示意，目录中记录的条目并非这么简单了，但也并不复杂，结构体如下：

```
1914 struct ext4_dir_entry_2 {                                                                                                                               
1915     __le32  inode;          /* Inode number */
1916     __le16  rec_len;        /* Directory entry length */
1917     __u8    name_len;       /* Name length */
1918     __u8    file_type;
1919     char    name[EXT4_NAME_LEN];    /* File name */
1920 };

```
可以看出，该条目就五个成员变量，其中name是变长的。其中file_type就是目录中对应文件所属的类型，就是经典的7类型，不多说。


自EXT2以来，在目录下查找目录下的文件，就是一个线性扫描的过程。约三四年前我写过一篇解析EXT2目录内容的文章，
但是目录下文件数目比较少的情况下，这种方法还是不错的。但是如果一个目录下有几万几十万个条目，这个方法就比较慢了。 原因在于线性扫描，而且，1个block（4096字节），基本只能放下几十~200个条目，一旦需要几十几百个block，那么为了获取子文件的inode，这个DISK IO的消耗是不能忍受的。因此开发了dir_index的功能。

换了一个思路，就是hash tree的方式来存放entry，而不是线性往后追加。



注意，并不说打开了dir_index功能，所有的目录都一律使用hash tree的方式存储。当目录下的条目并不多的时候，并不采用hash tree，而是采用线性目录。

问题来了，如何判断一个目录是否采用了hash tree呢？


    root@node2:/data/osd.3/bean_test/7/8/9# debugfs /dev/sdc2
    debugfs:  stat bean_test/7/8/9
    Inode: 1573041   Type: directory    Mode:  0755   Flags: 0x81000
    Generation: 1854113010    Version: 0x00000000:0000003d
    User:     0   Group:     0   Size: 12288
    File ACL: 0    Directory ACL: 0
    Links: 2   Blockcount: 24
    Fragment:  Address: 0    Number: 0    Size: 0
     ctime: 0x56f4e050:4de990ac -- Fri Mar 25 14:53:04 2016
     atime: 0x56f4d6a6:3149d714 -- Fri Mar 25 14:11:50 2016
     mtime: 0x56f4e050:4de990ac -- Fri Mar 25 14:53:04 2016
    crtime: 0x56f4d6a6:3149d714 -- Fri Mar 25 14:11:50 2016
    Size of extra inode fields: 28
    EXTENTS:
    (0):6299856, (1-2):6301284-6301285

看到Flags的值为0x81000，这里的1，即hash index标志位，如果该位置1 表示采用了hash tree的方式组织目录，否则就是线性方式。

    EXT4_INODE_INDEX    = 12,   /* hash-indexed directory */ 

或者采用这种方式：


```

    root@node2:~# debugfs /dev/sdc2  -R "htree bean_test/7/8/9"

    Root node dump:
             Reserved zero: 0
             Hash Version: 1
             Info length: 8
             Indirect levels: 0
             Flags: 0
    Number of entries (count): 2
    Number of entries (limit): 508
    Entry #0: **Hash** 0x00000000, block 1
    Entry #1: **Hash** 0x78dd80dc, block 2
    
    Entry #0: **Hash** 0x00000000, block 1


    1576441 0x6052ada6-6d9549a5 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_22   
    1577404 0x74726c1a-28130637 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_29   
    1579737 0x74a5bbd6-b1650489 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_16   
    1580026 0x70833b1a-0badd977 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_0   
    1582614 0x1fb33024-7dc128d5 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_4   
    1583380 0x5cb40f50-fe4ebf40 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_7   
    1583455 0x003490e6-9250c69c (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_14   
    1584946 0x01fe133e-1ea34c2c (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_11   

    1610001 0x64a9e024-6132bf17 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_34   
    1611053 0x0b8cfa12-77f58c7a (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_30   
    1611308 0x103ebd64-95e9f6f6 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_37   
    1615540 0x41d8c886-bcf98137 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_52   
    1615626 0x1cf710e4-79891985 (2192) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_57   
    Entry #1: Hash 0x78dd80dc, block 2
    
    1600703 0x78dd80dc-b1a50749 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_59   
    1614083 0x802cf708-11025989 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_55   
    1615338 0x84d8b090-57dacb1b (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_49   
    1589688 0x9087818a-41b39780 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_2   
    1584612 0x9d060434-3aa55540 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_21   
  
  
    1598473 0xec267b26-37eb05f4 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_48   
    1613924 0xf8c296e2-99357e31 (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_43   
    1587880 0xfddbae16-312d91ec (68) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_1   
    1616194 0xef7c75a8-52978227 (2056) 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_32   
    ---------------------
    (END)

```

linear directory or hash tree
-----------------------------
内核如何决策，合适使用线性目录。内核采用的算法是，如果1个block（默认为4096字节）已经存放不下了已有的entry，那么就要转向hash tree了。

这部分逻辑在fs/ext4/namei.c中的ext4_add_entry中。如果内核发现一个block采用线性存放的方法，已经无法放下所有的entry，就会通过make_indexed_dir函数将所有条目转换成hash tree的方式存放，同时会给dir对应的inode设置EXT4_INODE_INDEX标志，指示目前目录已经采用hash tree的方式存放条目。

注意当你一个目录有几千几万个文件时，hash tree是有优势的，但是如果我只有100个文件，采用hash tree反倒有点浪费性能，因为线性目录只需要读取该block，所有的信息都在其中，但是hash tree 不得不多读取一个block，如图所示。

![](/assert/EXT4/htree_directory.png)

ceph的情况比较有意思，文件名基本是比较规整的，命名有一套规范，有的文件名有38个字节，当然有个文件名有58个字节。如果尝试让所有的文件都位于一个4K的block之中，只能用比较恶劣的情况来决定存放文件的数目，我采用58个字节。

测试发现存放57个之后，一个block就放不下了，就不得不采用hash tree来查找了。

```

    Inode: 1310731   Type: directory    Mode:  0755   Flags: 0x80000
    Generation: 1854237480    Version: 0x00000000:0000003a
    User:     0   Group:     0   Size: 4096
    File ACL: 0    Directory ACL: 0
    Links: 2   Blockcount: 8
    Fragment:  Address: 0    Number: 0    Size: 0
     ctime: 0x56f50da3:13cd12cc -- Fri Mar 25 18:06:27 2016
     atime: 0x56f50da2:51ca3784 -- Fri Mar 25 18:06:26 2016
     mtime: 0x56f50da3:13cd12cc -- Fri Mar 25 18:06:27 2016
    crtime: 0x56f50da2:51ca3784 -- Fri Mar 25 18:06:26 2016
    Size of extra inode fields: 28
    EXTENTS:
    (0):5251282
    (END)

    root@node2:~# dd if=/dev/sdc2 of=raw_dir_data bs=4K skip=5251282 count=1

    root@node2:~# ./parse_linear_dir raw_dir_data 
     offset | inode number | rec_len | name_len | file_type | name
    ======================================================================
         0:      1310731          12           1           2 .
        12:      1310725          12           2           2 ..
        24:      1310766          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_54
        92:      1310772          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_19
       160:      1310781          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_28
       228:      1310800          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_11
       296:      1310801          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_32
       364:      1310811          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_56
       432:      1310820          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_38
       500:      1310828          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_34
       568:      1310837          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_46
       636:      1310842          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_17
       704:      1310846          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_45

	....

      3424:      1311500          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_35
      3492:      1311511          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_23
      3560:      1311533          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_50
      3628:      1311565          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_40
      3696:      1311610          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_37
      3764:      1311622          68          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_24
      3832:      1311623         264          58           1 100010d56eb.00000000__head_29C681AC__1_ffffffffffffffff_52
    
```   
 
 ceph上使用该特性
-------------

ceph 的fileStore实现为了防止单个目录下有太多的文件，才用了动态分裂的方法，当某个目录下文件数目太多，就将文件分摊到多个文件夹.

```
current/19.2d_head/DIR_D/DIR_2/DIR_8/DIR_A:
total 16180
drwxrwxrwx  2 root root    4096 Mar 25 22:06 ./
drwxrwxrwx 18 root root   32768 Mar 25 15:50 ../
-rw-r--r--  1 root root  380928 Mar 25 20:59 10000f0df00.00000000__head_22C0A82D__13
-rw-r--r--  1 root root  180224 Mar 25 21:24 10000f88280.00000000__head_77F2A82D__13
-rw-r--r--  1 root root 2826240 Mar 25 19:26 10000fb5d28.00000000__head_01DAA82D__13
-rw-r--r--  1 root root   20480 Mar 25 06:15 1000120b117.00000000__head_5D03A82D__13
-rw-r--r--  1 root root  270336 Mar 25 06:16 1000120badb.00000000__head_5197A82D__13
-rw-r--r--  1 root root  147456 Mar 25 06:25 100012100b3.00000000__head_A432A82D__13
-rw-r--r--  1 root root 1798144 Mar 25 09:35 10001265799.00000000__head_3079A82D__13
-rw-r--r--  1 root root 2039808 Mar 25 09:47 1000126b801.00000000__head_D955A82D__13
-rw-r--r--  1 root root 2404352 Mar 25 10:08 10001274dea.00000000__head_5A52A82D__13
-rw-r--r--  1 root root  221184 Mar 25 10:33 10001280303.00000000__head_4539A82D__13
-rw-r--r--  1 root root 1429504 Mar 25 10:54 100012899bf.00000000__head_A4CBA82D__13
-rw-r--r--  1 root root  778240 Mar 25 11:44 100012a028b.00000000__head_9431A82D__13
-rw-r--r--  1 root root   20480 Mar 25 14:28 100012e9269.00000000__head_D550A82D__13
-rw-r--r--  1 root root  819200 Mar 25 15:11 100012fbb07.00000000__head_636BA82D__13
-rw-r--r--  1 root root  126976 Mar 25 18:21 1000134f568.00000000__head_D018A82D__13
-rw-r--r--  1 root root    5120 Mar 25 18:53 1000135e124.00000000__head_B1FBA82D__13
-rw-r--r--  1 root root  290816 Mar 25 20:25 1000138608c.00000000__head_61D1A82D__13
-rw-r--r--  1 root root    5120 Mar 25 20:47 1000138f118.00000000__head_FD4EA82D__13
-rw-r--r--  1 root root  237568 Mar 25 20:54 100013928a9.00000000__head_5762A82D__13
-rw-r--r--  1 root root   11264 Mar 25 21:04 10001396b18.00000000__head_0CF1A82D__13
-rw-r--r--  1 root root 2396160 Mar 25 21:29 100013a18a8.00000000__head_2767A82D__13
-rw-r--r--  1 root root   20480 Mar 25 22:06 100013b2124.00000000__head_FD89A82D__13
```

这些object的名字都是有规则的：其中head后面的A4CBA82D是hash值，根据object id计算出来的hash值，我们看到，这一排hash值最后一个字母都是D。

随着某个PG下的object 也来越多，超过了一个阈值，ceph就会根据hash值的数字，将这些object分到不同的目录里，所有hash值最后一个字母是D的object就被划分到了19.2d_head下DIR_D目录下。

但是随着数据越写越多，19.2d_head/DIR_D下的文件也越来越多，就按照倒数第二个字母，再次分裂成多个子目录。因此hash值倒数第二个字母是2的都被划分到了19.2d_head/DIR_D/DIR_2

依次类推。

这个阈值是怎么算的？算法见ceph源码的src/os/HashIndex.cc

```
bool HashIndex::must_split(const subdir_info_s &info) {
  return (info.hash_level < (unsigned)MAX_HASH_LEVEL &&
	  info.objs > ((unsigned)(abs(merge_threshold)) * 16 * split_multiplier));
			    
}

```

看到了，其中关键的两个控制选项为：

```
"filestore_split_multiple": "2"
"filestore_merge_threshold": "10"
```

因此，当单个目录下文件书超过20\*16\*2 ＝ 320个，就会分裂。

按照之前的分析，64个object并不保证一定会使用线性目录，因为会有文件名的大小为58，因此前言中文章才建议使用：

```

filestore_merge_threshold = 2
filestore_split_multiple = 1

```

真正效果如何，还需要用事实说话， 目前我并没有实际的测试数据。
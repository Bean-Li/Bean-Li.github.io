---
layout: post
title: EXT4是否存在FS error
date: 2017-01-04 15:56:40
categories: linux
tag: linux
excerpt: 如何快速判断EXT4是否存在FS error
---

# 前言
异常掉电情况下，如果硬件保护不周，可能导致文件系统损坏。如何快速的判断某个EXT4 fs是否存在错误？

当然e2fsck -p 是一个方法，但是这种方法有两个问题：

* 不得不umount文件系统
* 如果存在文件系统错误，e2fsck 会修复之

如果仅仅是简单的判断某个线上文件系统是否存在错误，有无轻量级的方法呢？
tune2fs 来帮忙


# 方法

```
tune2fs -l /dev/sdX
```
这个命令可以帮忙检测EXT4文件系统是否存在文件系统错误。

输出如下：

```
root@scalars04:/var/log# tune2fs -l /dev/mapper/3600d02310004a2152807e4a4208b3006-part2 
tune2fs 1.42 (29-Nov-2011)
Filesystem volume name:   <none>
Last mounted on:          /data/osd.66
Filesystem UUID:          0a6eb642-a9f3-4f76-9f2d-b6aaf12605b9
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr dir_index filetype needs_recovery extent 64bit flex_bg sparse_super huge_file uninit_bg dir_nlink extra_isize
Filesystem flags:         signed_directory_hash 
Default mount options:    user_xattr acl
Filesystem state:         clean with errors
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              615763968
Block count:              9852209915
Reserved block count:     0
Free blocks:              3533386093
Free inodes:              614654549
First block:              0
Block size:               4096
Fragment size:            4096
Blocks per group:         32768
Fragments per group:      32768
Inodes per group:         2048
Inode blocks per group:   128
Flex block group size:    16
Filesystem created:       Mon Jan 18 20:47:21 2016
Last mount time:          Tue Nov 29 14:53:52 2016
Last write time:          Fri Dec 16 03:35:36 2016
Mount count:              13
Maximum mount count:      -1
Last checked:             Mon Jan 18 20:47:21 2016
Check interval:           0 (<none>)
Lifetime writes:          27 TB
Reserved blocks uid:      0 (user root)
Reserved blocks gid:      0 (group root)
First inode:              11
Inode size:	          256
Required extra isize:     28
Desired extra isize:      28
Journal inode:            8
Default directory hash:   half_md4
Directory Hash Seed:      4166ec4b-4463-4c13-9395-23c1a7a0bc5b
Journal backup:           inode blocks
FS Error count:           233
First error time:         Tue May 24 10:29:21 2016
First error function:     ext4_mb_generate_buddy
First error line #:       756
First error inode #:      0
First error block #:      0
Last error time:          Fri Dec 16 03:35:36 2016
Last error function:      ext4_mb_generate_buddy
Last error line #:        757
Last error inode #:       0
Last error block #:       0
root@scalars04:/var/log# 

```

# 原理
tune2fs属于 e2fsprogs 这个deb package：

```
root@BEAN-3:~# which tune2fs
/sbin/tune2fs
root@BEAN-3:~# dpkg -S /sbin/tune2fs

e2fsprogs: /sbin/tune2fs
```

从代码上看：
```
void list_super (struct ext2_super_block * s)
{
	list_super2(s, stdout);
}


void list_super2(struct ext2_super_block * sb, FILE *f)
{
    ...
	if (sb->s_first_error_time) {
		tm = sb->s_first_error_time;
		fprintf(f, "First error time:         %s", ctime(&tm));
		memset(buf, 0, sizeof(buf));
		strncpy(buf, (char *)sb->s_first_error_func,
			sizeof(sb->s_first_error_func));
		fprintf(f, "First error function:     %s\n", buf);
		fprintf(f, "First error line #:       %u\n",
			sb->s_first_error_line);
		fprintf(f, "First error inode #:      %u\n",
			sb->s_first_error_ino);
		fprintf(f, "First error block #:      %llu\n",
			sb->s_first_error_block);
	}
	if (sb->s_last_error_time) {
		tm = sb->s_last_error_time;
		fprintf(f, "Last error time:          %s", ctime(&tm));
		memset(buf, 0, sizeof(buf));
		strncpy(buf, (char *)sb->s_last_error_func,
			sizeof(sb->s_last_error_func));
		fprintf(f, "Last error function:      %s\n", buf);
		fprintf(f, "Last error line #:        %u\n",
			sb->s_last_error_line);
		fprintf(f, "Last error inode #:       %u\n",
			sb->s_last_error_ino);
		fprintf(f, "Last error block #:       %llu\n",
			sb->s_last_error_block);
	}
	...
}
```
内核代码中fs/ext4/super.c中有__save_error_info 函数会将错误信息记录在superblock：

```
static void __save_error_info(struct super_block *sb, const char *func,
			    unsigned int line)
{
	struct ext4_super_block *es = EXT4_SB(sb)->s_es;

	EXT4_SB(sb)->s_mount_state |= EXT4_ERROR_FS;
	if (bdev_read_only(sb->s_bdev))
		return;
	es->s_state |= cpu_to_le16(EXT4_ERROR_FS);
	es->s_last_error_time = cpu_to_le32(get_seconds());
	strncpy(es->s_last_error_func, func, sizeof(es->s_last_error_func));
	es->s_last_error_line = cpu_to_le32(line);
	if (!es->s_first_error_time) {
		es->s_first_error_time = es->s_last_error_time;
		strncpy(es->s_first_error_func, func,
			sizeof(es->s_first_error_func));
		es->s_first_error_line = cpu_to_le32(line);
		es->s_first_error_ino = es->s_last_error_ino;
		es->s_first_error_block = es->s_last_error_block;
	}
	/*
	 * Start the daily error reporting function if it hasn't been
	 * started already
	 */
	if (!es->s_error_count)
		mod_timer(&EXT4_SB(sb)->s_err_report, jiffies + 24*60*60*HZ);
	le32_add_cpu(&es->s_error_count, 1);
}
```

会调用该函数的场景比较多，就不一一赘述了。

 
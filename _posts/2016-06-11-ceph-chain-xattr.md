---
layout: post
title: ceph internal 之 chain_xattr
date: 2016-05-29 14:57:40
categories: ceph-internal
tag: ceph-internal
excerpt: 本文学习下filestore下chain_xattr相关部分。
---

## 前言

ObjectStore是ceph OSD中最重要的一个概念，它封装了所有对底层存储的IO操作。

![](assets/ceph_internals/classObjectStore__inherit__graph.png)

对于底层存储而言，底层的基本元素不再是文件，而是Object。对于任何一种Object，最多有以下三个组成部分

* byte data： 对象的内容
* xattr ： 类似于文件的扩展属性信息，由一个或者多个key-value pair组成的集合
* omap ： 用途和xattr一样，只是存放的位置不再是存放在文件系统提供的xattr中，而是存放在leveldb或Rocksdb中


byte data 是对象的数据，自不必提。其中xattr和omap都是key value对的形式，用来描述对象的某条属性。但是为什么已经有了xattr，还要omap呢？ 

原因是xattr这种形式，必须依赖于本地文件系统的扩展属性方面的功能，即它利用本地文件系统中的文件的xattr来存取对象的属性信息。既然使用到了本地文件系统的xattr，必然受到本地文件系统的制约。有些本地文件系统xattr的长度有限制，比如EXT4,限制扩展属性的value大小不得超4K，XFS扩展属性的value最大不得超过64K.

注意，XFS的64K这个长度基本上是可以满足Ceph的要求，但是EXT4 的4K就略小了一些。因此，如果属性的值很大，超过4K，很难直接放到本地文件的扩展属性中。

由于这个限制，Ceph采用了两个办法：

第一个办法是特别长的的kv信息，直接存放到DBObjectMap中，即前面提到的omap。所谓DBObjectMap，就是利用LevelDB或RocksDB，存放和对象关联的key-value对。

另外一个方法是对象对应的本地文件的多个属性组成一个真正意义的属性，比如说键值是key，但是对应的value非常大，有10K无法存放到1个扩展属性中，就采用级连的方式

* key    :   [0，4K）
* key@1  :   [4K,8K)
* key@2  :   [8K,10K)

注意啊，上面这个例子仅仅是示意，不能将这个例子套用在EXT4上，因为EXT4文件系统支撑不了10K大小的扩展属性，多个扩展属性级连夜不行。
这就是ceph中的chain_xattr。本文重点介绍chain_xattr.


## Linux中的扩展属性

xattr，是Extended Attributes的含义，即文件扩展属性，简称EA。这是一种以key-value对形式将元数据和文件inode关联的技术。Linux从2.6起，开始支持EA。

提起EA，总是会和访问权限控制（ACL）相关联，事实上ACL的存取，和扩展属性本质是一样的。传统的访问控制，使用user group other三个层级来控制文件的访问权限，但是这种控制粒度太粗，不能灵活地细粒度地控制访问权限。本位重点并非ACL，所以我们回到扩展属性。


EA是有命名空间的概念，EA的命名格式为 namespace.name,其中namespace用来讲EA从功能上划分成截然不同的几类

* user
* trusted
* system
* security

上述4类中，system 目前用来支持访问控制列表ACL，security用来支持SELinux，而user和trusted是可以用于用户进程来设置。要存取 trusted EA，进程必须具有特权 CAP_SYS_ADMIN。

因此，user这个命名空间的EA，被Ceph大量使用, 我们看下如下函数：

```
static void get_attrname(const char *name, char *buf, int len)
{
  snprintf(buf, len, "user.ceph.%s", name);
}

bool parse_attrname(char **name)
{
  if (strncmp(*name, "user.ceph.", 10) == 0) {
    *name += 10;
    return true;
  }
  return false;
}
```

ceph将某个属性 name，命名为 user.ceph.$name ，设置到和对象对应的本地文件中。

当然了，除了object本身需要的元数据信息外，为了内部流程控制，也有一些额外的控制相关的扩展属性：

```
#define XATTR_SPILL_OUT_NAME "user.cephos.spill_out"
#define XATTR_NO_SPILL_OUT "0"
#define XATTR_SPILL_OUT "1"
```
比如这个，这个属性的含义是，是否所有存在扩展属性存放在omap中，如果该扩展属性的值是0，表示并没有属性存放到DBObjectMap中，如果是1，表示除了文件系统的EA外，还有部分属性信息存放在DBObjectMap中。


我们继续讨论Linux下的EA。user扩展属性，只能作用于目录和普通文件。除此外，VFS 对EA做了一些限制：

* EA 	名称的长度不得超过255 字节
* EA 值的长度不得超过64K

这个64K的限制位于内核的include/uapi/linux/limits.h

```
#ifndef _LINUX_LIMITS_H
#define _LINUX_LIMITS_H

#define NR_OPEN	        1024

#define NGROUPS_MAX    65536	/* supplemental group IDs are available */
#define ARG_MAX       131072	/* # bytes of args + environ for exec() */
#define LINK_MAX         127	/* # links a file may have */
#define MAX_CANON        255	/* size of the canonical input queue */
#define MAX_INPUT        255	/* size of the type-ahead buffer */
#define NAME_MAX         255	/* # chars in a file name */
#define PATH_MAX        4096	/* # chars in a path name including nul */
#define PIPE_BUF        4096	/* # bytes in atomic write to a pipe */
#define XATTR_NAME_MAX   255	/* # chars in an extended attribute name */
#define XATTR_SIZE_MAX 65536	/* size of an extended attribute value (64k) */
#define XATTR_LIST_MAX 65536	/* size of extended attribute namelist (64k) */

#define RTSIG_MAX	  32

#endif
```

事实上，XFS/ReiserFS都已经支持64K的属性值，但是很不幸，EXT4 并不支持这么大的属性值。

接下来，我们测试下ext4的xattr相关操作：

```
root@node-186:~/xattr_test# touch file1
root@node-186:~/xattr_test# getfattr file1
root@node-186:~/xattr_test#
root@node-186:~/xattr_test# setfattr -n user.key1 -v "value1" file1
root@node-186:~/xattr_test# getfattr -d  file1
# file: file1
user.key1="value1"

root@node-186:~/xattr_test# for((i=0;i<10;i++)); do setfattr -n user.key${i} -v "value${i}" file1 ; done
root@node-186:~/xattr_test# getfattr -d  file1
# file: file1
user.key0="value0"
user.key1="value1"
user.key2="value2"
user.key3="value3"
user.key4="value4"
user.key5="value5"
user.key6="value6"
user.key7="value7"
user.key8="value8"
user.key9="value9"



root@node-186:~/xattr_test# setfattr -x user.key4 file1
root@node-186:~/xattr_test# getfattr -d  file1
# file: file1
user.key0="value0"
user.key1="value1"
user.key2="value2"
user.key3="value3"
user.key5="value5"
user.key6="value6"
user.key7="value7"
user.key8="value8"
user.key9="value9"

root@node-186:~/xattr_test#

```

上面给出了扩展属性的新增，查看以及删除。


## xattr 的相关系统调用

上述简单介绍了扩展属性的一些内容，并且使用getfattr和setfattr两个tool测试了下EXT4对扩展属性的支持。

###创建和修改扩展属性

```
       #include <sys/types.h>
       #include <attr/xattr.h>

       int setxattr (const char *path, const char *name,
                       const void *value, size_t size, int flags);
       int lsetxattr (const char *path, const char *name,
                       const void *value, size_t size, int flags);
       int fsetxattr (int filedes, const char *name,
                       const void *value, size_t size, int flags);
```

如果扩展属性不存在，那么就创建，如果扩展属性存在，那么就修改。当然了，flags中可以提供标志位，更精准地控制API的行为：

* XATTR_CREATE
如果给定名称的EA已经存在，则失败
* XATTR_REPLACE
如果给定名称的EA存在，则失败


### 获取扩展属性的值

```
       #include <sys/types.h>
       #include <attr/xattr.h>

       ssize_t getxattr (const char *path, const char *name,
                            void *value, size_t size);
       ssize_t lgetxattr (const char *path, const char *name,
                            void *value, size_t size);
       ssize_t fgetxattr (int filedes, const char *name,
                            void *value, size_t size);
```

name用来置顶属性的key值，value为用户指定的缓冲区，size为缓冲区的大小。如果掉用成功，扩展属性的值被拷贝到用户指定的缓冲区，并且将拷贝的字节数返回。

如果找不到 名为name的属性，返回－1，并置错误码为ENODATA
如果用户的缓冲区太小，放不下属性的值，则返回－1，并置错误码为ERANGE


这里面有一个技巧，即将size指定为0，那么系统调用会忽略用户指定的缓冲区，而是将EA值的大小返回。这种使用方法可以有效地探查属性值的大小，从而提前得知需要分配多大的缓冲区来存放属性值。


### 删除某扩展属性

```
       #include <sys/types.h>
       #include <attr/xattr.h>

       int removexattr (const char *path, const char *name);
       int lremovexattr (const char *path, const char *name);
       int fremovexattr (int filedes, const char *name);

```

删除名为name的扩展属性，如果找不到该属性，返回－1，并置errno为ENODATA。

### 获取与文件相关的所有扩展属性的名称

```
       #include <sys/types.h>
       #include <attr/xattr.h>

       ssize_t listxattr (const char *path,
                            char *list, size_t size);
       ssize_t llistxattr (const char *path,
                            char *list, size_t size);
       ssize_t flistxattr (int filedes,
                            char *list, size_t size);
```

将扩展属性的名称，放入到list中，当然每一个属性的name，以"\0"为结尾，所有属性名称的总长度，作为返回值返回。

```
listLen = listxattr(path,list,XATTR_SIZE) ;

for(pos = 0 ; pos < listLen; pos += strlen(&list[pos])+1)
{
    cur_xattr = &list[pos] ;
}
```


和getxattr一样，如果size是0，系统调用会忽略 list这个buffer，返回存放所有EA列表所需缓冲区的大小。这个功能非常有用，获取之前，先探查，然后分配足够大的空间，然后调用listxattr来获取名称列表。

介绍了这么多，基本将Linux下扩展属性相关的系统调用介绍完毕。

Ceph因为其跨OS平台的需求，所以这部分代码位于 src/common/xattr.c，我们只需要关注Linux部分即可，基本就是本部分介绍的系统调用。


## chain_xattr

前面提到了，由于本地文件系统存在一些限制，可能属性值需要的空间非常大，，存放不下，ceph为了应对这种情况，采用了chain_xattr的方式来解决。

比如user.key是用户扩展属性的名称，该属性的值比较大，存放不下，用户可以将多个文件系统的扩展属性来表示一个逻辑意义上的扩展属性。

即如果需要多个键值才能存放下：

* user.key
* user.key@1
* user.key@2

这种方法中，有一个问题，即如果扩展属性中的原始name里面有@字符，怎么办，这种情况需要转义。
user.key@class 转义成user.key@@class

```
void get_raw_xattr_name(const char *name, int i, char *raw_name, int raw_len)
{
  int pos = 0;

  while (*name) {
    switch (*name) {
    case '@': /* escape it */
      pos += 2;
      assert (pos < raw_len - 1);
      *raw_name = '@';
      raw_name++;
      *raw_name = '@';
      break;
    default:
      pos++;
      assert(pos < raw_len - 1);
      *raw_name = *name;
      break;
    }
    name++;
    raw_name++;
  }

  if (!i) {
    *raw_name = '\0';
  } else {
    int r = snprintf(raw_name, raw_len - pos, "@%d", i);
    assert(r < raw_len - pos);
  }
}
```

这种级连的存在，就有一个文件，即用户的key到底有没有级连。

这个难题夜不难破解，存放的时候，按照固定大小分片存进本地文件系统的多个key值中。

```
template <bool skip_chain_cleanup=false, bool ensure_single_attr=false>
int chain_setxattr(
  const char *fn, const char *name, const void *val, size_t size)
{
  int i = 0, pos = 0;
  char raw_name[CHAIN_XATTR_MAX_NAME_LEN * 2 + 16];
  int ret = 0;
  size_t max_chunk_size =
    ensure_single_attr ? size : get_xattr_block_size(size);

  static_assert(
    !skip_chain_cleanup || ensure_single_attr,
    "skip_chain_cleanup must imply ensure_single_attr");

  do {
    size_t chunk_size = (size < max_chunk_size ? size : max_chunk_size);
    get_raw_xattr_name(name, i, raw_name, sizeof(raw_name));
    size -= chunk_size;

    int r = sys_setxattr(fn, raw_name, (char *)val + pos, chunk_size);
    if (r < 0) {
      ret = r;
      break;
    }
    pos  += chunk_size;
    ret = pos;
    i++;
    assert(size == 0 || !ensure_single_attr);
  } while (size);

  if (ret >= 0 && !skip_chain_cleanup) {
    int r;
    do {
      get_raw_xattr_name(name, i, raw_name, sizeof(raw_name));
      r = sys_removexattr(fn, raw_name);
      if (r < 0 && r != -ENODATA)
	ret = r;
      i++;
    } while (r != -ENODATA);
  }

  return ret;
}
```

注意其中的chunk_size，

```
#define CHAIN_XATTR_MAX_BLOCK_LEN 2048

/*
 * XFS will only inline xattrs < 255 bytes, so for xattrs that are
 * likely to fit in the inode, stripe over short xattrs.
 */
#define CHAIN_XATTR_SHORT_BLOCK_LEN 250
#define CHAIN_XATTR_SHORT_LEN_THRESHOLD 1000

int get_xattr_block_size(size_t size)
{
  if (size <= CHAIN_XATTR_SHORT_LEN_THRESHOLD)
    // this may fit in the inode; stripe over short attrs so that XFS
    // won't kick it out.
    return CHAIN_XATTR_SHORT_BLOCK_LEN;
  return CHAIN_XATTR_MAX_BLOCK_LEN;
}
```

如果EA的值，小于1K，那么分片的大小是250字节，否则，分片的大小是2048。

设置的时候，决定了文件系统层面的单个属性值最多为250或者2048，如果用户去取user.key的属性值的时候，取回来值正好为250或者正好是2048，那么就有很大的可能，该键值使用了级连（当然，也可能没有级连， 只是属性的值，恰好是250字节或者2048字节）。

我们来看，获取属性的值的方法，如果采用试探的方法，来判断是否多个文件系统层面的EA键值级连描述一个逻辑意义上的扩展属性。

```
int chain_getxattr(const char *fn, const char *name, void *val, size_t size)
{
  int i = 0, pos = 0;
  char raw_name[CHAIN_XATTR_MAX_NAME_LEN * 2 + 16];
  int ret = 0;
  int r;
  size_t chunk_size;

  if (!size)
    return getxattr_len(fn, name);

  do {
    chunk_size = size;
    
    //i的值来决定是级连扩展属性的的第几片
    get_raw_xattr_name(name, i, raw_name, sizeof(raw_name));

    r = sys_getxattr(fn, raw_name, (char *)val + pos, chunk_size);
    if (i && r == -ENODATA) {
      ret = pos;
      break;
    }
    if (r < 0) {
      ret = r;
      break;
    }

    if (r > 0) {
      pos += r;
      size -= r;
    }

    i++;
  } while (size && (r == CHAIN_XATTR_MAX_BLOCK_LEN ||
		    r == CHAIN_XATTR_SHORT_BLOCK_LEN)); //取回来的属性值恰好是250或者2048，那么很有可能是级连，需要探查下一个，即i++

  if (r >= 0) {
    ret = pos;
    /* is there another chunk? that can happen if the last read size span over
       exactly one block */
    if (chunk_size == CHAIN_XATTR_MAX_BLOCK_LEN ||
	chunk_size == CHAIN_XATTR_SHORT_BLOCK_LEN) {
      get_raw_xattr_name(name, i, raw_name, sizeof(raw_name));
      r = sys_getxattr(fn, raw_name, 0, 0);
      if (r > 0) { // there's another chunk.. the original buffer was too small
        ret = -ERANGE;
      }
    }
  }
  return ret;
}

int chain_getxattr_buf(const char *fn, const char *name, bufferptr *bp)
{
  size_t size = 1024; // Initial
  while (1) {
    bufferptr buf(size);
    int r = chain_getxattr(
      fn,
      name,
      buf.c_str(),
      size);
    if (r > 0) {
      buf.set_length(r);
      if (bp)
	bp->swap(buf);
      return r;
    } else if (r == 0) {
      return 0;
    } else {
      if (r == -ERANGE) {
	size *= 2;
      } else {
	return r;
      }
    }
  }
  assert(0 == "unreachable");
  return 0;
}
```

注意，有可能事先准备的缓冲区的大小不够，那么，从chain_getxattr_buf 可以看出，如果返回值是ERANGE，表示准备的空间不足以放下属性的值，
那么，缓冲区size放大一倍，然后重新获取属性的值。

如果Linux提供了getxattr都有探查属性值所需空间大小的功能，ceph的chain_xattr也提供了类似的功能。


```
static int chain_fgetxattr_len(int fd, const char *name)
{
  int i = 0, total = 0;
  char raw_name[CHAIN_XATTR_MAX_NAME_LEN * 2 + 16];
  int r;

  do {
    get_raw_xattr_name(name, i, raw_name, sizeof(raw_name));
    r = sys_fgetxattr(fd, raw_name, 0, 0);
    if (!i && r < 0)
      return r;
    if (r < 0)
      break;
    total += r;
    i++;
  } while (r == CHAIN_XATTR_MAX_BLOCK_LEN ||
	   r == CHAIN_XATTR_SHORT_BLOCK_LEN);

  return total;
}
```

这个函数，用来探查，多大的缓冲区，才能存放下某个属性的值。它自然也会探查是否是多个文件系统的扩展属性来描述一个逻辑意义上的扩展属性。


还有一个很有意思的功能是列出说有逻辑意义上的扩展属性值，

```
int chain_listxattr(const char *fn, char *names, size_t len) {
  int r;

  if (!len)
    return sys_listxattr(fn, names, len) * 2;

  r = sys_listxattr(fn, 0, 0);
  if (r < 0)
    return r;

  size_t total_len = r * 2; // should be enough
  char *full_buf = (char *)malloc(total_len);
  if (!full_buf)
    return -ENOMEM;

  r = sys_listxattr(fn, full_buf, total_len);
  if (r < 0) {
    free(full_buf);
    return r;
  }

  char *p = full_buf;
  const char *end = full_buf + r;
  char *dest = names;
  char *dest_end = names + len;

  while (p < end) {
    char name[CHAIN_XATTR_MAX_NAME_LEN * 2 + 16];
    int attr_len = strlen(p);
    bool is_first;
    int name_len = translate_raw_name(p, name, sizeof(name), &is_first);
    if (is_first)  {
      if (dest + name_len > dest_end) {
        r = -ERANGE;
        goto done;
      }
      strcpy(dest, name);
      dest += name_len + 1;
    }
    p += attr_len + 1;
  }
  r = dest - names;

done:
  free(full_buf);
  return r;
}
```

这里面有个很有意思的地方，即is_first,

我们通过文件系统层面的listxattr	可以罗列所有文件系统层面的扩展属性的键值，比如

* user.key1
* user.key2
* user.key2@1
* user.key2@2
* user.key3

注意，其中逻辑意义上的扩展属性：

* user.key1
* user.key2
* user.key3

其中user.key2@1 和 user.key2@2，并不是独立的扩展属性，它只不过是user.key2这个扩展属性的分片而已。

这就牵扯到，从文件系统层面的扩展属性名称到逻辑意义上扩展属性名称的映射。

user.key2@2 这种，经过映射，就变成了user.key2，同时is_first = false
user.key2 这种，经过映射，变成了user.key2,同时is_first = true.


## 结尾
EXT4扩展属性提供的空间太小，给Ceph 提供的闪转腾挪的空间太小,所以，在今年[ceph社区讨论废弃对EXT4的支持](http://lists.ceph.com/pipermail/ceph-users-ceph.com/2016-April/008886.html)，一时间在社区引起轩然大波。
在filestore时代，使用EXT4作为OSD本地文件系统的情况，并不罕见，而BlueStore尚未成熟的情况下，废弃对EXT4的支持，的确引起了很大的反响。




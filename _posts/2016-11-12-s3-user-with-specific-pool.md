---
layout: post
title: 给不同的S3用户指定不同的pool
date: 2016-11-12 14:43:40
categories: ceph
tag: ceph
excerpt: 指定S3账户的默认pool，即S3用户的数据写入指定的pool
---

# 前言

如果不指定的话，所有的S3用户默认的pool都是同一个，本文提供一个方法，指定某个用户的默认pool，实现用户之间的差异化。

下面以用户bean_lee为例，为其创建pool－bean，希望该用户的所有用户数据写入pool-bean这个pool。

# 方法

## 第一步 设置region

通过如下命令可以获取region信息：

```
radosgw-admin region get > /tmp/region
```

我们看到region信息如下：

```
cat /tmp/region |python -mjson.tool 
{
    "api_name": "", 
    "default_placement": "default-placement", 
    "endpoints": [], 
    "is_master": "true", 
    "master_zone": "", 
    "name": "default", 
    "placement_targets": [
        {
            "name": "default-placement", 
            "tags": []
        }
    ], 
    "zones": [
        {
            "endpoints": [], 
            "log_data": "false", 
            "log_meta": "false", 
            "name": "default"
        }
    ]
}

```

我们可以修改placement_targets信息，改成如下：

```
cat /tmp/region |python -mjson.tool 
{
    "api_name": "", 
    "default_placement": "default-placement", 
    "endpoints": [], 
    "is_master": "true", 
    "master_zone": "", 
    "name": "default", 
    "placement_targets": [
        {
            "name": "default-placement", 
            "tags": []
        }, 
        {
            "name": "bean_lee", 
            "tags": [
                "pool-bean"
            ]
        }
    ], 
    "zones": [
        {
            "endpoints": [], 
            "log_data": "false", 
            "log_meta": "false", 
            "name": "default"
        }
    ]
}

```

然后将修改后的region，设置生效：

```
radosgw-admin region set < /tmp/region
```

## 第二步 设置zone信息

可以通过如下指令获取zone信息：

```
radosgw-admin zone get > /tmp/zone
```

可以查看zone相关的信息如下：

```
{ "domain_root": ".rgw",
  "control_pool": ".rgw.control",
  "gc_pool": ".rgw.gc",
  "log_pool": ".log",
  "intent_log_pool": ".intent-log",
  "usage_log_pool": ".usage",
  "user_keys_pool": ".users",
  "user_email_pool": ".users.email",
  "user_swift_pool": ".users.swift",
  "user_uid_pool": ".users.uid",
  "system_key": { "access_key": "",
      "secret_key": ""},
  "placement_pools": [
        { "key": "default-placement",
          "val": { "index_pool": ".rgw.buckets.index",
              "data_pool": ".rgw.buckets",
              "data_extra_pool": ".rgw.buckets.extra"}}]}
```

我们修改zone信息中的placement_pools信息，添加了一个key值为bean_lee的placement_pools ,最重要的信息是data_pool指定为我们pool。


```
{
    "control_pool": ".rgw.control", 
    "domain_root": ".rgw", 
    "gc_pool": ".rgw.gc", 
    "intent_log_pool": ".intent-log", 
    "log_pool": ".log", 
    "placement_pools": [
        {
            "key": "default-placement", 
            "val": {
                "data_extra_pool": ".rgw.buckets.extra", 
                "data_pool": ".rgw.buckets", 
                "index_pool": ".rgw.buckets.index"
            }
        }, 
        {
            "key": "bean_lee", 
            "val": {
                "data_extra_pool": ".rgw.buckets.extra", 
                "data_pool": "pool-bean", 
                "index_pool": ".rgw.buckets.index"
            }
        }
    ], 
    "system_key": {
        "access_key": "", 
        "secret_key": ""
    }, 
    "usage_log_pool": ".usage", 
    "user_email_pool": ".users.email", 
    "user_keys_pool": ".users", 
    "user_swift_pool": ".users.swift", 
    "user_uid_pool": ".users.uid"
}
```

通过如下命令，修改zone的信息

```
radosgw-admin zone set < /tmp/zone
```

## 第三步 修改user 信息

首先获取用户bean_lee的相关信息：

```
radosgw-admin metadata get user:bean_lee > /tmp/bean_lee
```

其信息如下：

```
{ "key": "user:bean_lee",
  "ver": { "tag": "_h1Z0FgVKaKIRvq-4ifElmA0",
      "ver": 3},
  "mtime": 1478767056,
  "data": { "user_id": "bean_lee",
      "display_name": "bean_lee",
      "email": "bean_lee@123.com",
      "suspended": 0,
      "max_buckets": 1000,
      "auid": 0,
      "subusers": [
            { "id": "bean_lee:bean_lee",
              "permissions": "full-control"}],
      "keys": [
            { "user": "bean_lee",
              "access_key": "5EJ6XVLI6V3W8TL9R97D",
              "secret_key": "2ppUXqDYI52AMZg+h4Oo+VTPgXbiRCFtgZgifGLI"}],
      "swift_keys": [
            { "user": "bean_lee",
              "secret_key": "111111"},
            { "user": "bean_lee:bean_lee",
              "secret_key": "111111"}],
      "caps": [],
      "op_mask": "read, write, delete",
      "default_placement": "",
      "placement_tags": [], 
      "bucket_quota": { "enabled": false,
          "max_size_kb": -1,
          "max_objects": -1},
      "user_quota": { "enabled": false,
          "max_size_kb": -1,
          "max_objects": -1},
      "temp_url_keys": []}}
```

将内容修改成如下：其中default_placement一行和placement_tags 一行被修改，修改成和region placement_targets 中新增的name和tag

```
{ "key": "user:bean_lee",
  "ver": { "tag": "_h1Z0FgVKaKIRvq-4ifElmA0",
      "ver": 3},
  "mtime": 1478767056,
  "data": { "user_id": "bean_lee",
      "display_name": "bean_lee",
      "email": "bean_lee@123.com",
      "suspended": 0,
      "max_buckets": 1000,
      "auid": 0,
      "subusers": [
            { "id": "bean_lee:bean_lee",
              "permissions": "full-control"}],
      "keys": [
            { "user": "bean_lee",
              "access_key": "5EJ6XVLI6V3W8TL9R97D",
              "secret_key": "2ppUXqDYI52AMZg+h4Oo+VTPgXbiRCFtgZgifGLI"}],
      "swift_keys": [
            { "user": "bean_lee",
              "secret_key": "111111"},
            { "user": "bean_lee:bean_lee",
              "secret_key": "111111"}],
      "caps": [],
      "op_mask": "read, write, delete",
      "default_placement": "bean_lee",
      "placement_tags": ["pool-bean"], 
      "bucket_quota": { "enabled": false,
          "max_size_kb": -1,
          "max_objects": -1},
      "user_quota": { "enabled": false,
          "max_size_kb": -1,
          "max_objects": -1},
      "temp_url_keys": []}}
```


其中default_placement一行和placement_tags 一行被修改，修改成和region placement_targets 中新增的name和tag

通过如下命令使其生效：

```
radosgw-admin metadata put user:bean_lee < /tmp/bean_lee
```

## 第四步 update regionmap

```
 radosgw-admin regionmap update
```

## 第五步 重启radosgw

```
/etc/init.d/radosgw restart 
```

第五步是否必需，我还不太确定，后面会测试

# 效果

通过s3cmd	命令创建bucket，同时上传100个小文件到bucket。

```
s3cmd mb s3://bean_pool2
Bucket 's3://bean_pool2/' created
root@BEAN-1:~# seq 0 99       | xargs -I{} -P 40 s3cmd put /var/log/kern.log s3://bean_pool2/{}
/var/log/kern.log -> s3://bean_pool2/32  [1 of 1]
/var/log/kern.log -> s3://bean_pool2/26  [1 of 1]
 498 of 498   100% in    0s    23.58 kB/s  done
 498 of 498   100% in    0s    43.91 kB/s  done
/var/log/kern.log -> s3://bean_pool2/36  [1 of 1]
/var/log/kern.log -> s3://bean_pool2/39  [1 of 1]
/var/log/kern.log -> s3://bean_pool2/38  [1 of 1]
 498 of 498   100% in    0s    25.48 kB/s  done
 498 of 498   100% in    0s    19.27 kB/s  done
/var/log/kern.log -> s3://bean_pool2/37  [1 of 1]
 498 of 498   100% in    0s    14.03 kB/s  done
/var/log/kern.log -> s3://bean_pool2/30  [1 of 1]
```
通过ceph df输出，可以看到如下一行：

```
pool-bean			17 		49800     0 35981M    100
```
最后的一行OBJECTS 为100，显示了用户的数据 100个对象（消耗了100个ceph底层对象），确实存放在指定的pool中。



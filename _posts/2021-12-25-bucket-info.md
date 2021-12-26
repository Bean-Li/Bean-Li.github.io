---
layout: post
title: 查看Bucket相关的metadata
date: 2021-12-26 10:29
categories: ceph
tag: ceph rgw
excerpt: 展示Ceph Bucket相关的元数据信息
---
# 前言

本文介绍，如何查看bucket相关的元数据信息。

```
root@CVM01:~# radosgw-admin metadata get bucket:bkt_test
{
    "key": "bucket:bkt_test",
    "ver": {
        "tag": "_avOPFvyGt4fut9GiJPsIXeK",
        "ver": 1
    },
    "mtime": "2021-04-22 07:22:12.706675Z",
    "data": {
        "bucket": {
            "name": "bkt_test",
            "marker": "7240a9f2-42ea-4f3d-b534-ad4d867ab9cb.372941258.1",
            "bucket_id": "7240a9f2-42ea-4f3d-b534-ad4d867ab9cb.372941258.1",
            "tenant": "",
            "explicit_placement": {
                "data_pool": "",
                "data_extra_pool": "",
                "index_pool": ""
            }
        },
        "owner": "s3user1",
        "creation_time": "2021-04-22 07:22:12.559921Z",
        "linked": "true",
        "has_bucket_info": "false"
    }
}
root@CVM01:~# radosgw-admin metadata get bucket.instance:bkt_test:7240a9f2-42ea-4f3d-b534-ad4d867ab9cb.372941258.1
{
    "key": "bucket.instance:bkt_test:7240a9f2-42ea-4f3d-b534-ad4d867ab9cb.372941258.1",
    "ver": {
        "tag": "_CPltbMF1_HktM5SUnn1zwPn",
        "ver": 4
    },
    "mtime": "2021-04-22 07:58:32.656184Z",
    "data": {
        "bucket_info": {
            "bucket": {
                "name": "bkt_test",
                "marker": "7240a9f2-42ea-4f3d-b534-ad4d867ab9cb.372941258.1",
                "bucket_id": "7240a9f2-42ea-4f3d-b534-ad4d867ab9cb.372941258.1",
                "tenant": "",
                "explicit_placement": {
                    "data_pool": "",
                    "data_extra_pool": "",
                    "index_pool": ""
                }
            },
            "creation_time": "2021-04-22 07:22:12.559921Z",
            "owner": "s3user1",
            "flags": 6,
            "zonegroup": "0a4d68f0-0a16-4e0c-a88b-d0b708ce75f1",
            "placement_rule": "default-placement",
            "has_instance_obj": "true",
            "quota": {
                "enabled": false,
                "check_on_raw": false,
                "max_size": -1,
                "max_size_kb": 0,
                "max_objects": -1
            },
            "num_shards": 128,
            "bi_shard_hash_type": 0,
            "requester_pays": "false",
            "has_website": "false",
            "swift_versioning": "false",
            "swift_ver_location": "",
            "index_type": 0,
            "mdsearch_config": [],
            "reshard_status": 0,
            "new_bucket_instance_id": "",
            "worm": {
                "enable": false
            }
        },
        "attrs": [
            {
                "key": "user.rgw.acl",
                "val": "AgKNAAAAAwIWAAAABwAAAHMzdXNlcjEHAAAAczN1c2VyMQQDawAAAAEBAAAABwAAAHMzdXNlcjEPAAAAAQAAAAcAAABzM3VzZXIxBQM6AAAAAgIEAAAAAAAAAAcAAABzM3VzZXIxAAAAAAAAAAACAgQAAAAPAAAABwAAAHMzdXNlcjEAAAAAAAAAAAAAAAAAAAAA"
            },
            {
                "key": "user.rgw.idtag",
                "val": ""
            },
            {
                "key": "user.rgw.lc",
                "val": "AQGJAAAAAQAAAAcAAABkZWZhdWx0BgF0AAAABwAAAGRlZmF1bHQAAAAABwAAAEVuYWJsZWQDAggAAAAAAAAAAAAAAAMCCQAAAAEAAAA1AAAAAAMCCAAAAAAAAAAAAAAAAAEBBAAAAAAAAAABAQwAAAAAAAAAAAAAAAAAAAABAQwAAAAAAAAAAAAAAAAAAAA="
            }
        ]
    }
}
```

# 如何修改bucket index shards

```
root@CVM01:~# radosgw-admin zonegroup get >  tmp.conf
{
    "id": "0a4d68f0-0a16-4e0c-a88b-d0b708ce75f1",
    "name": "default",
    "api_name": "",
    "is_master": "true",
    "endpoints": [],
    "hostnames": [],
    "hostnames_s3website": [],
    "master_zone": "7240a9f2-42ea-4f3d-b534-ad4d867ab9cb",
    "zones": [
        {
            "id": "7240a9f2-42ea-4f3d-b534-ad4d867ab9cb",
            "name": "default",
            "endpoints": [],
            "log_meta": "false",
            "log_data": "false",
            "bucket_index_max_shards": 0,
            "read_only": "false",
            "tier_type": "",
            "sync_from_all": "true",
            "sync_from": []
        }
    ],
    "placement_targets": [
        {
            "name": "default-placement",
            "tags": []
        }
    ],
    "default_placement": "default-placement",
    "realm_id": ""
}

```
修改 bucket_index_max_shards = 64 之后，然后执行：

```
radosgw-admin region set < tmp.conf
```
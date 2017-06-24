---
layout: post
title: CTDB之当public interface down
date: 2017-05-13 12:43:40
categories: CTDB
tag: CTDB
excerpt: 当public interface down了之后，CTDB如何检测到，延迟多久？
---
# 前言

前面有一篇文章：[CTDB 之 重启网络虚IP消失以后](http://bean-li.github.io/CTDB-vip-return/)介绍的是虚IP因为某种原因消失之后，CTDB如何检测到虚IP消失，如何发起虚IP的接管，原来负责该虚IP的物理节点如何重新接管虚IP。

本文介绍的是，如果CTDB用的物理网口发生down，CTDB如何检测到，如果存在虚IP的话， 该节点的虚IP又是如何被其他节点接管。

# 测试方法

注意我们的CTDB使用的public interface为：bond0

```
/usr/sbin/ctdbd --reclock=/var/share/ezfs/ctdb/recovery.lock --public-addresses=/etc/ctdb/public_addresses --public-interface=bond0 -d ERR
```

在我的bond0中，只有一网卡：

```
root@BEAN-1:/var/log/ctdb# cat /proc/net/bonding/bond0 
Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: fault-tolerance (active-backup)
Primary Slave: None
Currently Active Slave: eth0
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: eth0
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 7
Permanent HW addr: 00:0c:29:9b:e2:be
Slave queue ID: 0
```

现在我们通过ifconfig eth0 down的方法，来停掉CTDB的public interface，看下CTDB的反应。

操作的命令行如下：

```shell
onnode all ctdb setdebug INFO ; sleep 2 ; date ; ifconfig eth0 down ; for i in {1..10} ; do date && ctdb status && ctdb ip&&sleep 1 ; done  ; onnode all ctdb setdebug NOTICE 
```

注意，down掉往网卡之后，我们每秒钟都会执行ctdb status和ctdb ip，目的是观测，down掉网口多久之后，才会被ctdb检测到该事件，从而ctdb 集群将down 网口的node标记成UNHEALTHY，以及虚IP接管需要花费的时间。

输出如下：

```
>> NODE: 10.11.12.3 <<

>> NODE: 10.11.12.2 <<

>> NODE: 10.11.12.1 <<
Thu Jun 22 14:27:07 CST 2017

Thu Jun 22 14:27:07 CST 2017
Number of nodes:3
pnn:0 10.11.12.3       OK
pnn:1 10.11.12.2       OK
pnn:2 10.11.12.1       OK (THIS NODE)
Generation:1562912436
Size:3
hash:0 lmaster:0
hash:1 lmaster:1
hash:2 lmaster:2
Recovery mode:NORMAL (0)
Recovery master:0
Public IPs on node 2
172.16.146.193 2
172.16.146.192 1
172.16.146.191 0
...
Thu Jun 22 14:27:16 CST 2017
Number of nodes:3
pnn:0 10.11.12.3       OK
pnn:1 10.11.12.2       OK
pnn:2 10.11.12.1       OK (THIS NODE)
Generation:1562912436
Size:3
hash:0 lmaster:0
hash:1 lmaster:1
hash:2 lmaster:2
Recovery mode:NORMAL (0)
Recovery master:0
Public IPs on node 2
172.16.146.193 2
172.16.146.192 1
172.16.146.191 0
Thu Jun 22 14:27:17 CST 2017
Number of nodes:3
pnn:0 10.11.12.3       OK
pnn:1 10.11.12.2       OK
pnn:2 10.11.12.1       UNHEALTHY (THIS NODE)
Generation:1562912436
Size:3
hash:0 lmaster:0
hash:1 lmaster:1
hash:2 lmaster:2
Recovery mode:NORMAL (0)
Recovery master:0
Public IPs on node 2
172.16.146.193 2
172.16.146.192 1
172.16.146.191 0
Thu Jun 22 14:27:25 CST 2017
Number of nodes:3
pnn:0 10.11.12.3       OK
pnn:1 10.11.12.2       OK
pnn:2 10.11.12.1       UNHEALTHY (THIS NODE)
Generation:1562912436
Size:3
hash:0 lmaster:0
hash:1 lmaster:1
hash:2 lmaster:2
Recovery mode:NORMAL (0)
Recovery master:0
Public IPs on node 2
172.16.146.193 0
172.16.146.192 1
172.16.146.191 0
...
Thu Jun 22 14:27:31 CST 2017
Number of nodes:3
pnn:0 10.11.12.3       OK
pnn:1 10.11.12.2       OK
pnn:2 10.11.12.1       UNHEALTHY (THIS NODE)
Generation:1562912436
Size:3
hash:0 lmaster:0
hash:1 lmaster:1
hash:2 lmaster:2
Recovery mode:NORMAL (0)
Recovery master:0
Public IPs on node 2
172.16.146.193 0
172.16.146.192 1
172.16.146.191 0

>> NODE: 10.11.12.3 <<

>> NODE: 10.11.12.2 <<

>> NODE: 10.11.12.1 <<

```

注意，上图中 Thu Jun 22 14:27:07 CST 2017这个时间点，开始down eth0，但是直到10秒之后的Thu Jun 22 14:27:17 CST 2017，CTDB集群才检测出CTDB该节点UNHEALTHY。注意，Thu Jun 22 14:27:16 CST 2017的时候，CTDB对应节点依然是OK，并未检测出网卡down。

延迟时间一般多久？CTDB靠什么机制检测出来public interface down？ 这是下面要解决的问题。

#  检测机制

我们翻开ctdb的log ，可以看到如下信息：

```
2017/06/22 14:27:17.465327 [261821]: server/eventscript.c:762 Starting eventscript monitor
2017/06/22 14:27:17.502480 [261821]: 10.interface: ERROR: No active slaves for bond device bond0
2017/06/22 14:27:17.503291 [261821]: iface[bond0] has changed it's link status up => down
2017/06/22 14:27:17.503791 [261821]: server/eventscript.c:485 Eventscript monitor  finished with state 1
2017/06/22 14:27:17.503815 [261821]: monitor event failed - disabling node
2017/06/22 14:27:17.503825 [261821]: Node became UNHEALTHY. Ask recovery master 0 to perform ip reallocation
```

发现public interface down，依靠的是eventscripts的monitor机制。在[CTDB 中 eventscript功能的集成](http://bean-li.github.io/eventscript/) 这篇文章中，我们介绍了ctdb_check_health 函数在正常运行期间，会执行eventscripts下的所有脚本的monitor事件，其中10.interface脚本会负责检测网卡的健康情况。

注意这套eventscripts机制在正常运行期间，检查的周期是15秒：

```
MonitorInterval         = 15
```

因此，平均来说，网口down了7.5秒之后，才会检测到，最恶劣的情况下，15秒才会检测掉。

接下来我们要分析下10.interface的monitor事件。

## 10.interface的monitor行为

```shell
monitor_interfaces()
{
    get_all_interfaces                                                 
    delete_unexpected_ips

    fail=false
    up_interfaces_found=false

    for iface in $all_interfaces ;do
        ip addr show $iface 2>/dev/null >/dev/null || {
        echo "WARNING: Interface $iface does not exist but it is used by public addresses."
        continue
        }

        # These interfaces are sometimes bond devices
        # When we use VLANs for bond interfaces, there will only
        # be an entry in /proc for the underlying real interface
        /*注意下面这段逻辑是处理bond相关的配置的*/
        realiface=`echo $iface |sed -e 's/\..*$//'`
        bi=$(get_proc "net/bonding/$realiface" 2>/dev/null) && {
        echo "$bi" | grep -q 'Currently Active Slave: None' && {
            echo "ERROR: No active slaves for bond device $realiface"
            mark_down $iface
            continue
        }
        echo "$bi" | grep -q '^MII Status: up' || {
            echo "ERROR: public network interface $realiface is down"
            mark_down $iface
            continue
        }                                                                                                                                              
        echo "$bi" | grep -q '^Bonding Mode: IEEE 802.3ad Dynamic link aggregation' && {
            # This works around a bug in the driver where the
            # overall bond status can be up but none of the actual
            # physical interfaces have a link.
            echo "$bi" | grep 'MII Status:' | tail -n +2 | grep -q '^MII Status: up' || {
                echo "ERROR: No active slaves for 802.ad bond device $realiface"
                mark_down $iface
                continue
            }
        }
        mark_up $iface
        continue
        }
        
        /*如果不是bond，则走下面的逻辑*/
        case $iface in
        lo*)
        # loopback is always working
        mark_up $iface
        ;;
        ib*)
        # we dont know how to test ib links
        mark_up $iface
        ;;
        *)
        /*通过ethtool检测Link detected来判断网卡是否up*/
        [ -z "$iface" ] || {
            [ "$(basename $(readlink /sys/class/net/$iface/device/driver) 2>/dev/null)" = virtio_net ] ||
            ethtool $iface | grep -q 'Link detected: yes' || {
            # On some systems, this is not successful when a
            # cable is plugged but the interface has not been
            # brought up previously. Bring the interface up and                                                                                        
            # try again...
            ip link set $iface up
            ethtool $iface | grep -q 'Link detected: yes' || {
                echo "ERROR: No link on the public network interface $iface"
                mark_down $iface
                continue
            }
            }
            mark_up $iface
        }
        ;;
        esac

    done

    /*考虑多张网卡绑定的bond模式，分一下场景考虑：
     * 所有网卡都OK，那么自然$fail = False,直接返回0 
     * 如果存在某张网卡down，那么$fail =True，那么不返回，而是执行下面的一句
     * 只要up_interface_found True,表示bond中至少有1个可用，如果CTDB_PARTIALLY_ONLINE_INTERFACES为yes，
     * 仍然返回0，否则返回1,表示该node UNHEALTHY/
    $fail || return 0

    $up_interfaces_found && \
        [ "$CTDB_PARTIALLY_ONLINE_INTERFACES" = "yes" ] && \
        return 0

    return 1
}
```

对于bond的情况下，因为bond可能会有多张网卡，这些网卡中可能部分网卡up可用，部分不可用。这种情况下，应该返回0 还是返回1 ？ 取决于配置CTDB_PARTIALLY_ONLINE_INTERFACES。

这部分逻辑并不难读懂，我们单步执行，我们的场景一般都有bond，结果如下：

```shell
+ ctdb_check_args monitor
+ case "$1" in
+ case "$1" in
+ monitor_interfaces
+ get_all_interfaces
++ sed -e 's/^[^\t ]*[\t ]*//' -e 's/,/ /g' -e 's/[\t ]*$//' /etc/ctdb/public_addresses
+ all_interfaces=
+ '[' bond0 ']'
+ all_interfaces='bond0 '
+ '[' '' ']'
++ ctdb -Y ip -v
++ sed -e 1d -e 's/:[^:]*:$//' -e 's/^.*://' -e 's/,/ /g'
+ ctdb_ifaces='bond0
bond0
bond0'
++ echo bond0 bond0 bond0 bond0
++ sort -u
++ tr ' ' '\n'
+ all_interfaces=bond0
+ delete_unexpected_ips
+ '[' '' = yes ']'
+ return
+ fail=false
+ up_interfaces_found=false
+ for iface in '$all_interfaces'
+ ip addr show bond0
++ echo bond0
++ sed -e 's/\..*$//'
+ realiface=bond0
++ get_proc net/bonding/bond0
+ bi='Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: fault-tolerance (active-backup)
Primary Slave: None
Currently Active Slave: eth0
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: eth0
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 00:0c:29:77:bd:53
Slave queue ID: 0'
+ echo 'Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: fault-tolerance (active-backup)
Primary Slave: None
Currently Active Slave: eth0
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: eth0
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 00:0c:29:77:bd:53
Slave queue ID: 0'
+ grep -q 'Currently Active Slave: None'
+ echo 'Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: fault-tolerance (active-backup)
Primary Slave: None
Currently Active Slave: eth0
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: eth0
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 00:0c:29:77:bd:53
Slave queue ID: 0'
+ grep -q '^MII Status: up'
+ echo 'Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: fault-tolerance (active-backup)
Primary Slave: None
Currently Active Slave: eth0
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: eth0
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 00:0c:29:77:bd:53
Slave queue ID: 0'
+ grep -q '^Bonding Mode: IEEE 802.3ad Dynamic link aggregation'
+ mark_up bond0
+ up_interfaces_found=true
+ ctdb setifacelink bond0 up
+ continue
+ false
+ return 0
+ exit 0
```



## eventscripts检测到失败之后

注意，周期性执行eventscripts monitor，有一个回调函数：

```c
            ret = ctdb_event_script_callback(ctdb,              
                            ctdb->monitor->monitor_context, ctdb_health_callback,
                            ctdb, false,
                            CTDB_EVENT_MONITOR, "%s", "");

```

执行脚本之后，会调用ctdb_health_callback。

```c
static void ctdb_health_callback(struct ctdb_context *ctdb, int status, void *p)
{
    struct ctdb_node *node = ctdb->nodes[ctdb->pnn];
    TDB_DATA data;
    struct ctdb_node_flag_change c;                                                                                                                    
    uint32_t next_interval;
    int ret;
    TDB_DATA rddata;
    struct takeover_run_reply rd;
    const char *state_str = NULL;

    c.pnn = ctdb->pnn;
    c.old_flags = node->flags;

    rd.pnn   = ctdb->pnn;
    rd.srvid = CTDB_SRVID_TAKEOVER_RUN_RESPONSE;

    rddata.dptr = (uint8_t *)&rd;
    rddata.dsize = sizeof(rd);

    if (status == -ECANCELED) {
        DEBUG(DEBUG_ERR,("Monitoring event was cancelled\n"));
        goto after_change_status;
    }

    if (status == -ETIME) {
        ctdb->event_script_timeouts++;
        if (ctdb->event_script_timeouts >= ctdb->tunable.script_timeout_count) {
            DEBUG(DEBUG_ERR, ("Maximum timeout count %u reached for eventscript. Making node unhealthy\n", ctdb->tunable.script_timeout_count));
        } else {
            /* We pretend this is OK. */
            goto after_change_status;
        }
    }
    /*之前健康，但是这一次返回值不是0，那么该node的flags置上 NODE_FLAGS_UNHEALTHY标志位*/
    if (status != 0 && !(node->flags & NODE_FLAGS_UNHEALTHY)) {
        DEBUG(DEBUG_NOTICE,("monitor event failed - disabling node\n"));
        node->flags |= NODE_FLAGS_UNHEALTHY;
        ctdb->monitor->next_interval = 5;
        
        ctdb_run_notification_script(ctdb, "unhealthy");
    } else if (status == 0 && (node->flags & NODE_FLAGS_UNHEALTHY)) {
        /*如果之前不健康，这次返回健康，那么取消NODE_FLAGS_UNHEALTHY标志位*/
        DEBUG(DEBUG_NOTICE,("monitor event OK - node re-enabled\n"));
        node->flags &= ~NODE_FLAGS_UNHEALTHY;
        ctdb->monitor->next_interval = 5;

        ctdb_run_notification_script(ctdb, "healthy");
    }

after_change_status:
    next_interval = ctdb->monitor->next_interval;
    ctdb->monitor->next_interval *= 2;
    if (ctdb->monitor->next_interval > ctdb->tunable.monitor_interval) {
        ctdb->monitor->next_interval = ctdb->tunable.monitor_interval;                                                                                 
    }
    /*预约下次ctdb_check_health*/
    event_add_timed(ctdb->ev, ctdb->monitor->monitor_context,
                timeval_current_ofs(next_interval, 0),
                ctdb_check_health, ctdb);
    /*如果新旧标志位一样，没有发生任何变化，那么直接返回*/
    if (c.old_flags == node->flags) {
        return;
    }

    c.new_flags = node->flags;

    data.dptr = (uint8_t *)&c;
    data.dsize = sizeof(c);

    /* ask the recovery daemon to push these changes out to all nodes */
    ctdb_daemon_send_message(ctdb, ctdb->pnn,
                 CTDB_SRVID_PUSH_NODE_FLAGS, data);

    if (c.new_flags & NODE_FLAGS_UNHEALTHY) {
        state_str = "UNHEALTHY";
    } else {
        state_str = "HEALTHY";
    }

    /* ask the recmaster to reallocate all addresses */
    DEBUG(DEBUG_ERR,("Node became %s. Ask recovery master %u to perform ip reallocation\n",
             state_str, ctdb->recovery_master));
    /*注意，如果节点情况发生比那话，需要告诉recovery master，发起takeover，重启发起虚IP的接管*/
    ret = ctdb_daemon_send_message(ctdb, ctdb->recovery_master, CTDB_SRVID_TAKEOVER_RUN, rddata);
    if (ret != 0) {
        DEBUG(DEBUG_ERR,(__location__ " Failed to send ip takeover run request message to %u\n", ctdb->recovery_master));
    }                                                                                                                                                  
}
```


从上面的讨论不难看出，无论node是从HEALTHY到UNHEALTHY，还是从UNHEALTHY到HEALTHY，都会及时通知其他节点，并发消息给rec master，发起takeover。

我们跳到rec master 节点，可以看到如下的日志：

```
2017/06/22 14:27:19.630180 [recoverd:98081]: recovery master forced ip reallocation
2017/06/22 14:27:19.636677 [recoverd:98081]:  172.16.146.193 -> 0 [+14641]
2017/06/22 14:27:19.732740 [97940]: server/ctdb_takeover.c:152 public address '172.16.146.193' now assigned to iface 'bond0' refs[-1]
2017/06/22 14:27:19.732781 [97940]: Takeover of IP 172.16.146.193/24 on interface bond0
2017/06/22 14:27:19.732794 [97940]: Monitoring event was cancelled
2017/06/22 14:27:19.732806 [97940]: server/eventscript.c:584 Sending SIGTERM to child pid:358288
2017/06/22 14:27:19.732841 [97940]: server/eventscript.c:762 Starting eventscript takeip bond0 172.16.146.193 24
2017/06/22 14:27:19.733316 [97940]: server/ctdb_takeover.c:163 public address '172.16.146.191' now unassigned (old iface 'bond0' refs[-1])
2017/06/22 14:27:19.733344 [97940]: server/ctdb_takeover.c:152 public address '172.16.146.191' now assigned to iface 'bond0' refs[-1]
```

rec master通过计算，将public interface down的节点负责的IP 172.16.146.193，分配给了 pnn为0 的node，凑巧是rec master节点。，后面可以看到，该节点确实Takeover of IP 172.16.146.193。

takeover的逻辑，我们此处按下不提，后续希望有单独一篇文章介绍takeover。

总之从上面讨论可以看出，一旦网卡损坏，public interface不能工作，虚IP漂移的延迟主要在于网卡down的发现。而发现机制的延迟，主要在于MonitorInterval，默认情况下15秒，因此平均延迟在7~9秒左右，最大延迟在15~17秒左右。






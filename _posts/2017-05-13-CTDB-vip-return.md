---
layout: post
title: CTDB 之 重启网络虚IP消失以后
date: 2017-05-13 12:43:40
categories: CTDB
tag: CTDB
excerpt: 重启网络之后，CTDB IP会被剔除掉，但是CTDB进程会检测到这个事件，重新设置，接管该虚IP。How？
---

# 前言

最近在测试网络服务重启，对集群的影响。其中重要一项是对 CTDB cluster的影响。我发现，网络重启之后，该节点接管的虚IP依然存在，如同持久化的IP一样。 CTDB 如何做到的？ 它如何检测到该IP已经不见，又是如何重新接管该虚IP的？


# 现象

我们以一个3 nodes的CTDB集群为例：

```
root@BEAN-2:/var/log/ctdb# ctdb ip
Public IPs on node 1
172.16.146.132 2
172.16.146.131 0
172.16.146.133 1


root@BEAN-2:/var/log/ctdb＃ ip a

....
7: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 00:0c:29:70:38:84 brd ff:ff:ff:ff:ff:ff
    inet 172.16.146.252/24 brd 172.16.146.255 scope global bond0
       valid_lft forever preferred_lft forever
    inet 172.16.146.133/24 brd 172.16.146.255 scope global secondary bond0
       valid_lft forever preferred_lft forever

```

注意，这个虚IP 133 被当前节点接管。如果我重启网络， 发现网络重启完毕后，该虚IP仍然在

```
root@BEAN-2:/var/log/ctdb# /etc/init.d/networking restart 
....

root@BEAN-2:/var/log/ctdb＃ ip a
9: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 00:0c:29:70:38:84 brd ff:ff:ff:ff:ff:ff
    inet 172.16.146.252/24 brd 172.16.146.255 scope global bond0
       valid_lft forever preferred_lft forever
    inet 172.16.146.133/24 brd 172.16.146.255 scope global secondary bond0
       valid_lft forever preferred_lft forever
```

CTDB 如何做到的？

# 分析

启动CTDB之后，会有两个进程，一个叫做CTDB Daemon，另外一个叫做Recoverd Daemon

```bash
root      990089  0.4  0.3  19968 13736 ?        SLs  May12  11:33 /usr/sbin/ctdbd --reclock=/var/share/ezfs/ctdb/recovery.lock --public-addresses=/etc/ctdb/public_addresses --public-interface=bond0 -d NOTICE
root      990237  0.1  0.1  21660  6424 ?        S    May12   4:21 /usr/sbin/ctdbd --reclock=/var/share/ezfs/ctdb/recovery.lock --public-addresses=/etc/ctdb/public_addresses --public-interface=bond0 -d NOTICE

```

我们都知道，多个节点中，有一个节点是recmater，即recovery master，这个节点是恢复流程的主节点。无论是不是recmaster，都有一个进程负责recovery相关的事宜，而该进程执行的任务，都在server/recoverd.c文件中的main_loop中。事实上，我一直想做和正在做的事情，就是分析这个函数，只不过硬分析代码，比较枯燥，我们用苏轼推荐的八面受敌法，来学习这个函数，即人为制造各种异常，看看ctdb如何反应，将各种场景都考虑清楚了，这个函数我们基本也就整明白了。

## 发现异常

异常的发现是由网络重启节点的recovery daemon发现的，在其main_loop函数中，有如下代码：

```c
    /* verify that we have all ip addresses we should have and we dont
     * have addresses we shouldnt have.
     */ 
    if (ctdb->tunable.disable_ip_failover == 0) {
        if (rec->ip_check_disable_ctx == NULL) { 
            if (verify_local_ip_allocation(ctdb, rec, pnn, nodemap) != 0) {
                DEBUG(DEBUG_ERR, (__location__ " Public IPs were inconsistent.\n"));
            }           
        }       
    }   


    /*如果当前节点不是recmaster，当前main_loop函数可以返回了
     *即，recmaster承担的职责要多一些，普通非recmaster职责少一些*/
    if (pnn != rec->recmaster) {
        return;          
    }   
```

recmaster 承担的职责要比普通的非 recmaster承担的职责要多一些。在返回之前，会执行一个verify_local_ip_allocation函数，该函数前面的注释已经写的比较明确了，该函数的作用是检查，检查

 * 是否应该由我们负责IP，我们尚未负责
 * 不该由我们负责的IP，我们仍然持有着
 
```c
static int verify_local_ip_allocation(struct ctdb_context *ctdb, struct ctdb_recoverd *rec, uint32_t pnn, struct ctdb_node_map *nodemap)
{
    TALLOC_CTX *mem_ctx = talloc_new(NULL);
    struct ctdb_control_get_ifaces *ifaces = NULL;
    struct ctdb_all_public_ips *ips = NULL;
    struct ctdb_uptime *uptime1 = NULL;
    struct ctdb_uptime *uptime2 = NULL;
    int ret, j;
    bool need_iface_check = false;
    bool need_takeover_run = false;

    /*获取两次uptime，是为了确认，在我们执行的过程中，是不是已经发起了recovery*/
    ret = ctdb_ctrl_uptime(ctdb, mem_ctx, CONTROL_TIMEOUT(),
                CTDB_CURRENT_NODE, &uptime1);
    if (ret != 0) {
        DEBUG(DEBUG_ERR, ("Unable to get uptime from local node %u\n", pnn));
        talloc_free(mem_ctx);
        return -1;
    }
    
    /* read the interfaces from the local node */
    ret = ctdb_ctrl_get_ifaces(ctdb, CONTROL_TIMEOUT(), CTDB_CURRENT_NODE, mem_ctx, &ifaces);
    if (ret != 0) { 
        DEBUG(DEBUG_ERR, ("Unable to get interfaces from local node %u\n", pnn));
        talloc_free(mem_ctx);
        return -1;
    }

    if (!rec->ifaces) {
        need_iface_check = true;
    } else if (rec->ifaces->num != ifaces->num) {
        need_iface_check = true;
    } else if (memcmp(rec->ifaces, ifaces, talloc_get_size(ifaces)) != 0) {
        need_iface_check = true;
    }

    if (need_iface_check) {
        DEBUG(DEBUG_NOTICE, ("The interfaces status has changed on "
                     "local node %u - force takeover run\n",
                     pnn));
        need_takeover_run = true;
    }

    /* read the ip allocation from the local node */
    ret = ctdb_ctrl_get_public_ips(ctdb, CONTROL_TIMEOUT(), CTDB_CURRENT_NODE, mem_ctx, &ips);
    if (ret != 0) {
        DEBUG(DEBUG_ERR, ("Unable to get public ips from local node %u\n", pnn));
        talloc_free(mem_ctx);
        return -1;
    }
    ret = ctdb_ctrl_uptime(ctdb, mem_ctx, CONTROL_TIMEOUT(),
                CTDB_CURRENT_NODE, &uptime2);
    if (ret != 0) {
        DEBUG(DEBUG_ERR, ("Unable to get uptime from local node %u\n", pnn));
        talloc_free(mem_ctx);
        return -1;
    }

    /* skip the check if the startrecovery time has changed */
    if (timeval_compare(&uptime1->last_recovery_started,
                &uptime2->last_recovery_started) != 0) {
        DEBUG(DEBUG_NOTICE, (__location__ " last recovery time changed while we read the public ip list. skipping public ip address check\n"));
        talloc_free(mem_ctx);
        return 0;
    }

    /* skip the check if the endrecovery time has changed */
    if (timeval_compare(&uptime1->last_recovery_finished,
                &uptime2->last_recovery_finished) != 0) {
        DEBUG(DEBUG_NOTICE, (__location__ " last recovery time changed while we read the public ip list. skipping public ip address check\n"));
        talloc_free(mem_ctx);
        return 0;
    }

    /* skip the check if we have started but not finished recovery */
    if (timeval_compare(&uptime1->last_recovery_finished,
                &uptime1->last_recovery_started) != 1) {
        DEBUG(DEBUG_INFO, (__location__ " in the middle of recovery or ip reallocation. skipping public ip address check\n"));
        talloc_free(mem_ctx);

        return 0;
    }

    talloc_free(rec->ifaces);
    rec->ifaces = talloc_steal(rec, ifaces);

    /* verify that we have the ip addresses we should have
       and we dont have ones we shouldnt have.
       if we find an inconsistency we set recmode to
       active on the local node and wait for the recmaster
       to do a full blown recovery.
       also if the pnn is -1 and we are healthy and can host the ip
       we also request a ip reallocation.
    */
    if (ctdb->tunable.disable_ip_failover == 0) {
        for (j=0; j<ips->num; j++) {
            if (ips->ips[j].pnn == -1 && nodemap->nodes[pnn].flags == 0) {
                DEBUG(DEBUG_CRIT,("Public address '%s' is not assigned and we could serve this ip\n",
                        ctdb_addr_to_str(&ips->ips[j].addr)));
                need_takeover_run = true;
            } else if (ips->ips[j].pnn == pnn) {
                if (!ctdb_sys_have_ip(&ips->ips[j].addr)) {
                    DEBUG(DEBUG_CRIT,("Public address '%s' is missing and we should serve this ip\n",
                        ctdb_addr_to_str(&ips->ips[j].addr)));
                    need_takeover_run = true;
                }
            } else {         
                if (ctdb_sys_have_ip(&ips->ips[j].addr)) {
                    DEBUG(DEBUG_CRIT,("We are still serving a public address '%s' that we should not be serving.\n",
                        ctdb_addr_to_str(&ips->ips[j].addr)));
                    need_takeover_run = true;
                }

            } else {
                if (ctdb_sys_have_ip(&ips->ips[j].addr)) {
                    DEBUG(DEBUG_CRIT,("We are still serving a public address '%s' that we should not be serving.\n",
                        ctdb_addr_to_str(&ips->ips[j].addr))); 
                    need_takeover_run = true;
                }
            }
        }
    }

    if (need_takeover_run) {
        struct takeover_run_reply rd;
        TDB_DATA data;

        DEBUG(DEBUG_CRIT,("Trigger takeoverrun\n"));

        rd.pnn = ctdb->pnn;
        rd.srvid = 0;
        data.dptr = (uint8_t *)&rd;
        data.dsize = sizeof(rd);

        ret = ctdb_client_send_message(ctdb, rec->recmaster, CTDB_SRVID_TAKEOVER_RUN, data);
        if (ret != 0) {
            DEBUG(DEBUG_ERR,(__location__ " Failed to send ipreallocate to recmaster :%d\n", (int)rec->recmaster));
        }
    }
    talloc_free(mem_ctx);
    return 0;
}
```

注意，这个函数严格来讲，可以分成两个部分来理解，第一部分是iface是否有变化，第二部分是public IP部分是否有异常。

第一部分截至到ctdb_ctrl_get_public_ips调用之前，如果发现recovery daemon中记录的iface和 ctdb daemon记录的ifaces不一致，那么就会标记need_takeover_run标志位。这一部分并非我们的重点，我们略过。

第二部分是我们的重点，

对于我们的重启network的测试，会导致我们负责的虚IP丢失，因为毕竟虚IP没有记录在配置文件里面，是由CTDB进程负责的IP。

```c
              else if (ips->ips[j].pnn == pnn) {
                if (!ctdb_sys_have_ip(&ips->ips[j].addr)) {
                    DEBUG(DEBUG_CRIT,("Public address '%s' is missing and we should serve this ip\n",
                        ctdb_addr_to_str(&ips->ips[j].addr)));
                    need_takeover_run = true;                       
                }

```
这一部分逻辑非常重要，对于我们这种测试场景而言。就是这部分逻辑发现了，本应该由我们负责的虚IP，而我们节点实际上并没有该IP。这时候会打印如下输出，并且标记need_takeover_run = true

```
2017/05/14 12:31:00.793190 [recoverd:862344]: Public address '172.16.146.133' is missing and we should serve this ip
2017/05/14 12:31:00.793251 [recoverd:862344]: Trigger takeoverrun
```

如果，need_takeover_run标志位为true,那么会向recmater发送CTDB_SRVID_TAKEOVER_RUN消息:

```c
    if (need_takeover_run) {
        struct takeover_run_reply rd;
        TDB_DATA data;

        DEBUG(DEBUG_CRIT,("Trigger takeoverrun\n"));

        rd.pnn = ctdb->pnn;
        rd.srvid = 0;
        data.dptr = (uint8_t *)&rd;
        data.dsize = sizeof(rd);

        ret = ctdb_client_send_message(ctdb, rec->recmaster, CTDB_SRVID_TAKEOVER_RUN, data);
        if (ret != 0) {
            DEBUG(DEBUG_ERR,(__location__ " Failed to send ipreallocate to recmaster :%d\n", (int)rec->recmaster));
        }                
    }
```

## recmaster 处理CTDB_SRVID_TAKEOVER_RUN 事件

注意monitor_cluster 函数中，已经注册了该事件的处理函数：

```
        /* register a message port for performing a takeover run */
        ctdb_client_set_message_handler(ctdb, CTDB_SRVID_TAKEOVER_RUN, ip_reallocate_handler, rec);
```
也就说，收到该事件，需要执行ip_reallocate_handler函数：


```c
/*
  handler for ip reallocate, just add it to the list of callers and 
  handle this later in the monitor_cluster loop so we do not recurse
  with other callers to takeover_run()
*/

static void ip_reallocate_handler(struct ctdb_context *ctdb, uint64_t srvid, TDB_DATA data, void *private_data)
{
        struct ctdb_recoverd *rec = talloc_get_type(private_data, struct ctdb_recoverd);
        struct ip_reallocate_list *caller;

        if (data.dsize != sizeof(struct rd_memdump_reply)) {
                DEBUG(DEBUG_ERR, (__location__ " Wrong size of return address.\n"));
                return;
        }

        if (rec->ip_reallocate_ctx == NULL) {
                rec->ip_reallocate_ctx = talloc_new(rec);
                CTDB_NO_MEMORY_FATAL(ctdb, rec->ip_reallocate_ctx);
        }

        caller = talloc(rec->ip_reallocate_ctx, struct ip_reallocate_list);
        CTDB_NO_MEMORY_FATAL(ctdb, caller);

        caller->rd   = (struct rd_memdump_reply *)talloc_steal(caller, data.dptr);
        caller->next = rec->reallocate_callers;
        rec->reallocate_callers = caller;

        return;
}

```

创建一个任务，放入rec->raallocate_callers链表，本不是立刻处理这个请求，而是在recovery daemon的main_loop中处理事件，防范递归执行：

在recmaster节点的main_loop中，会判断是否有reallocate_callers，如果有的话，会在main_loop中处理：

```
        /* if there are takeovers requested, perform it and notify the waiters */
        if (rec->reallocate_callers) {
                process_ipreallocate_requests(ctdb, rec);
        }
```


## 处理

下面我们可以进入process_ipreallocate_requests函数，来看下recmaster如何处理。

```c
static void process_ipreallocate_requests(struct ctdb_context *ctdb, struct ctdb_recoverd *rec)
{                                                                                                                                                   
    TALLOC_CTX *tmp_ctx = talloc_new(ctdb);
    TDB_DATA result;
    int32_t ret;
    struct ip_reallocate_list *callers;
    uint32_t culprit;

    DEBUG(DEBUG_INFO, ("recovery master forced ip reallocation\n"));

    /* update the list of public ips that a node can handle for
       all connected nodes
    */
    ret = ctdb_reload_remote_public_ips(ctdb, rec, rec->nodemap, &culprit);
    if (ret != 0) {
        DEBUG(DEBUG_ERR,("Failed to read public ips from remote node %d\n",
                 culprit));
        rec->need_takeover_run = true;
    }
    if (ret == 0) {
        ret = ctdb_takeover_run(ctdb, rec->nodemap);
        if (ret != 0) {
            DEBUG(DEBUG_ERR,("Failed to reallocate addresses: ctdb_takeover_run() failed.\n"));
            rec->need_takeover_run = true;
        }
    }

```
从recmaster节点中的日志里，可以看到如下打印：

```
2017/05/14 12:31:01.406400 [recoverd:990237]: recovery master forced ip reallocation
```
ctdb_takeover_run 这个函数应该算是虚IP管理的核心了，它的意思是根据当前的nodemap，以及所有的虚IP，计算出来，哪个虚IP应该有哪个node接管。这里面有两件事情

* 不该我接管的虚IP，我应该释放掉
* 该我接管的虚IP，我应该接管

因此ctdb_takeover_run 函数可以分成3部分，

* 根据nodemap和虚IP，计算出负责关系

```c
    /* Do the IP reassignment calculations */
    ctdb_takeover_run_core(ctdb, nodemap, &all_ips);
```
* 通知所有节点，不要接管不该自己接管的IP：

```c
            if (tmp_ip->addr.sa.sa_family == AF_INET) {
                ipv4.pnn = tmp_ip->pnn;
                ipv4.sin = tmp_ip->addr.ip;

                timeout = TAKEOVER_TIMEOUT();
                data.dsize = sizeof(ipv4);
                data.dptr  = (uint8_t *)&ipv4;
                state = ctdb_control_send(ctdb, nodemap->nodes[i].pnn,
                        0, CTDB_CONTROL_RELEASE_IPv4, 0,
                        data, async_data,
                        &timeout, NULL);
            } else {
                ip.pnn  = tmp_ip->pnn;
                ip.addr = tmp_ip->addr;                                                                                                             

                timeout = TAKEOVER_TIMEOUT();
                data.dsize = sizeof(ip);
                data.dptr  = (uint8_t *)&ip;
                state = ctdb_control_send(ctdb, nodemap->nodes[i].pnn,
                        0, CTDB_CONTROL_RELEASE_IP, 0,
                        data, async_data,
                        &timeout, NULL);
            }
```
* 通知所有节点接管自己应该负责的虚IP

```c
        if (tmp_ip->addr.sa.sa_family == AF_INET) {
            ipv4.pnn = tmp_ip->pnn;
            ipv4.sin = tmp_ip->addr.ip;

            timeout = TAKEOVER_TIMEOUT();
            data.dsize = sizeof(ipv4);
            data.dptr  = (uint8_t *)&ipv4;
            state = ctdb_control_send(ctdb, tmp_ip->pnn,
                    0, CTDB_CONTROL_TAKEOVER_IPv4, 0,
                    data, async_data,
                    &timeout, NULL);
        } else {
            ip.pnn  = tmp_ip->pnn;
            ip.addr = tmp_ip->addr;

            timeout = TAKEOVER_TIMEOUT();
            data.dsize = sizeof(ip);
            data.dptr  = (uint8_t *)&ip;
            state = ctdb_control_send(ctdb, tmp_ip->pnn,
                    0, CTDB_CONTROL_TAKEOVER_IP, 0,
                    data, async_data,
                    &timeout, NULL);
        } 
```

* 第四步，通知所有节点执行所有脚本的ipreallocated事件。

```c
ipreallocated:  
    /* tell all nodes to update natwg */
    /* send the flags update natgw on all connected nodes */
    data.dptr  = discard_const("ipreallocated");
    data.dsize = strlen((char *)data.dptr) + 1; 
    nodes = list_of_connected_nodes(ctdb, nodemap, tmp_ctx, true);
    if (ctdb_client_async_control(ctdb, CTDB_CONTROL_RUN_EVENTSCRIPTS,
                      nodes, 0, TAKEOVER_TIMEOUT(),
                      false, data,
                      NULL, NULL,
                      NULL) != 0) {
        DEBUG(DEBUG_ERR, (__location__ " ctdb_control to updatenatgw failed\n"));
    }

```
注意，第二步和第三步之间，必须要插入等待，第二步完成之后，方可进行第三步:

```c
    if (ctdb_client_async_wait(ctdb, async_data) != 0) {
        DEBUG(DEBUG_ERR,(__location__ " Async control CTDB_CONTROL_RELEASE_IP failed\n"));
        talloc_free(tmp_ctx);
        return -1;
    }
```
以我们的CTDB集群为例，三个node，三个虚IP，通过计算每个节点都会收到2条这样的消息：XX不该由你负责，如果你有这个IP的话，请release； 同时任何节点也会收到这样的消息，即XX 应该由你负责，如果你还没有IP，请接管这个IP。

```
node 253 which should hold 131:
-------------------------------
2017/05/14 12:31:01.408634 [990089]: server/ctdb_takeover.c:163 public address '172.16.146.133' now unassigned (old iface '__none__' refs[0])
2017/05/14 12:31:01.408708 [990089]: server/ctdb_takeover.c:163 public address '172.16.146.132' now unassigned (old iface '__none__' refs[0])

2017/05/14 12:31:01.409693 [990089]: Redundant takeover of IP 172.16.146.131/24 on interface bond0 (ip already held)

node 252 which should hold 133:
---------------------------------
2017/05/14 12:31:01.408735 [862113]: server/ctdb_takeover.c:163 public address '172.16.146.132' now unassigned (old iface '__none__' refs[0])
2017/05/14 12:31:01.408802 [862113]: server/ctdb_takeover.c:163 public address '172.16.146.131' now unassigned (old iface '__none__' refs[0])


2017/05/14 12:31:01.409617 [862113]: server/ctdb_takeover.c:132 public address '172.16.146.133' still assigned to iface 'bond0'
2017/05/14 12:31:01.409637 [862113]: Takeover of IP 172.16.146.133/24 on interface bond0
2017/05/14 12:31:01.409650 [862113]: server/eventscript.c:762 Starting eventscript takeip bond0 172.16.146.133 24

2017/05/14 12:31:01.524279 [862113]: server/eventscript.c:485 Eventscript takeip bond0 172.16.146.133 24 finished with state 0
```


## 所有节点处理 CTDB_CONTROL_RELEASE_IPv4事件

注意，对于重启网络这种情况，因为CTDB nodemap并未发生变化，并没有任何一个节点的CTDB进程死掉或者失联，因此，各个节点负责的虚IP还是和重启之前一样。

以node 253为例(IP为10.11.12.3)，该节点负责的虚IP是 172.16.146.131，一直都是该IP。

```
root@BEAN-3:/var/log/ctdb# ctdb ip
Public IPs on node 0
172.16.146.132 2
172.16.146.131 0
172.16.146.133 0

root@BEAN-3:/var/log/ctdb# ctdb status
Number of nodes:3
pnn:0 10.11.12.3       OK (THIS NODE)
```

它会先收到2个请求，通知它释放IP 172.16.146.132和172.16.146.133，如果它有这两个IP的话。可是事实上，该CTDB node根本就没有接管这两个IP，因此，打开debug的话我们可以看到如下的日志：

```
2017/05/14 12:07:05.905499 [990089]: Redundant release of IP 172.16.146.133/24 on interface __none__ (ip not held)
2017/05/14 12:07:05.905517 [990089]: server/ctdb_takeover.c:163 public address '172.16.146.133' now unassigned (old iface '__none__' refs[0])

2017/05/14 12:07:05.905644 [990089]: Redundant release of IP 172.16.146.132/24 on interface __none__ (ip not held)
2017/05/14 12:07:05.905656 [990089]: server/ctdb_takeover.c:163 public address '172.16.146.132' now unassigned (old iface '__none__' refs[0])

```
即，我们压根就不负责这两个IP，因此，你让我释放，纯属多余，我就直接返回了，代码如下：

```c
int32_t ctdb_control_release_ip(struct ctdb_context *ctdb, 
                struct ctdb_req_control *c,
                TDB_DATA indata, 
                bool *async_reply)
{
  
      ...
      if (!ctdb_sys_have_ip(&pip->addr)) {
        DEBUG(DEBUG_DEBUG,("Redundant release of IP %s/%u on interface %s (ip not held)\n", 
            ctdb_addr_to_str(&pip->addr),
            vnn->public_netmask_bits, 
            ctdb_vnn_iface_string(vnn)));
        ctdb_vnn_unassign_iface(ctdb, vnn);
        return 0;           
    }
    ...
}

static void ctdb_vnn_unassign_iface(struct ctdb_context *ctdb,
                    struct ctdb_vnn *vnn)
{
    DEBUG(DEBUG_INFO, (__location__ " public address '%s' "
               "now unassigned (old iface '%s' refs[%d])\n",
               ctdb_addr_to_str(&vnn->public_address),
               ctdb_vnn_iface_string(vnn),
               vnn->iface?vnn->iface->references:0));
    if (vnn->iface) {
        vnn->iface->references--;
    }
    vnn->iface = NULL;
    if (vnn->pnn == ctdb->pnn) {
        vnn->pnn = -1;
    }
```

## 所有节点处理 CTDB_CONTROL_TAKEOVER_IPv4事件

所有节点都会收到一条请求，要求负责起自己应该负责的虚IP。这里面有2个情况，一种是整个事件的无辜者，即该节点原本就一直在负责该虚IP，并未有任何松动或者异常。

对于这种节点，我们通常能够看到如下打印：

```
2017/05/14 12:31:01.409693 [990089]: Redundant takeover of IP 172.16.146.131/24 on interface bond0 (ip already held)
```
简单说就是一句话，啥也不用干，老子原本就接管着该IP呢。

但是对于整个震荡的始作俑者，它重启了网络，导致了虚IP的丢失，它的情况就不太一样，它需要重新接管这个虚IP，它的log如下所示：

```
2017/05/14 12:31:01.409617 [862113]: server/ctdb_takeover.c:132 public address '172.16.146.133' still assigned to iface 'bond0'
2017/05/14 12:31:01.409637 [862113]: Takeover of IP 172.16.146.133/24 on interface bond0
2017/05/14 12:31:01.409650 [862113]: server/eventscript.c:762 Starting eventscript takeip bond0 172.16.146.133 24

2017/05/14 12:31:01.524279 [862113]: server/eventscript.c:485 Eventscript takeip bond0 172.16.146.133 24 finished with state 0
2017/05/14 12:31:01.524340 [862113]: server/ctdb_takeover.c:362 sending TAKE_IP for '172.16.146.133'

```

也就说，该节点要实打实地重新接管自己原本该负责的172.16.146.133这个IP.
```c
int32_t ctdb_control_takeover_ip(struct ctdb_context *ctdb,
                 struct ctdb_req_control *c,
                 TDB_DATA indata,                  
                 bool *async_reply)
{
    int ret;
    struct ctdb_public_ip *pip = (struct ctdb_public_ip *)indata.dptr;
    struct ctdb_vnn *vnn;
    bool have_ip = false;
    bool do_updateip = false;
    bool do_takeip = false;
    struct ctdb_iface *best_iface = NULL;

    /*每个节点都有一个PNN号码，该号码的值对应的是 
      nodes文件中的行号，从0 开始*/
    if (pip->pnn != ctdb->pnn) {
       /*pnn不是我的pnn，表示发错了，一般不会走入本分支*/
        DEBUG(DEBUG_ERR,(__location__" takeoverip called for an ip '%s' "
        "with pnn %d, but we're node %d\n",
                 ctdb_addr_to_str(&pip->addr),
                 pip->pnn, ctdb->pnn));
        return -1;
    }

    /* update out vnn list */
    /*让我接管的IP，压根就不是合法的虚IP，直接返回*/
    vnn = find_public_ip_vnn(ctdb, &pip->addr);
    if (vnn == NULL) {
        DEBUG(DEBUG_INFO,("takeoverip called for an ip '%s' that is not a public address\n",
            ctdb_addr_to_str(&pip->addr)));
        return 0;
    }

    /* update out vnn list */
    vnn = find_public_ip_vnn(ctdb, &pip->addr);
    if (vnn == NULL) {
        DEBUG(DEBUG_INFO,("takeoverip called for an ip '%s' that is not a public address\n",
            ctdb_addr_to_str(&pip->addr)));
        return 0;
    }

    have_ip = ctdb_sys_have_ip(&pip->addr);
    best_iface = ctdb_vnn_best_iface(ctdb, vnn);
    if (best_iface == NULL) {
        DEBUG(DEBUG_ERR,("takeoverip of IP %s/%u failed to find"
                 "a usable interface (old %s, have_ip %d)\n",
                 ctdb_addr_to_str(&vnn->public_address),
                 vnn->public_netmask_bits,
                 ctdb_vnn_iface_string(vnn),
                 have_ip));
        return -1;
    }

    if (vnn->iface == NULL && vnn->pnn == -1 && have_ip && best_iface != NULL) {
        DEBUG(DEBUG_ERR,("Taking over newly created ip\n"));
        have_ip = false;
    }
    
    if (vnn->iface == NULL && have_ip) {
        DEBUG(DEBUG_CRIT,(__location__ " takeoverip of IP %s is known to the kernel, "
                  "but we have no interface assigned, has someone manually configured it? Ignore for now.\n",
                 ctdb_addr_to_str(&vnn->public_address)));
        return 0;
    }

    if (vnn->pnn != ctdb->pnn && have_ip && vnn->pnn != -1) {
        DEBUG(DEBUG_CRIT,(__location__ " takeoverip of IP %s is known to the kernel, "
                  "and we have it on iface[%s], but it was assigned to node %d"
                  "and we are node %d, banning ourself\n",
                 ctdb_addr_to_str(&vnn->public_address),
                 ctdb_vnn_iface_string(vnn), vnn->pnn, ctdb->pnn));
        ctdb_ban_self(ctdb);
        return -1;
    }

    if (vnn->pnn == -1 && have_ip) {
        vnn->pnn = ctdb->pnn;
        DEBUG(DEBUG_CRIT,(__location__ " takeoverip of IP %s is known to the kernel, "
                  "and we already have it on iface[%s], update local daemon\n",
                 ctdb_addr_to_str(&vnn->public_address),
                  ctdb_vnn_iface_string(vnn)));
        return 0;
    }   
    if (vnn->iface) {
        if (vnn->iface->link_up) {
            /* only move when the rebalance gains something */
            if (vnn->iface->references > (best_iface->references + 1)) {
                do_updateip = true;
            }
        } else if (vnn->iface != best_iface) {
            do_updateip = true;
        }
    }

    /*我们走到是这个分支，毫无疑问，我们并没有这个IP，因此需要设置do_takeip=true*/
    if (!have_ip) {
        if (do_updateip) {
            ctdb_vnn_unassign_iface(ctdb, vnn);
            do_updateip = false;
        }
        do_takeip = true;
    }

    if (do_takeip) {
        ret = ctdb_do_takeip(ctdb, c, vnn);
        if (ret != 0) {
            return -1;
        }
    } else if (do_updateip) {
        ret = ctdb_do_updateip(ctdb, c, vnn);          
        if (ret != 0) {
            return -1;
        }
    } else{
        /*
         * The interface is up and the kernel known the ip
         * => do nothing
         */
        DEBUG(DEBUG_INFO,("Redundant takeover of IP %s/%u on interface %s (ip already held)\n",
            ctdb_addr_to_str(&pip->addr),
            vnn->public_netmask_bits,
            ctdb_vnn_iface_string(vnn)));
        return 0;
    }

    /* tell ctdb_control.c that we will be replying asynchronously */
    *async_reply = true;

    return 0;
}
```

对于我们这种情况而言，我们并没有这个public ip，因此，我们需要执行ctdb_do_takeip函数：

```c
static int32_t ctdb_do_takeip(struct ctdb_context *ctdb,
                  struct ctdb_req_control *c,
                  struct ctdb_vnn *vnn)
{                                                                          
    int ret;
    struct ctdb_do_takeip_state *state;

    ret = ctdb_vnn_assign_iface(ctdb, vnn);
    if (ret != 0) {
        DEBUG(DEBUG_ERR,("Takeover of IP %s/%u failed to "
                 "assin a usable interface\n",
                 ctdb_addr_to_str(&vnn->public_address),
                 vnn->public_netmask_bits));
        return -1;
    }

    state = talloc(vnn, struct ctdb_do_takeip_state);
    CTDB_NO_MEMORY(ctdb, state);

    state->c = talloc_steal(ctdb, c);
    state->vnn   = vnn;

    DEBUG(DEBUG_NOTICE,("Takeover of IP %s/%u on interface %s\n",
                ctdb_addr_to_str(&vnn->public_address),
                vnn->public_netmask_bits,
                ctdb_vnn_iface_string(vnn)));

    /*全局之眼，真正接管IP的事情，交给了eventscript*/
    ret = ctdb_event_script_callback(ctdb,
                     state,
                     ctdb_do_takeip_callback,
                     state,
                     false,
                     CTDB_EVENT_TAKE_IP,
                     "%s %s %u",
                     ctdb_vnn_iface_string(vnn),
                     ctdb_addr_to_str(&vnn->public_address),
                     vnn->public_netmask_bits);
                     
    if (ret != 0) {
        DEBUG(DEBUG_ERR,(__location__ " Failed to takeover IP %s on interface %s\n",
            ctdb_addr_to_str(&vnn->public_address),
            ctdb_vnn_iface_string(vnn)));
        talloc_free(state);
        return -1;
    }

    return 0;
}

```
最终也是调用了eventscript执行来执行takeover ip，如下所示：

```
2017/05/14 12:31:01.409650 [862113]: server/eventscript.c:762 Starting eventscript takeip bond0 172.16.146.133 24

2017/05/14 12:31:01.524279 [862113]: server/eventscript.c:485 Eventscript takeip bond0 172.16.146.133 24 finished with state 0
```

其中比较重要的两个脚本是10.interface 和13.per_ip_routing。 先简单介绍下 10.interface吧。

```bash
# called when ctdbd wants to claim an IP address
     takeip)
	iface=$2
	ip=$3
	maskbits=$4

	add_ip_to_iface $iface $ip $maskbits || {
		exit 1;
	}

	# cope with the script being killed while we have the interface blocked
	iptables -D INPUT -i $iface -d $ip -j DROP 2> /dev/null

	# flush our route cache
	set_proc sys/net/ipv4/route/flush 1
	;;

```
这个 add_ip_to_iface实现在functions中：

```
add_ip_to_iface()
{
        local _iface=$1
        local _ip=$2
        local _maskbits=$3
        local _state_dir="$CTDB_VARDIR/state/interface_modify"
        local _lockfile="$_state_dir/$_iface.flock"
        local _readd_base="$_state_dir/$_iface.readd.d"

        mkdir -p $_state_dir || {
                ret=$?
                echo "Failed to mkdir -p $_state_dir - $ret"
                return $ret
        }

        test -f $_lockfile || {
                touch $_lockfile
        }
     
        flock --timeout 30 $_lockfile $CTDB_BASE/interface_modify.sh add "$_iface" "$_ip" "$_maskbits" "$_readd_base"
        return $?
}

```
除了这些锁部分之外，调用了 interface_modify.sh中的内容：

```bash
add_ip_to_iface()
{
	local _iface=$1
	local _ip=$2
	local _maskbits=$3
	local _readd_base=$4
	local _script_dir="$_readd_base/$_ip.$_maskbits"

	# we make sure the interface is up first
	/*先把对应的NIC up*/
	ip link set $_iface up || {
		echo "Failed to bringup interface $_iface"
		return 1;
	}
	/*调用ip addr add来添加ip*/
	ip addr add $_ip/$_maskbits brd + dev $_iface || {
		echo "Failed to add $_ip/$_maskbits on dev $_iface"
		return 1;
	}

	mkdir -p $_script_dir || {
		echo "Failed to mkdir -p $_script_dir"
		return 1;
	}

	rm -f $_script_dir/*

	return 0;
}
```
至此，我们的虚IP就又回来了。

要介绍完整流程，还需要介绍ipreallocated，太长了，我们就此结束。后面希望有专门的代码来介绍eventscript的各个事件，以及触发条件。


#  尾声

这个流程太长了，简单概括下流程

* 重启网络的节点会在 recovery daemon 的main_loop 函数中，会发现，自己该负责的虚IP不见了， 发送处理CTDB_SRVID_TAKEOVER_RUN到recmaster节点
* recmaster收到事件后，放入ctdb_recoverd->reallocate_callers链表中
* 在recover daemon的下一个main_loop中会处理，调用process_ipreallocate_requests处理这个事件

  * 调用ctdb_takeover_run_core重新计算各个节点需要负责的虚IP
  * 向所有节点发送消息，释放自己不应该负责的虚IP
  * 向所有节点发消息，接管自己需要接管的虚IP
  * 所有节点执行ipreallocated


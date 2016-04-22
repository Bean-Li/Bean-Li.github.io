---
layout: post
title: RHEL7 安装KVM 及 PCI直通
date: 2016-03-30 23:20:40
categories: linux
tag: linux KVM
excerpt: 命令行访问amazon s3 object
---

# RHEL7安装KVM及PCI直通

## REHEL7的安装
因为安装RHEL7是为了安装KVM，并且讲我们的存储安装在虚拟机中，其中为了提升存储的性能，需要讲RAID卡直通给KVM虚拟机。
因为RAID卡需要直通给虚拟机，因此，系统盘不应该在RAID卡上，否则RAID卡直通，会导致RHEL7无法进入系统。

首先我从RAID组中取出一块SATA盘，放入主机机箱中，通过SAS线直接和主板相连，除此SATA盘外，还组建了3个RAID分别是10T 8T和一个400G的SSD，通过RAID卡和主板相连。

![image](/assets/RHEL_KVM/RHEL-welcome.jpg)


![image](/assets/RHEL_KVM/RHEL-software-select.jpg)

![image](/aseets/RHEL_KVM/RHEL-disk-select-1.jpg)

![image](/assets/RHEL_KVM/RHEL-disk-select-2.jpg)

上述步骤中，有两个部分值得注意：

* 软件包部分我选择了 Virtualization Host。
* 分区方式选择默认即可

默认情况下，约有50G的启动分区，其它所有空间除了少量分给swap和boot外，基本都分配给了/home分区。对于我的情况而言，/home所在地分区越有2.5T左右。这没有关系，因为后面我们完全可指定 KVM的image文件的路径。


## 设置软件仓库

我们采用yum install的方式安装相关的Packages，因此，需要设置软件仓库repo。当然首当其冲的是设置网络。

我用手动修改配置文件的方式：关键在于IPADDR/NETMASK/GATEWAY/DNS1 .

```
[root@localhost sysconfig]# cat /etc/sysconfig/network-scripts/ifcfg-enp6s0f0 
HWADDR="60:EB:69:9B:54:1C"
TYPE="Ethernet"
BOOTPROTO="static"
IPADDR="10.16.17.196"
NETMASK="255.255.255.0"
GATEWAY="10.16.17.254"
DNS1="114.114.114.114"
IPV4_FAILURE_FATAL="no"
NAME="enp6s0f0"
UUID="569b59ae-0201-41b3-96ee-2cfd063485ae"
ONBOOT="yes"
```

网络设置好后，不妨通过ping baidu来确认能够联通外网。

确保能连通外网后，首先下载epel这个RPM。

```
[root@localhost ~]# wget https://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-6.noarch.rpm
--2016-04-21 18:19:23--  https://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-6.noarch.rpm
Resolving dl.fedoraproject.org (dl.fedoraproject.org)... 209.132.181.26, 209.132.181.23, 209.132.181.24, ...
Connecting to dl.fedoraproject.org (dl.fedoraproject.org)|209.132.181.26|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 14432 (14K) [application/x-rpm]
Saving to: ‘epel-release-7-6.noarch.rpm.1’

100%[===================================================================================================================>] 14,432      37.2KB/s   in 0.4s   

2016-04-21 18:19:26 (37.2 KB/s) - ‘epel-release-7-6.noarch.rpm.1’ saved [14432/14432]

[root@localhost ~]# rpm -ivh epel-release-7-6.noarch.rpm
warning: epel-release-7-6.noarch.rpm: Header V3 RSA/SHA256 Signature, key ID 352c64e5: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:epel-release-7-6                 ################################# [100%]

```

安装好该RPM后，事实上我们已经有了第一个软件仓库，我们还需要一个。

手工创建并编辑/etc/yum.repo.d/CentOS-base.repo，并将如下内容写入该文件：

```
[base]
name=CentOS-5-Base
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever5&arch=$basearch&repo=os
#baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
baseurl=http://ftp.sjtu.edu.cn/centos/7/os/$basearch/
gpgcheck=0
gpgkey=http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-7
#released updates
[updates]
name=CentOS-$releasever – Updates
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates
baseurl=http://ftp.sjtu.edu.cn/centos/7/updates/$basearch/
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
[extras]
name=CentOS-$releasever – Extras
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras
baseurl=http://ftp.sjtu.edu.cn/centos/7/extras/$basearch/
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
[centosplus]
name=CentOS-$releasever – Plus
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus
baseurl=http://ftp.sjtu.edu.cn/centos/7/centosplus/$basearch/
gpgcheck=0
enabled=0

```
编辑完该文件后，可以执行如下命令，导入相关的key

```
rpm --import http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-7
```

至此创建软件仓库的任务完成，不妨执行以下命令检验下效果：

```
yum install -y atop screen 
```

##安装KVM相关的Packages

安装包之前，不妨检查是否打开VT-D，CPU不支持，安装后面的包也只能徒劳无功，方法如下：

```
[root@localhost mirror]# cat /proc/cpuinfo |grep -Ei "vmx|svm" | wc -l
24

- vmx is for Intel processors 
- svm is for AMD processors 
```

intel的CPU，cpuinfo中会输出vmx，而AMD的CPU会输出svm，如果并没有任何信息出现，那么，需要从BIOS中打开 VT-D（Intel Virtualization Technology for Directed I/O）。

接下来，我们可以安装KVM虚拟化相关的包了。

```
yum install -y  lvm2 qemu-kvm libvirt virt-install bridge-utils virt-install
yum -y install kvm virt-manager   xauth dejavu-lgc-sans-fonts
yum  -y install   libvirt-python libguestfs-tools l virt-viewer
```

包安装完毕后，启动libvirtd服务，并将该服务放入开机启动选项：

```
systemctl start libvirtd
systemctl enable libvirtd
```

安装好之后，不妨执行如下命令来检查相关的包确实已经安装完毕：

```
[root@localhost network-scripts]# sudo virsh -c qemu:///system list
 Id    Name                           State
----------------------------------------------------
```

##设置网络

默认情况下，创建出来的虚拟机只能访问同一个Server上的其它虚拟机，如果向让VM跳出这个限制，访问自己的LAN，必须设置Network Bridge。

我们第一步设置的网络就要修改：

修改前，网络如下：

```
[root@localhost sysconfig]# cat /etc/sysconfig/network-scripts/ifcfg-enp6s0f0 
HWADDR="60:EB:69:9B:54:1C"
TYPE="Ethernet"
BOOTPROTO="static"
IPADDR="10.16.17.196"
NETMASK="255.255.255.0"
GATEWAY="10.16.17.254"
DNS1="114.114.114.114"
IPV4_FAILURE_FATAL="no"
NAME="enp6s0f0"
UUID="569b59ae-0201-41b3-96ee-2cfd063485ae"
ONBOOT="yes"
```

调整之后：

```
[root@localhost network-scripts]# cat ifcfg-enp6s0f0 
HWADDR="60:EB:69:9B:54:1C"
TYPE="Ethernet"
BOOTPROTO="static"
#IPADDR="10.16.17.196"
#NETMASK="255.255.255.0"
#GATEWAY="10.16.17.254"
#DNS1="114.114.114.114"
IPV4_FAILURE_FATAL="no"
NAME="enp6s0f0"
UUID="569b59ae-0201-41b3-96ee-2cfd063485ae"
ONBOOT="yes"
BRIDGE=kvmbr0

[root@localhost network-scripts]# cat ifcfg-kvmbr0 
DEVICE="kvmbr0"
#HWADDR="60:EB:69:9B:54:1C"
TYPE="BRIDGE"
BOOTPROTO="static"
IPADDR="10.16.17.196"
NETMASK="255.255.255.0"
GATEWAY="10.16.17.254"
DNS1="114.114.114.114"
IPV4_FAILURE_FATAL="no"
ONBOOT=yes

```

添加网路转发功能：

```
[root@localhost ~]# echo "net.ipv4.ip_forward = 1"|sudo tee /etc/sysctl.d/99-ipforward.conf
net.ipv4.ip_forward = 1
[root@localhost ~]# sudo sysctl -p /etc/sysctl.d/99-ipforward.conf
```


##创建KVM虚拟机
首先是准备存放ISO的路径和存放虚拟机的路径,因为/home/目录最大，有2T的空间，因此不妨将虚拟机存放在该路径。事实上可以存放你希望存放的位置，只要空间足够即可。

```
mkdir /home/ISO/
mkdir /home/VM/
chmod 0777 /home/ISO
chmod 0777 /home/VM
```

将镜像文件上传到/home/ISO

```
将镜像文件放到/home/ISO/
[root@localhost home]# cd ISO/
[root@localhost ISO]# ll
total 1662448
-rwxr--r--. 1 qemu qemu 1702346752 Apr 21 18:37 VirtualStor Scaler-v6.1-210~201603211641~ecbe1bb.iso
```

接下来，就可以安装虚拟机了，我选择用命令行的方式

```
virt-install --network bridge:kvmbr0  --name VirtualStor_Scaler_6.1 --ram=32768 --vcpus=16 --disk path=/home/VM/bean_scaler_6.1,size=60  --graphics vnc,listen=0.0.0.0,port=5950  --cdrom=/home/ISO/VirtualStor\ Scaler-v6.1-210~201603211641~ecbe1bb.iso
```

* --name 指定虚拟机的名字
* --ram  指定内存大小，单位是MB
* --vcps 指定CPU的核数
* --disk path  指定虚拟机文件存放的位置， size为虚拟机的大小，单位是GB
* --graphics 需要图像化界面，因此使用vnc以及后面的参数
* --cdrom  指定安装镜像的路径

执行完上述命令后，你就看到了我们产品 VirutalStor Scaler的安装画面：

![image](/assets/RHEL_KVM/kvm-install-1.png)

接下来的事情，就比较简单了，就是安装我们的VirtualStor Scaler，我就不赘述了，最终的结果一定是
![image](/assets/RHEL_KVM/kvm-install-2.png)


##PCI直通（pass through）准备工作

事实上，安装VirtualStor Scaler需要花费很长的时间，我们可以利用这段时间来做PCI设备的直通，当虚拟机安装好了之后，重启RHEL物理机来设置PCI直通。


直通之前，我们不妨先看下我们要直通的设备，目标是RAID卡。我们的RHEL物理机上有以下块设备

```
[root@localhost ISO]# lsbk
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda               8:0    0  10.9T  0 disk 
└─sda1            8:1    0     5T  0 part 
sdb               8:16   0   8.2T  0 disk 
sdc               8:32   0   372G  0 disk 
├─sdc1            8:33   0   500M  0 part 
└─sdc2            8:34   0 371.5G  0 part 
  ├─rhel00-root 253:2    0    50G  0 lvm  
  ├─rhel00-home 253:3    0 297.9G  0 lvm  
  └─rhel00-swap 253:4    0  23.6G  0 lvm  
sdd               8:48   0   2.7T  0 disk 
├─sdd1            8:49   0     1M  0 part 
├─sdd2            8:50   0   500M  0 part /boot
└─sdd3            8:51   0   2.7T  0 part 
  ├─rhel-root   253:0    0    50G  0 lvm  /
  ├─rhel-swap   253:1    0  23.6G  0 lvm  [SWAP]
  └─rhel-home   253:5    0   2.7T  0 lvm  /home
```

其中sdd是我们RHEL系统盘，（其中sdc看到也有rhel的启动信息，这是安装过程中，我曾误安装到SSD上，不要管它），其余的10.9T的RAID5和8.2T的RAI5，以及372G的SSD RAID，在通过RAID与主板相连。这是我们直通的目标，我们希望KVM虚拟机可以直接操作这三组RAID。

要想直通，需要修改grub信息，修改/etc/default/grub的GRUB\_CMDLINE_LINUX

![image](/assets/RHEL_KVM/grub.png)

```
[root@localhost default]# cat /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="rd.lvm.lv=rhel/root crashkernel=auto  rd.lvm.lv=rhel/swap vconsole.font=latarcyrheb-sun16 vconsole.keymap=us rhgb quiet intel_iommu=on vfio_iommu_type1.allow_unsafe_interrupts=1  pci-stub.ids=1000:0079"
GRUB_DISABLE_RECOVERY="true"

```

注意，反色的部分，都是新加入的内容。首先，Intel设备为了支持Pass Through需要首先设置

```
intel_iommu=on vfio_iommu_type1.allow_unsafe_interrupts=1
```
这部分内容相对是死的，比较好。最后面的pci-stub.ids=后面的数字需要和真正的PCI设备对应上。

我们以RAID卡为例

```
lspci -vvvnn |less
```
从中搜索RAID可以得到如下内容：

```
04:00.0 RAID bus controller [0104]: LSI Logic / Symbios Logic MegaRAID SAS 2108 [Liberator] [1000:0079] (rev 05)
        Subsystem: Dell PERC H700 Adapter [1028:1f16]
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr+ Stepping- SERR+ FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0, Cache Line Size: 256 bytes
        Interrupt: pin A routed to IRQ 30
        Region 0: I/O ports at a000 [size=256]
        Region 1: Memory at df47c000 (64-bit, non-prefetchable) [size=16K]
        Region 3: Memory at df4c0000 (64-bit, non-prefetchable) [size=256K]
        Expansion ROM at df480000 [disabled] [size=256K]
        Capabilities: [50] Power Management version 3
                Flags: PMEClk- DSI- D1+ D2+ AuxCurrent=0mA PME(D0-,D1-,D2-,D3hot-,D3cold-)
                Status: D0 NoSoftRst+ PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [68] Express (v2) Endpoint, MSI 00
                DevCap: MaxPayload 4096 bytes, PhantFunc 0, Latency L0s <64ns, L1 <1us
                        ExtTag+ AttnBtn- AttnInd- PwrInd- RBE+ FLReset+
                DevCtl: Report errors: Correctable+ Non-Fatal+ Fatal+ Unsupported-
                        RlxdOrd+ ExtTag- PhantFunc- AuxPwr- NoSnoop+ FLReset-
                        MaxPayload 128 bytes, MaxReadReq 512 bytes
```

注意第一行的［1000:0079］就是需要添加到/etc/default/grub中神秘数字pci-stub.ids=1000:0079。

同时也要注意04:00.0这个指引了PCI设备的标号，通过它，我们也可以得到pci设备更多信息

```

[root@localhost ~]# virsh nodedev-list | grep pci
pci_0000_00_00_0
pci_0000_00_01_0
pci_0000_00_02_0
pci_0000_00_03_0
pci_0000_00_07_0
pci_0000_00_09_0
pci_0000_00_13_0
pci_0000_00_14_0
pci_0000_00_14_1
pci_0000_00_14_2
pci_0000_00_14_3
pci_0000_00_1a_0
pci_0000_00_1a_1
pci_0000_00_1a_2
pci_0000_00_1a_7
pci_0000_00_1c_0
pci_0000_00_1d_0
pci_0000_00_1d_1
pci_0000_00_1d_2
pci_0000_00_1d_7
pci_0000_00_1e_0
pci_0000_00_1f_0
pci_0000_00_1f_2
pci_0000_00_1f_3
pci_0000_04_00_0
pci_0000_05_00_0

[root@localhost ~]#  virsh nodedev-dumpxml   pci_0000_04_00_0 
<device>
  <name>pci_0000_04_00_0</name>
  <path>/sys/devices/pci0000:00/0000:00:07.0/0000:04:00.0</path>
  <parent>pci_0000_00_07_0</parent>
  <driver>
    <name>megaraid_sas</name>
  </driver>
  <capability type='pci'>
    <domain>0</domain>
    <bus>4</bus>
    <slot>0</slot>
    <function>0</function>
    <product id='0x0079'>MegaRAID SAS 2108 [Liberator]</product>
    <vendor id='0x1000'>LSI Logic / Symbios Logic</vendor>
    <pci-express>
      <link validity='cap' port='0' speed='5' width='8'/>
      <link validity='sta' speed='5' width='8'/>
    </pci-express>
  </capability>
</device>
```

从描述device的xml中不难看出，1000是vendor信息，即设备提供商的标号，0x0079是产品标号，这两个结合在一起，标示了LSI公司的MegaRAID SAS 2108这个PCI设备。


/etc/default/grub仅仅是grub配置文件的模版，需要执行如下命令修改真正的grub文件

```
[root@localhost default]# grub2-mkconfig -o /boot/grub2/grub.cfg 
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-123.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-123.el7.x86_64.img
Found linux image: /boot/vmlinuz-0-rescue-91af03da51e94cbebc9957e0fcba297b
Found initrd image: /boot/initramfs-0-rescue-91af03da51e94cbebc9957e0fcba297b.img
Found Red Hat Enterprise Linux Server release 7.0 (Maipo) on /dev/mapper/rhel00-root
done
```

执行完毕之后，需要等待VirtualStor的虚拟机安装完毕，待其安装完毕后，重启机器，是grub配置生效，从而开始直通PCI设备。

```
reboot
```


注意，本小节是PCI设备直通的必须要做的动作，如果不做这些动作，添加PCI设备之后，启动KVM虚拟机就会有如下的报错信息：

![image](/assets/RHEL_KVM/vfio-pci-error.png)

曾经花费了很多的力气解决这个vfio-pic could not be initialized的错误。

## PCI设备直通给虚拟机

系统重启完毕后，需要首先调用virt-manager,调出图形化界面，暂时先不要启动虚拟机，先添加PCI设备RAID卡。

```
virt-manager
```


![image](/assets/RHEL_KVM/add-pci-1.png)

![image](/assets/RHEL_KVM/add-pci-2.png)

![image](/assets/RHEL_KVM/add-pci-3.png)

![image](/assets/RHEL_KVM/add-pci-4.png)

注意第三张图中的MegaRAID SAS前面的标号0000:04:00:0和lspci -vvvnn行首的标号一致，第四张图中，添加完毕，在左边栏，看到的PCI 0000:04:00.0也是同一个标号。

这时候，就可以点击启动虚拟机了，虚拟机启动之后，就可以看到直通进去的3个RAID了：

![image](/assets/RHEL_KVM/final-result.png)




	



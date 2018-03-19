---
title: flannel源码分析
date: 2018-03-13 09:55:47
tags: [flannel, vxlan]
---

flannel是coreos开源的一个容器网络解决方案。[代码地址](https://github.com/coreos/flannel)

本文尝试对flannel的核心代码进行分析，需要对docker和linux网络通信有一些基础的了解。

<!--more-->

#### 代码简要分析

```golang
// 入口main.go
main
    LookupExtIface： 输入接口名称或IP，输出*backend.ExternalInterface。主要通过接口名称或是IP获取接口实体（net.Interface）、接口的地址等
    newSubnetManager： 输入无，输出一个subnet.Manager。只是获取了一个etcdv2的实体。此时并未直接和etcd通信
        ReadSubnetFromSubnetFile： 如果有的话，从配置文件（默认是/run/flannel/subnet.env）中读取子网的信息。是为了flanneld进程重启，能够直接从本地恢复，不用从etcd读取。
    shutdownHandler： 信号SIGNAL处理的协程
    mustRunHealthz：注册检查flannel是否健康的http服务
    getConfig：从etcd中读取配置（类似这样的配置{ "Network": "1.1.0.0/16", "Backend": { "Type": "vxlan", "VNI": 18888 } }）
    bm.GetBackend：获取后端实体，为后面服务
    RegisterNetwork：只分析vxlan的实现。
        newVXLANDevice：创建vxlan设备。包括创建vxlan的虚拟接口flannel.18888
        be.subnetMgr.AcquireLease：尝试从etcd中分配一个子网。整体逻辑是能复用的子网复用，不能复用子网的新租用。同时子网的释放也在此处进行（存疑）
    network.SetupAndEnsureIPTables：下发iptables表项。暂时略过
    WriteSubnetFile： 和ReadSubnetFromSubnetFile是对称操作
    bn.Run：后端实体的运行协程
      subnet.WatchLeases：监控子网ip租期的协程。里面不停调用下级函数。
          sm.WatchLeases：
            m.registry.watchSubnets：读取etcd中相关的key为"/coreos.com/subnets"(存疑）的value。每分配掉一个子网或是释放了一个子网则产生事件
      nw.handleSubnetEvents：对事件进行处理的协程。
         如果是子网分配事件subnet.EventAdded，当前操作系统下三条表项
            arp：对端子网（也就是产生事件的子网）网关的mac地址是对端vtep的mac地址（linux操作系统中看到的flannel.18888那个接口的mac）
            fdb: 端口-mac表。要到对端vtep的mac地址必须从flannel配置的外网端口出去
            路由表：到对端子网下一跳是对端子网网关
          如果是子网释放事件subnet.EventRemoved，则是相反的操作（略过）
```


#### flannel启动前相关的表项

```bash
[root@centos2 ~]# ip ad
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:3a:f2:c9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.31.27/24 brd 192.168.31.255 scope global enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe3a:f2c9/64 scope link
       valid_lft forever preferred_lft forever
[root@centos2 ~]#
[root@centos2 ~]# ip route
default via 192.168.31.1 dev enp0s3
169.254.0.0/16 dev enp0s3 scope link metric 1002
192.168.31.0/24 dev enp0s3 proto kernel scope link src 192.168.31.27
[root@centos2 ~]# ip nei
192.168.31.1 dev enp0s3 lladdr 48:7b:6b:d6:10:c4 STALE
192.168.31.116 dev enp0s3 lladdr 08:00:27:63:bf:77 REACHABLE
192.168.31.121 dev enp0s3 lladdr 48:4d:7e:df:e4:6d REACHABLE
[root@centos2 ~]# bridge fdb
33:33:00:00:00:01 dev enp0s3 self permanent
01:00:5e:00:00:01 dev enp0s3 self permanent
33:33:ff:3a:f2:c9 dev enp0s3 self permanent
```


#### flannel启动后相关的表项
```bash
[root@centos2 ~]# ip ad
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:3a:f2:c9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.31.27/24 brd 192.168.31.255 scope global enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe3a:f2c9/64 scope link
       valid_lft forever preferred_lft forever
3: flannel.18888: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN
    link/ether ea:27:46:35:e7:bb brd ff:ff:ff:ff:ff:ff
    inet 1.1.9.0/32 scope global flannel.18888
       valid_lft forever preferred_lft forever
    inet6 fe80::e827:46ff:fe35:e7bb/64 scope link
       valid_lft forever preferred_lft forever
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP
    link/ether 02:42:05:f9:c5:a4 brd ff:ff:ff:ff:ff:ff
    inet 1.1.9.1/24 brd 1.1.9.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:5ff:fef9:c5a4/64 scope link
       valid_lft forever preferred_lft forever
6: veth477d1b3@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master docker0 state UP
    link/ether b6:7f:1f:4f:ab:bf brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::b47f:1fff:fe4f:abbf/64 scope link
       valid_lft forever preferred_lft forever
[root@centos2 ~]# ip route
default via 192.168.31.1 dev enp0s3
1.1.9.0/24 dev docker0 proto kernel scope link src 1.1.9.1
# 新增的ROUTE表项
1.1.95.0/24 via 1.1.95.0 dev flannel.18888 onlink
169.254.0.0/16 dev enp0s3 scope link metric 1002
192.168.31.0/24 dev enp0s3 proto kernel scope link src 192.168.31.27
[root@centos2 ~]# ip nei
192.168.31.1 dev enp0s3 lladdr 48:7b:6b:d6:10:c4 STALE
192.168.31.116 dev enp0s3 lladdr 08:00:27:63:bf:77 REACHABLE
192.168.31.121 dev enp0s3 lladdr 48:4d:7e:df:e4:6d REACHABLE
# 新增的ARP表项
1.1.95.0 dev flannel.18888 lladdr 7e:b8:53:ac:90:d3 PERMANENT
[root@centos2 ~]#  bridge fdb
33:33:00:00:00:01 dev enp0s3 self permanent
01:00:5e:00:00:01 dev enp0s3 self permanent
33:33:ff:3a:f2:c9 dev enp0s3 self permanent
# 新增的FDB表项
7e:b8:53:ac:90:d3 dev flannel.18888 dst 192.168.31.116 self permanent
33:33:00:00:00:01 dev docker0 self permanent
01:00:5e:00:00:01 dev docker0 self permanent
33:33:ff:f9:c5:a4 dev docker0 self permanent
02:42:05:f9:c5:a4 dev docker0 master docker0 permanent
02:42:05:f9:c5:a4 dev docker0 vlan 1 master docker0 permanent
b6:7f:1f:4f:ab:bf dev veth477d1b3 master docker0 permanent
b6:7f:1f:4f:ab:bf dev veth477d1b3 vlan 1 master docker0 permanent
33:33:00:00:00:01 dev veth477d1b3 self permanent
01:00:5e:00:00:01 dev veth477d1b3 self permanent
33:33:ff:4f:ab:bf dev veth477d1b3 self permanent
```
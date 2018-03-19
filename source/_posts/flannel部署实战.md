---
title: flannel部署实战
date: 2018-03-12 18:17:36
tags: [centos, flannel, vxlan, 网络]
---

本文基于virtualbox和CentOS-7-x86_64-Minimal-1708.iso完成flannel的部署和测试

<!--more-->

## 操作系统的安装和配置

1. 下载centos iso和使用virtualbox安装，virtualbox至少需要两台虚拟机。iso推荐 [163镜像](http://mirrors.163.com/centos/7/isos/x86_64/)。
2. 配置系统centos。两台虚拟机都安装如下命令行配置好。
```bash
# 关闭firewalld
systemctl disable firewalld

# 关闭NetworkManager
systemctl disable NetworkManager

# 关闭selinux
sed -i 's/\(^\s*SELINUX=\).*/\1disabled/g' /etc/selinux/config

# 安装软件
yum install -y vim bridge-utils screen

# 配置网络
pushd /etc/sysconfig/network-scripts/ > /dev/null
IFACE=$(find -type f -name "ifcfg-e*"  | awk -F'/' '{print $NF}'  | sort | sed -n '1p')
sed -i 's/\(^\s*BOOTPROTO=\).*/\1static/g' $IFACE
sed -i 's/\(^\s*ONBOOT=\).*/\1"yes"/g' $IFACE

echo 'IPADDR=192.168.31.27' >> $IFACE
echo 'NETMASk=255.255.255.0' >> $IFACE
echo 'GATEWAY=192.168.31.1' >> $IFACE
echo 'DNS1=223.5.5.5' >> $IFACE
popd > /dev/null

# 安装docker
# 参考链接：https://yq.aliyun.com/articles/110806
# 安装必要的一些系统工具
yum install -y yum-utils device-mapper-persistent-data lvm2
# 添加软件源信息
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 更新并安装 Docker-CE
yum makecache fast
yum -y install docker-ce
# 开启Docker服务
service docker start
# Docker开机启动
systemctl enable docker

# 配置docker加速
# 参考链接：https://yq.aliyun.com/articles/29941
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
    "registry-mirrors": ["https://xxxxx.mirror.aliyuncs.com"]
}
EOF
systemctl daemon-reload
systemctl restart docker

# 重启系统
reboot

```


## etcd和flannel的安装
### etcd的安装
1. 从 https://github.com/coreos/etcd/releases 下载合适的包，解压缩，并将etcd、etcdctl放到$PATH目录下。
推荐/sbin
2. etcd的自动发现：通过官网自动发现 https://discovery.etcd.io/new?size=2 ，得到16进制的字符串（比如ef7bb34a532601a66cbb8c08b45de6f6 ），记下这个字符串
3. 通过下面的命令行对分别对虚拟机启动etcd：
```bash
# 第1台，将XXXXX替换成第2步获取的16进制字符串
screen etcd --name etcd1 --initial-advertise-peer-urls http://192.168.31.59:2380 --listen-peer-urls http://0.0.0.0:2380 --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://192.168.31.59:2379 -discovery https://discovery.etcd.io/XXXXX --initial-cluster-state new

# 第2台，将XXXXX替换成第2步获取的16进制字符串
screen etcd --name etcd2 --initial-advertise-peer-urls http://192.168.31.232:2380 --listen-peer-urls http://0.0.0.0:2380 --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://192.168.31.232:2379 -discovery https://discovery.etcd.io/XXXXX --initial-cluster-state new

# 设置flanneld需要的键值对
etcdctl set /coreos.com/network/config '{ "Network": "1.1.0.0/16", "Backend": { "Type": "vxlan", "VNI": 18888 } }'

```
### flannel的安装
1. 从 https://github.com/coreos/flannel/releases 下载合适的包，解压缩，并将flanneld、 mk-docker-opts.sh放到$PATH目录下。推荐/sbin
2. 修改docker.service文件
```bash
sed -i '/ExecStart=/iEnvironmentFile=\/run\/docker_opts.env' /usr/lib/systemd/system/docker.service
sed -i 's/ExecStart=.*/ExecStart=\/usr\/bin\/dockerd \${DOCKER_OPT_BIP} \${DOCKER_OPT_IPMASQ} \${DOCKER_OPT_MTU}/g' /usr/lib/systemd/system/docker.service 
```
3. 调用mk-docker-opts.sh生成docker.service需要的参数并重启docker服务
```bash
mk-docker-opts.sh  && systemctl daemon-reload && systemctl restart docker
```
4. 启动flannel
```bash
screen flanneld
```
## 容器间互通
```bash
root@a26d8179ca1e:/# ip ad
# 容器ip地址为1.1.15.2
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 02:42:01:01:0f:02 brd ff:ff:ff:ff:ff:ff
    inet 1.1.15.2/24 brd 1.1.15.255 scope global eth0
       valid_lft forever preferred_lft forever
# ping其他容器（不同宿主机）
root@a26d8179ca1e:/# ping 1.1.2.2
PING 1.1.2.2 (1.1.2.2) 56(84) bytes of data.
64 bytes from 1.1.2.2: icmp_seq=1 ttl=62 time=0.346 ms
64 bytes from 1.1.2.2: icmp_seq=2 ttl=62 time=0.654 ms
^C
--- 1.1.2.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.346/0.500/0.654/0.154 ms
# ping外部主机
root@a26d8179ca1e:/# ping 192.168.31.1
PING 192.168.31.1 (192.168.31.1) 56(84) bytes of data.
64 bytes from 192.168.31.1: icmp_seq=1 ttl=253 time=0.637 ms
64 bytes from 192.168.31.1: icmp_seq=2 ttl=253 time=0.929 ms
^C
--- 192.168.31.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.637/0.783/0.929/0.146 ms
root@a26d8179ca1e:/#
```

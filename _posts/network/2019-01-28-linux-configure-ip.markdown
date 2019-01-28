---
layout: post
title: "linux configure ip"
date: 2019-01-28 21:00:00 +0800
categories: network
---

大部分操作系统默认是通过 dhcp(动态主机配制协议) 自动分配 ip、子网掩码、默认网关、dns 服务器 ip 等参数。客户机会使用这些参数完成个人的网络配置，进行网络通信

在某些时候，我们需要通过 ifconfig 命令手动配制 ip

##### 给网卡配制静态 ip
不能给不存在的网卡配置 ip 信息，所以先通过 ifconfig 命令观察一下网卡信息<br>

<img src="/images/ifconfig-1.png" width="480" height="130" />

发现有 enp1s0 网卡(是本机的副网卡)，通过下面命令给其配置 ip 并指定子网掩码
``` bash
sudo ifconfig enp1s0 192.168.43.88 netmask 255.255.255.0 up
```

再通过 ifconfig 命令查看结果

<img src="/images/ifconfig-2.png" width="480" height="130" />


##### 设置网卡别名(单网卡多 ip)
给 enp1s0 再配置两个 ip，本质上是生成了两个虚拟网卡
``` bash
sudo ifconfig enp1s0:1 11.11.11.11 netmask 255.255.255.0
sudo ifconfig enp1s0:1 99.99.99.99 netmask 255.255.255.0
```

通过 ifconfig 命令查看一下效果

<img src="/images/ifconfig-3.png" width="480" height="200" />

在设置 ip 别名或副网卡 ip 时，如果增加的是和局域网同一网段的 ip，那么除了本机外局域网内其他机器也都可以 ping 通这个 ip。反之 ip，那么就只有本机可以 ping 通

<img src="/images/ifconfig-4.png" width="480" height="300" />


##### 让配制永久生效
像上面这样直接用 ifconfig 命令配置的信息在关闭或重启后就会丢失。可以通过下列方式使其永久生效

1. 将 config 配置命令写到自启动文件 /etc/rc.local 中，这样每次开机时会自动执行文件中的配置命令

2. 修改 /etc/network/interfaces 文件

``` bash
# interfaces(5) file used by ifup(8) and ifdown(8)
auto lo
iface lo inet loopback

auto enp1s0 enp1s0:0 enp1s0:1
iface enp1s0 inet static
    address 192.168.43.87
    netmask 255.255.255.0
iface enp1s0:0 inet static
    address 11.11.11.11
    netmask 255.255.255.0
iface enp1s0:1 inet static
    address 99.99.99.99
    netmask 255.255.255.0
```

保存上面的配置后执行 sudo /etc/init.d/networking restart 命令使更新的配置文件生效


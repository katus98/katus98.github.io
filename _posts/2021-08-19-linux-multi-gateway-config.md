---
title: Linux服务器多网卡多网关配置解决方案
author: katus
date: 2021-08-19 11:14:51 +0800
categories: [Operations]
tags: [Linux, Computer Network]
typora-root-url: ..
---

> 本文为本人尝试修复实验室Linux服务器网络的过程中的经验总结，如有错误，还望不吝赐教。

## 一、网络情况

### 软硬件条件

#### 硬件条件

+ 服务器四台
+ 每台服务器有万兆网卡2块（光纤口）、千兆网卡2块（RJ45）
+ 交换机、路由器、网线若干
+ 入户网线1条

#### 软件条件

+ 服务器操作系统：CentOS 8
+ 入户网线可以通过L2TP登录校园网进而访问互联网
+ 校园网内的固定IP四个

### 预期效果

+ 四台服务器之间通过万兆网卡组合局域网，网段192.168.1.0\24
+ 四台服务器的其中一个千兆网卡各持有一个校园网固定IP，用于远程访问，网段10.79.224.0/21
+ 四台服务器均需要访问互联网，由于服务器本身的L2TP拨号需要占用四个上网账号且不稳定，计划将L2TP的上网账号配置在路由器，形成新的局域网，进而上网，局域网网段192.168.10.0/24

## 二、网络拓扑

### 网络拓扑图

![网络拓扑图](/assets/img/post/2021-08-19-linux-multi-gateway-config-1.png)

### 注意事项

+ 路由器需要支持L2TP拨号功能，而且将校园网上网账号配置在其中。
+ 路由器需要支持DHCP分配IP时，将指定IP与指定的MAC地址绑定的功能，方便后期的服务器网卡配置。
+ 虽然采用无线路由器，但是实际上不使用其无线功能，均采用有线连接，以保证稳定性和安全性。
  如果路由器的LAN口有限，可以续接交换机，对网络拓扑没有影响。

## 三、服务器配置

> 针对每一台服务器的配置均相同，仅有IP地址不同，以td0服务器的配置为例。
> 所有的配置均使用root用户进行为宜。

### 1.路由配置

- 查看路由

  ```shell
  # 配置前通过route命令查看路由, 仅增加缺失的路由
  route
  ```

- 添加路由

  ```shell
  # 万兆光纤内网
  route add -net 192.168.1.0 netmask 255.255.255.0 dev eno1np0
  # 千兆内网
  route add -net 192.168.10.0 netmask 255.255.255.0 dev eno3
  # 千兆外网
  route add -net 10.79.224.0 netmask 255.255.248.0 dev eno4
  # 默认网关
  route add default gw 192.168.10.1 dev eno3
  ```

+ 路由配置结果

![](/assets/img/post/2021-08-19-linux-multi-gateway-config-2.png)

> 需要保证图片中default、10.79.224.0、192.168.1.0、192.168.10.0四行内容。
> 172.17.0.0是Docker容器的网络，无需关心。
> 192.168.122.0是服务器维护的专用网络，也无需修改。

### 2.路由表配置

- 新增路由表

  ```shell
  # 新增lan_q与wan_q两个路由表
  echo "101 lan_q" >> /etc/iproute2/rt_tables
  echo "102 wan_q" >> /etc/iproute2/rt_tables
  # 也可以使用vi命令进行文件编辑
  ```

- 配置路由表内容

  ```shell
  # 定义路由表规则
  ip route add default via 192.168.10.1 dev eno3 table lan_q
  ip route add default via 10.79.231.254 dev eno4 table wan_q
  ```

- 配置IP规则

  ```shell
  # 不同的IP访问流量走不同的路由表(路由规则)
  ip rule add from 192.168.10.3 table lan_q
  ip rule add from 10.79.231.83 table wan_q
  ```

### 3.其他必要命令（非配置内容）

```shell
# 查看指定路由表的路由规则
ip route list table <table_name>
# 清空指定路由表的路由规则
ip route flush table <table_name>
# 查看路由和IP规则
ip route show
ip rule show
# 网络配置重载
nmcli c reload
```

+ 命令参考：[Linux 路由表设置 之 route 指令详解](https://www.cnblogs.com/baiduboy/p/7278715.html)

### 4.额外配置

- 配置主机名（略）

### 5.检验

#### 连通性检验

```shell
ping 192.168.1.7
ping 10.79.231.86
ping www.baidu.com
```

#### IP规则检验

- 在校园网环境中通过ssh直连固定IP，查看是否可以连接。

## 四、额外说明

如以本文为参考，请参照自己的网络环境中的IP分配，勿盲从。

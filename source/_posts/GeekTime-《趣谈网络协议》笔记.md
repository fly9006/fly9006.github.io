---
title: GeekTime -《趣谈网络协议》笔记
date: 2019-05-01 19:32:09
tags:
- 笔记
- 极客时间
- 网络协议
categories:
- 极客时间
---

# 趣谈网络协议

### 01 为什么要学习网络协议

**协议三要素**：

- 语法，就是一段内容要符合一定的规则和格式
- 语义，就是一段内容要代表某种意义
- 顺序，定义先干啥，后干啥

**大致的协议分层框架**

- 应用层

  DHCP HTTP HTTPS DNS RPC GTP P2P RTMP

- 传输层

  TCP UDP

- 网络层

  IP ICMP OSPF BGP

- 链路层

  ARP VLAN STP

- 物理层

### 02 为什么要学习网络协议

- 只要是在网络上跑的包，都是完整的。可以有下层没上层，但绝不可能有上层没下层
- 一个原则：对TCP协议来说，三次握手也好，重试也好，只要想发出去包，就要有IP层和MAC层，不然发不出去

### 03 最熟悉又陌生的命令行

- 如何查看当前操作系统的IP地址
  - win系统：ipconfig
  - linux系统：ifconfig，ip addr
- IP是地址，又定位功能；MAC是身份证，无定位功能
- CIDR可以用来片段是不是本地人
- IP分公有IP和私有IP

### 04 DHCP与PXE：IP是怎么来的，又是怎么没的？

- 如何配置IP地址
  - net-tools
  - iproute2
- 动态主机配置协议（Dynamic Host Configuration Protocol），简称DHCP
- DHCP工作方式
  - DHCP Discover
  - DHCP Server
  - DHCP Offer
- IP地址的收回和续租
- 预启动执行环境（PXE，Pre-boot Execution Environment）

### 05 从物理层到MAC层

- MAC，全称是Medium Access Control，即媒体访问控制
- 通过ARP协议，已知IP地址获取MAC地址

### 06 交换机与VLAN

- STP协议，Spanning Tree Protocol，最小生成树
  - Root Bridge，根交换机
  - Designated Bridges，指定交换机
  - Bridge Protocol Data Units（BPDU），网桥协议数据单元
  - Priority Vector，优先级向量
- 利用STP协议解决环路问题，将又环路的图变成没有环的树
- VLAN，虚拟局域网，用来解决广播和安全问题

### 07 ICMP与ping

- ICMP全称Internet Control Mesaage Protocol，互联网控制报文协议
- ping就是查询报文，是一种主动请求，并且获得主动应答的ICMP协议
- Traceroute，使用的是差错报文

### 08 世界那么大，我想出网关

- 网关往往是一个路由器，是一个三层转发的设备，理由如何寻找下一跳的规则
- NAT，Network Address Translation
- 不改变IP地址的网关。称为转发网关；改变IP地址的网关，称为NAT网关

### 09 路由协议

- 动态路由算法
- 距离矢量路由算法，distance vector routing
  - BGP，Border Gateway Protocol，外网路由协议
- 链路状态路由算法，link state routing
  - OSPF，Open Shortest Path First，开放式最短路径优先，称为IGP，Interior Gateway Protocol，内部网关协议
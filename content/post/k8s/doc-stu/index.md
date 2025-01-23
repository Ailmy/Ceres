---
title: "K8s基础学习笔记"
description: 
date: 2025-01-21T11:05:50+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories:
  - k8s
tags:
    - 学习笔记
---
## 基础
##### 基础设施的迭代
* IAAS 基础设施是一个服务
> 可以理解为 IAAS 是一个基础设施的服务，提供了基础设施的服务，比如虚拟机，网络，存储等等，比如基于OpenStack，云服务器，云硬盘，云数据库等等。
* PAAS 平台是一个服务
> 可以理解为 PAAS 是一个平台的服务，提供了平台的服务，比如开发平台，运行平台，测试平台等等，比如Kubernetes,Swarm,Apache Mesos等等。
* SAAS 软件是一个服务
> 租户隔离共用一个软件

##### kubernetes优势
* 服务发现和负载均衡
* 存储编排（添加任何本地或云存储系统）
* 自动部署和回滚
* 自动分配CPU和内存 -弹性伸缩
* 自我修复
* Secret和配置管理
* 大型规模的支持
* 开源

### Pod概念
Pod是一个逻辑概念，是一组容器的集合，是最小调度单位

一个Pod内的容器与Pause容器共享名字空间（Network，PID，IPC），**Pause**容器是一个占位容器，不做任何事情，只是为了让Pod内的其他容器能够共享网络和存储卷等资源。

### Kubernetes网络模型

#### CNI (Container Network Interface)
CNI是一个规范，定义了容器网络的接口，Kubernetes使用CNI来实现容器网络。

![CNI规范](assets/CNI规范.png)
![CNI第三方插件对比](assets/CNI第三方插件对比.png)  
![网络模型名词解释](assets/网络模型名词解释.png)
![封装网络示例图](assets/封装网络示例图.png)
![未封装网络示例图](assets/未封装网络示例图.png)

#### calico架构
##### calico架构图
![calico架构](assets/calico架构.png)
1.Felix 
* 管理网络接口
* 编写路由
* 编写ACL（访问控制列表）
* 报告状态

2.bird
* BGP Client将通过BGP协议广播告诉剩余calico节点，从而实行网络互通

3.confd
* 通过监听etcd以了解BGP配置和全局默认值的更改，Confd根据ETCD中的数据的更新，动态生成BIRD配置文件，当配置文件更改时，confd触发BIRD重新加载新文件

##### calico网络模型

###### VXLAN
VXLAN 即Virtual eXtensible Local Area Network，是一种虚拟化技术，用于在二层网络上建立逻辑二层网络，从而实现跨主机的二层网络通信。
是Linux本身支持的一种网络虚拟化技术，VXLAN可以完全在内核态实现封装和解封装，从而通过隧道机制，构建出覆盖网络



数据包封包：在vxlan设备上将pod发来的数据包源、目的mac替换为本机vxlan网卡和对端节点vxlan网卡的mac，外层udp目的ip地址根据路由和对端vxlan的mac查fdb表获取

优势：只要k8s三层节点互通，可以跨网段，对主机网关路由没有特殊要求，各个node节点通过vxlan设备实现基于三层的二层互通

> 基于三层的“二层”通信，即vxlan包封装在udp数据包中，要求udp在k8s节点间三层可达；二层即vxlan封包的源mac地址和目的mac地址是自己的vxlan设备mac和对端vxlan设备mac实现通讯

缺点：封包和解包会存在一定的性能损耗

![vxlan示例图](assets/vxlan示例图.png)

vxlan底层封装模型
![vxlan底层封装模型](assets/vxlan底层封装模型.png)

###### 在calico中使用vxlan
![在calico中使用vxlan](assets/在calico中使用vxlan.png)

##### IPIP

数据包封包：在tunl0设备上将pod发来的数据包源、目的mac替换为本机tunl0网卡和对端节点tunl0网卡的mac，外层数据包目的根据路由得到

解包和封包会存在一定性能损耗

* Linux原生支持
* IPIP隧道的工作原理是将源主机的IP数据包封装在一个新的数据包中，新的IP数据包的目的地址是隧道的另一端，在隧道的另一端，接受方将解封原始IP数据包，并将其传递给目的主机

> IPIP隧道可以在不同网络之间建立连接，例如在IPv4网络和IPv6网络之间建立连接，或者在不同的IPv4网络之间建立连接

![IPIP示例图](assets/IPIP示例图.png)
![IPIP封装模型](assets/IPIP封装模型.png)
![使用IPIP](assets/使用IPIP.png)

##### BGP
边界网关协议（Border Gateway Protocol，BGP）是互联网上一个核心的去中心化自治路由协议，它通过维护IP路由表或“前缀”表来实现自治系统之间的可达性，属于矢量路由协议

![BGP示例图](assets/BGP示例图.png)

BGP
* 数据包封包：不需要进行数据包封包
* 优点： 不用封包解包，通过BGP协议可实现pod网络在主机间的三层可达
* 缺点：跨网段时，配置较为复杂网络要求高，主机网关路由也需要充当BGP Speaker

配置方法
![BGP使用](assets/BGP使用.png)

## Kubernetes安装

### 安装方式
#### kubeadm
组件通过容器化方式运行

优势
* 简单
* 自愈

缺点
* 无法自定义

#### 二进制安装
组件变成系统进程的方式运行

优点
* 能够更灵活的安装集群

## K8s资源清单



# 网络基础知识


<!--more-->

# 网络

## 一、CIDR

- CIDR（Classless Inter-Domain Routing，无类域间路由选择）它消除了传统的 A 类、B 类和 C 类地址以及划分子网的概念，因而可以更加有效地分配 IPv4 的地址空间。它可以将好几个 IP 网络结合在一起，使用一种无类别的域际路由选择算法，使它们合并成一条路由从而较少路由表中的路由条目减轻 Internet 路由器的负担。

#### CIDR 记法

CIDR 还使用“斜线记法”，它又称为 CIDR 记法，即在 IP 地址后面加上一个斜线“/”，然后写上网络前缀所占的比特数（这个数值对应于三级编址中子网掩码中比特 1 的个数）。 IP 地址::={<网络前缀*>,<*主机号>}

### 二、 IPSec （Internet Protocol Security）

**学习链接**

https://support.huawei.com/enterprise/zh/doc/EDOC1100008291/370abd69

https://support.huawei.com/enterprise/zh/doc/EDOC1100041456/f2298f86

https://cshihong.github.io/2019/04/09/IPSec-VPN%E4%B9%8BIKEv2%E5%8D%8F%E8%AE%AE%E8%AF%A6%E8%A7%A3/ ike2

#### 2.1 概述

- 是一系列网络协议的集合

<img src="https://download.huawei.com/mdl/image/download?uuid=fe4b80a544094edc82693030af0b7376" alt="img" style="zoom:100%;" />

AH（Authentication Header） 验证头

- 报文头验证协议，提供数据校验功能。

ESP（Encapsulating Security Payload）封装安全载荷

- 提供数据加密功能

**比较**

- AH 和 ESP 都能提供验证功能，主要采用算法
  - MD5、SHA1、SHA2-256**（建议）**、SHA2-384、SHA2-512
- ESP 可以提供加密功能
  - DES、3DES、AES**（建议）**

#### 2.2 IKE（Internet Key Exchange）因特网密钥交换

- 可以动态协商密钥
- IKE 建立在 ISAKMP 框架之上，采用 DH（Diffie-Hellman）算法，可以提升密钥的安全性，并降低 IPSec 管理复杂度

#### 2.3 IPSec 原理介绍

- ipsec 在对等体之间建立安全联盟 SA（security association）

##### SA 由三元组标识

- SPI（security parameter index）安全参数索引： 32 位的唯一标识 SA 的数值
- 目的 IP 地址
- 使用的安全协议（AH 或 ESP）

* SA 是单向逻辑，为了建立双向连接，需要有两个安全联盟
* 另外，SA 的个数还与安全协议相关。如果只使用 AH 或 ESP 来保护两个对等体之间的流量，则对等体之间就有两个 SA，每个方向上一个。如果对等体同时使用了 AH 和 ESP，那么对等体之间就需要四个 SA，每个方向上两个，分别对应 AH 和 ESP。

**封装模式**

1. 传输模式

<img src="https://download.huawei.com/mdl/image/download?uuid=f0852c852404411f929b2fe01de30b92" alt="img" style="zoom:100%;" />

2. 隧道模式：

<img src="https://download.huawei.com/mdl/image/download?uuid=fa9bc6e3d2bd4b4da98d3c997582044c" alt="img" style="zoom:100%;" />

##### IKE 与 IPSec 关系

<img src="https://download.huawei.com/mdl/image/download?uuid=e9a70ce686f1456bbe060cd6234b7d7e" alt="img" style="zoom:100%;" />

- 对等体之间建立一个 IKE SA 完成身份验证和密钥信息交换后，在 IKE SA 的保护下，根据配置的 AH/ESP 安全协议等参数协商出一对 IPSec SA。此后，对等体间的数据将在 IPSec 隧道中加密传输


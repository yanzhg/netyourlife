# 深入浅出 VXLAN 技术原理与实战（华为 \+ Cisco 双厂商配置）

**前言**

随着云计算、虚拟化、多租户数据中心的普及，传统 VLAN 网络暴露出致命短板：**4094 数量上限、无法跨三层二层互通、虚拟机迁移受限**。

为了解决上述问题，业界推出了 **VXLAN（Virtual Extensible LAN，虚拟可扩展局域网）**。

目前 VXLAN 已经成为**数据中心主流overlay虚拟化技术**，几乎所有云数据中心、私有云、虚拟化组网均基于 VXLAN 实现。

本篇文章带你从零吃透 VXLAN：

✅ 为什么需要 VXLAN？

✅ VXLAN 核心原理、报文封装、核心组件

✅ 二层 VXLAN 经典组网场景解析

✅ **华为、Cisco 双厂商静态 VXLAN 完整配置**

✅ 命令验证 \+ 排错思路

---

## 一、传统 VLAN 的四大痛点

1\. **数量不足**：VLAN 12位标签，最大 4094 个，无法满足多租户云场景；

2\. **二层域受限**：VLAN 是二层技术，无法跨越三层网关；

3\. **虚拟机迁移困难**：VM 跨主机、跨机房迁移需要二层网络贯通，传统网络做不到；

4\. **大二层环路风险**：传统二层依赖 STP，链路阻塞、带宽利用率低。

---

## 二、VXLAN 核心技术原理

### 2\.1 什么是 VXLAN？

**VXLAN 是一种三层叠加二层的网络虚拟化技术（Overlay）**。

简单理解：**把传统二层以太网报文，封装在 UDP 报文中，通过底层三层网络传输。**

物理网络只负责三层转发，上层虚拟网络负责业务隔离，实现**物理与逻辑网络解耦**。

### 2\.2 VXLAN 三大核心组件

**1\. VTEP（VXLAN Tunnel Endpoint）**

VXLAN 隧道端点，一般为接入/汇聚交换机。

负责：报文封装、解封装、隧道建立。

**2\. VNI（VXLAN Network Identifier）**

24 位标识符，取值范围：**0\~16777215**

作用等同于 VLAN，但是容量扩大百万倍，完美适配多租户隔离。

**3\. VXLAN Tunnel 隧道**

建立在底层三层 IP 网络之上的虚拟隧道，VTEP 之间通过隧道传输封装后的流量。

### 2\.3 VXLAN 报文封装（MAC\-in\-UDP）

从内到外封装顺序：

**原始二层帧 → VXLAN Header → UDP Header → IP Header → Ethernet Header**

关键参数：

• UDP 目的端口：**4789**（IANA 标准端口）

• VXLAN 头部携带 24bit VNI，用于租户隔离

### 2\.4 VXLAN 两种主流模式

**L2 VXLAN（二层虚拟化）**：跨三层网络实现二层互通，适合虚拟机迁移、同网段互通。

**L3 VXLAN（三层虚拟化）**：通过分布式网关实现不同 VNI、不同网段互通，大型数据中心主流。

---

## 三、实战组网场景（静态二层 VXLAN）

### 3\.1 组网拓扑与需求

**拓扑描述：**

PC1 —— Cisco SW1（VTEP1）—— 三层公网 —— 华为 SW2（VTEP2）—— PC2

**设备地址规划：**

• Cisco SW1 LoopBack0：1\.1\.1\.1/32

• 华为 SW2 LoopBack0：2\.2\.2\.2/32

• VNI：1000

• 业务网段：192\.168\.10\.0/24

**需求：**

PC1、PC2 同网段，跨越三层物理网络，通过 VXLAN 隧道实现二层直通。

---

## 四、Cisco IOS\-XE VXLAN 完整配置

设备：Cisco Catalyst 3650/3850/9300 通用配置

```Plain Text
# 进入全局配置模式
enable
configure terminal

# 配置隧道源环回口
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 no shutdown
 exit

# 指定VXLAN全局源接口
vxlan source-interface Loopback0

# 绑定VNI并指定静态对端VTEP
vxlan vni 1000
 peer 2.2.2.2
 exit

# 创建业务VLAN并关联VNI
vlan 10
 name VXLAN_VNI1000
 exit
vxlan vlan 10 vni 1000

# 接入终端接口
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit

# 底层路由（保证对端环回可达）
ip route 2.2.2.2 255.255.255.255 <下一跳IP>

```

### Cisco 验证命令

```Plain Text
show vxlan tunnel
show vxlan vni
show running-config | section vxlan
```

---

## 五、华为 VRP VXLAN 完整配置

设备：华为 CE6800/CE5800/S5735 等支持 VXLAN 设备通用

```Plain Text
# 进入系统视图
system-view

# 开启VXLAN功能
vxlan enable

# 配置环回口（隧道源地址）
interface LoopBack0
 ip address 2.2.2.2 255.255.255.255
 quit

# 配置全局隧道源地址
tunnel global source-address 2.2.2.2

# 创建VSI虚拟转发实例并绑定VNI
vsi vxlan1000
 vxlan 1000
 peer 1.1.1.1
 quit
 quit

# 接入侧接口绑定VSI
interface GigabitEthernet 0/0/1
 vtep access port
 access vsi vxlan1000
 quit

# 底层静态路由
ip route-static 1.1.1.1 255.255.255.255 <下一跳IP>

```

### 华为验证命令

```Plain Text
display vxlan tunnel
display vsi brief
display vxlan vtep
```

---

## 六、业务测试结果

PC1：192\.168\.10\.10/24

PC2：192\.168\.10\.20/24

两端终端无需网关，直接 ping 互通。

**原理验证：**

终端普通二层流量 —\> VTEP 封装 VXLAN —\> 三层网络转发 —\> 对端 VTEP 解封装 —\> 还原二层流量转发给终端。

---

## 七、静态 VXLAN 优缺点总结

### 优点

✅ 无需复杂协议、无需组播

✅ 配置简单、稳定性极高、运维压力小

✅ 适合中小型数据中心、固定站点虚拟化组网

### 缺点

❌ 所有对端 VTEP 需要手动配置，大型组网工作量巨大

❌ 无法自动发现邻居、无法自动同步 MAC 表项

---

## 八、进阶方向：EVPN\+VXLAN

生产环境中大型云数据中心**基本不使用静态 VXLAN**，而是使用 **EVPN\+VXLAN**：

• 通过 BGP EVPN 协议动态学习 VTEP、MAC、网段路由

• 无需手动配置 peer，支持大规模弹性扩缩容

• 支持分布式网关、三层互通、多活冗余

---

## 九、总结

1\. VXLAN 解决了传统 VLAN 数量不足、二层域受限的痛点，是数据中心虚拟化基石；

2\. VXLAN 核心是 **MAC\-in\-UDP 三层承载二层**；

3\. 静态 VXLAN 适合小规模场景，**华为、Cisco 配置逻辑一致、命令体系不同**；

4\. 掌握静态 VXLAN 是入门，**EVPN\+VXLAN** 是企业高阶实战必备技能。


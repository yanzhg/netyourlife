# 网络设备巡检命令大全（Cisco + 华为）

分为**通用基础巡检、接口链路、设备硬件、路由交换、安全 / 防火墙、日志告警、运维排错**七大模块，区分思科（IOS/IOS-XE/NX-OS）、华为（VRP5/VRP8），按日常巡检优先级排序，可直接现场执行。

> 说明：华为设备默认用户视图 `<>`，系统视图 `[]`；Cisco 分为用户模式 `>`、特权模式 `#`、全局配置模式 `(config)#`，**巡检仅使用用户 / 特权模式，不进配置模式**。

------

## 一、前置准备（登录与权限）

### Cisco

```
! 进入特权模式（必做，大部分巡检命令需要）
enable
```

### 华为

```
# 无需额外切换，普通用户视图即可执行巡检命令，部分硬件/诊断命令需管理员权限
```

------

## 二、基础信息巡检（设备版本、名称、运行时长、许可）

### 1. 设备版本 & 系统信息

#### Cisco

```
show version          ! 系统版本、镜像、运行时长、设备型号、CPU/内存、序列号
show run              ! 查看完整当前配置
show startup-config   ! 查看开机启动配置
show license          ! 查看许可信息、授权状态
```

#### 华为

```
display version       ! 版本、型号、运行时长、序列号
display current-configuration  ! 完整当前配置
display saved-configuration    ! 保存的启动配置
display license       ! 许可/授权信息
```

### 2. 设备名称 & 管理信息

#### Cisco

```
show running-config | include hostname
```

#### 华为

```
display sysname
```

------

## 三、硬件巡检（电源、风扇、温度、单板、模块）

### 1. 整机硬件状态

#### Cisco（交换机 / 路由器 / 防火墙通用）

```
show environment      ! 温度、风扇、电源状态（核心硬件巡检）
show module           ! 机框、业务板卡、模块状态（框式设备）
show inventory        ! 整机部件、光模块、线缆、序列号清单
show power            ! 电源单独查看
show fan              ! 风扇单独查看
```

#### 华为（交换机 / 路由器 / 防火墙通用）

```
display environment   ! 温度、风扇、电源、电压状态（核心）
display device        ! 整机、单板、子卡、模块在位/状态
display power         ! 电源详细信息
display fan           ! 风扇详细信息
display elabel        ! 电子标签、整机/部件序列号
```

### 2. 光模块 / 光功率巡检（重点，光纤链路必查）

#### Cisco

```
show interfaces transceiver detail  ! 所有光模块：收发光功率、温度、电压、模块型号
show ip interface brief | include up|down  ! 配合接口状态排查光模块
```

#### 华为

```
display transceiver information  ! 光模块基础信息
display transceiver diagnosis    ! 实时收发光功率、温度、电压（巡检核心）
```

------

## 四、接口 & 链路巡检（以太网、聚合、VLAN、状态、流量）

### 1. 接口整体状态（快速巡检所有接口）

#### Cisco

```
show ip interface brief        ! 三层接口：IP、协议状态、UP/DOWN（高频常用）
show interfaces status         ! 二层接口：状态、VLAN、速率、双工（交换机专用）
show interfaces description     ! 查看接口描述
```

#### 华为

```
display ip interface brief     ! 三层接口IP、协议状态
display interface brief        ! 所有接口汇总（二层+三层，最常用）
display interface description  ! 接口备注描述
```

### 2. 接口详细信息（速率、双工、错包、丢包、CRC）

> 重点关注：**CRC 错误、输入 / 输出错误、丢弃包、过载**，代表链路异常

#### Cisco

```
show interfaces GigabitEthernet 0/1   ! 替换为实际接口名，查看详细统计
show interfaces errors                ! 接口错误包统计汇总
```

#### 华为

```
display interface GigabitEthernet 0/0/1  ! 查看单接口详细统计
display interface error-statistics       ! 接口错误包统计
```

### 3. 链路聚合（Eth-Trunk / Port-channel）

#### Cisco

```
show etherchannel summary    ! 聚合组状态、成员接口、负载分担
```

#### 华为

```
display eth-trunk brief      ! 聚合组、成员接口、状态
```

### 4. VLAN 信息（二层交换机）

#### Cisco

```
show vlan brief              ! VLAN 列表、端口划分
show vtp status              ! VTP 协议状态（老式交换机组网）
```

#### 华为

```
display vlan brief           ! VLAN 及端口划分
display vlan                 ! 完整VLAN信息
```

### 5. 接口流量统计

#### Cisco

```
show interfaces | include input rate|output rate
```

#### 华为

```
display interface traffic
```

------

## 五、路由 & 转发巡检（三层网络、静态路由、动态路由）

### 1. 路由表（核心）

#### Cisco

```
show ip route                ! IPv4 路由表
show ipv6 route              ! IPv6 路由表（如有）
```

#### 华为

```
display ip routing-table     ! IPv4 路由表
display ipv6 routing-table  ! IPv6 路由表
```

### 2. 静态路由

#### Cisco

```
show running-config | include ip route
```

#### 华为

```
display ip routing-table protocol static
```

### 3. 动态路由协议（OSPF / RIP / BGP 主流）

#### OSPF

**Cisco**

```
show ip ospf neighbor        ! OSPF 邻居状态（重点：FULL 正常）
show ip ospf database       ! OSPF 链路状态库
show ip ospf brief
```

**华为**

```
display ospf peer brief     ! OSPF 邻居（状态Full为正常）
display ospf lsdb           ! OSPF 链路库
display ospf brief
```

#### BGP

**Cisco**

```
show ip bgp summary         ! BGP 邻居、会话状态
```

**华为**

```
display bgp peer brief      ! BGP 邻居状态
```

------

## 六、ARP & MAC 地址表（二层转发、终端接入巡检）

### 1. MAC 地址表（交换机核心）

#### Cisco

```
show mac address-table       ! 整机MAC地址表、端口对应关系
show mac address-table count ! MAC条目数量
```

#### 华为

```
display mac-address          ! MAC地址表
display mac-address total    ! MAC条目统计
```

### 2. ARP 表（IP+MAC 对应，排查地址冲突）

#### Cisco

```
show arp                     ! ARP 表项
show ip arp
```

#### 华为

```
display arp                  ! ARP 表项
display arp anti-attack      ! ARP 防攻击状态（安全巡检）
```

------

## 七、防火墙 & 安全设备专项巡检（Cisco ASA / 华为 USG）

### Cisco ASA 防火墙

```
show version
show running-config
show interface status
show conn                    ! 会话连接数
show cpu usage               ! CPU占用
show memory usage            ! 内存占用
show failover                ! 双机热备/主备状态
show logging buffer          ! 日志缓存
```

### 华为 USG 防火墙

```
display version
display current-configuration
display interface brief
display firewall session table  ! 会话表
display cpu-usage               ! CPU使用率
display memory-usage            ! 内存使用率
display hrp state               ! 双机热备HRP状态
```

------

## 八、CPU & 内存（设备性能巡检，高危项）

> 参考阈值：**CPU 持续 >80%、内存持续 >85% 属于异常**

### Cisco

```
show processes cpu           ! CPU 使用率、进程占用
show processes memory        ! 内存使用率
```

### 华为

```
display cpu-usage            ! CPU 整机/各模块占用
display memory-usage         ! 内存使用情况
```

------

## 九、日志 & 告警（故障、异常、历史记录）

### 1. 本地日志缓存

#### Cisco

```
show logging                 ! 日志配置、日志缓冲区
show logging buffer         ! 查看本地日志（告警、报错、端口上下线）
```

#### 华为

```
display logbuffer            ! 本地日志缓存（最常用）
display trapbuffer           ! 告警陷阱、系统告警
```

### 2. 系统告警 / 故障信息

#### Cisco

```
show alarms                  ! 系统告警
```

#### 华为

```
display alarm active         ! 当前活跃告警（紧急/重要告警优先处理）
display alarm history        ! 历史告警记录
```

------

## 十、高可用 / 冗余协议（VRRP/HSRP/ 堆叠 / 集群）

### 1. VRRP（华为主流）/ HSRP（Cisco 主流）

#### Cisco HSRP

```
show standby brief           ! HSRP 主备状态、优先级
```

#### 华为 VRRP

```
display vrrp brief           ! VRRP 备份组、主备状态
```

### 2. 设备堆叠 / 集群

#### Cisco 堆叠

```
show switch stack           ! 堆叠状态、成员设备
```

#### 华为 堆叠 / 集群

```
display stack               ! 堆叠状态
display css status          ! 集群（CSS）状态
```

------

## 十一、巡检速查精简版（现场快速执行清单）

### 【Cisco 极简巡检命令集】

```
enable
show version
show environment
show inventory
show ip interface brief
show interfaces status
show interfaces transceiver detail
show mac address-table
show ip route
show arp
show processes cpu
show processes memory
show logging buffer
show alarm
```

### 【华为 极简巡检命令集】

```
display version
display environment
display device
display interface brief
display transceiver diagnosis
display mac-address
display ip routing-table
display arp
display cpu-usage
display memory-usage
display logbuffer
display alarm active
```

------

## 巡检判断标准（现场快速排障）

1. **硬件**：风扇 / 电源 / 单板全 `Normal/Online`，温度无高温告警
2. **接口**：业务接口状态 `Up`，无大量 CRC、错包、丢包
3. **光模块**：收发光功率在设备标称正常区间（一般 -10~-27dBm）
4. **性能**：CPU、内存无长期高占用
5. **协议**：OSPF/BGP/VRRP 邻居状态正常，路由表无缺失
6. **告警**：无紧急、重要级活跃告警
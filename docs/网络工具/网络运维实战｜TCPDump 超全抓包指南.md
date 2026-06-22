# 网络运维实战｜TCPDump 超全抓包指南

在 Linux 服务器、线上生产环境中，大多数服务器**无图形界面、无法使用 Wireshark**，此时 TCPDump 就是线上故障排查的终极抓包神器。

TCPDump 是 Linux 系统原生、轻量级、高性能的命令行抓包工具，支持精准过滤、后台常驻抓包、保存 pcap 离线分析，是运维排查**业务丢包、接口超时、TCP重传、端口异常、数据包错乱**的核心工具。

本文基于生产环境实战场景，从零讲解 TCPDump 安装、核心命令、过滤语法、离线分析、避坑准则、零影响抓包技巧，适合所有线上服务器故障排查落地使用。

## 一、TCPDump 工具简介

**博文定位**：Linux 服务器无界面抓包、线上业务故障排查、生产环境无损抓包实操指南

**核心优势**：轻量低耗、无需图形、支持复杂过滤、可后台运行、不抢占业务资源、适配所有服务器环境

**适用场景**：业务间歇性超时、接口请求丢包、TCP 重传异常、内网通信异常、服务器对外访问异常、安全流量分析

## 二、TCPDump 安装方法

主流 Linux 发行版均可直接通过 yum / apt 快速安装，大部分 CentOS 系统**默认自带**。

### 1\. CentOS / RHEL

```Plain Text
yum install tcpdump -y
```

### 2\. Ubuntu / Debian

```Plain Text
apt install tcpdump -y
```

### 3\. 查看版本（验证安装）

```Plain Text
tcpdump --version
```

## 三、TCPDump 核心基础用法

TCPDump 命令语法灵活，支持网卡、IP、端口、协议多层过滤，是精准抓包的关键。

### 1\. 查看服务器所有网卡

```Plain Text
tcpdump -D
```

生产环境务必先确认网卡名称（eth0、ens33、bond0 等），避免抓错网卡。

### 2\. 监听指定网卡所有流量

```Plain Text
tcpdump -i eth0
```

实时滚动输出网卡全部数据包，适合简单流量观测。

### 3\. 不解析域名、纯IP输出

```Plain Text
tcpdump -i eth0 -n
```

**生产必加参数**：禁止DNS反向解析，大幅减少服务器资源消耗，避免卡顿。

### 4\. 实时输出详细协议内容

```Plain Text
tcpdump -i eth0 -nnv
```

\-nn：不解析域名、不解析端口协议名；\-v：展示详细数据包信息

## 四、生产高频精准过滤命令（核心实操）

无过滤的全量抓包会产生海量日志、占用磁盘IO，生产环境**禁止裸抓包**。以下为运维最常用的精准过滤语句。

### 1\. 只抓取指定 IP 流量

```Plain Text
tcpdump -i eth0 -nn host 192.168.1.100
```

抓取与该IP所有进出通信数据包

### 2\. 只抓取指定源IP / 目的IP

```Plain Text
# 源IP
tcpdump -i eth0 -nn src host 192.168.1.100
# 目的IP
tcpdump -i eth0 -nn dst host 192.168.1.100
```

### 3\. 抓取指定端口流量

```Plain Text
# 单个端口
tcpdump -i eth0 -nn port 8080
# 多个端口
tcpdump -i eth0 -nn port 80 or port 443
```

### 4\. 抓取指定协议流量

```Plain Text
# TCP
tcpdump -i eth0 -nn tcp
# UDP
tcpdump -i eth0 -nn udp
# ICMP（ping）
tcpdump -i eth0 -nn icmp
```

### 5\. 组合条件过滤（生产最常用）

```Plain Text
# 抓取指定IP+指定端口TCP流量
tcpdump -i eth0 -nn host 192.168.1.100 and port 3306 and tcp
```

精准过滤数据库、接口、业务服务专属流量，排查业务丢包、超时问题。

## 五、保存 PCAP 文件 \+ Wireshark 离线分析

线上实时抓包可读性差，生产环境标准做法：**TCPDump 远端抓包保存 pcap → Wireshark 本地可视化分析**。

### 1\. 保存为 pcap 格式

```Plain Text
tcpdump -i eth0 -nn host 192.168.1.100 -w /tmp/traffic.pcap
```

\-w 参数：写入文件，不输出屏幕，降低资源占用

### 2\. 限制抓包数量，避免文件过大

```Plain Text
tcpdump -i eth0 -nn -c 10000 -w /tmp/traffic.pcap
```

\-c 指定数据包个数，自动停止，防止磁盘打满

### 3\. 本地 Wireshark 分析流程

1\. 通过 Xftp / SCP 将服务器 pcap 文件下载到本地；

2\. 直接拖拽进入 Wireshark；

3\. 使用图形过滤器分析：TCP 重传、乱序、窗口过小、握手失败、ACK 异常；

4\. 精准定位业务卡顿、超时、丢包根因。

## 六、生产环境零影响抓包核心技巧

TCPDump 虽轻量，但错误使用依然会导致**服务器卡顿、磁盘爆满、业务抖动**，以下为生产标准规范。

### 1\. 必须加精准过滤条件

禁止直接 `tcpdump -i eth0` 全量抓包，高并发服务器瞬间产生GB级流量日志，抢占CPU与磁盘IO，引发业务卡顿。

### 2\. 后台静默抓包，不占用终端

```Plain Text
nohup tcpdump -i eth0 -nn port 443 -w /tmp/443.pcap 
```

后台运行，断开SSH不终止抓包，适合长时间间歇性故障排查。

### 3\. 限制文件大小与包数量

线上故障未知持续时间，必须限制包数量或拆分文件，防止磁盘被打满导致服务宕机。

### 4\. 抓包完成务必手动终止进程

后台运行的 tcpdump 不会自动退出，排查结束后需查找并 kill 进程，避免常驻占用资源。

```Plain Text
ps -ef | grep tcpdump
kill -9 进程ID
```

### 5\. 尽量避开业务高峰期

核心业务高峰期，优先使用精准小流量抓包，不做全量抓取，最大限度降低影响。

## 七、TCPDump 高频故障排查实战场景

### 1\. 业务间歇性超时

抓取业务端口流量，通过 Wireshark 分析是否存在**TCP Retransmission 重传、Zero Window 窗口满、Dup ACK**，定位链路不稳定或服务器性能瓶颈。

### 2\. 服务器被恶意访问

抓取异常外联流量、陌生IP爆破端口，溯源攻击IP与攻击行为。

### 3\. 跨服务器通信失败

双向抓包排查：是本机没发包、对方没回包、还是中间防火墙丢弃数据包。

## 八、生产环境 TCPDump 避坑总结

1\. **永远带过滤条件**：禁止裸抓全量网卡流量；

2\. **永远保存 pcap 分析**：线上看文字日志效率极低，离线可视化排障最快；

3\. **必加 \-nn 参数**：关闭解析，保护服务器性能；

4\. **后台抓包必收尾**：防止进程常驻、磁盘爆满；

5\. **故障优先双向抓包**：精准定位是发送端、链路、接收端哪一环故障。



# Linux TC（Traffic Control）命令详解

`tc` 是 **Linux 内核流量控制系统（Traffic Control）** 工具，**管控网卡出站带宽、限流、限速、流量整形、QoS、报文排队、丢包、流量分类**，只对 ** 出方向（egress）** 原生生效；入方向限速需要配合 `ifb` 虚拟网卡。

## 一、核心用途

1. **带宽限速**：整机 / 单个 IP / 端口限速上传 / 下载
2. **流量整形**：平滑突发流量，避免网卡瞬间打满、队列拥塞丢包
3. **QoS 优先级**：关键业务（SSH / 数据库）优先发包，P2P / 下载降优先级
4. **随机丢包**：模拟网络抖动、丢包、延迟（测试环境常用）
5. **流量分流**：不同端口 / IP 走不同带宽队列

> 内核依赖：`CONFIG_NET_SCHED`、各类队列模块（htb、pfifo、netem）

## 二、常用队列规则

表格

|     队列     |                           用途                           |
| :----------: | :------------------------------------------------------: |
| `pfifo_fast` |                  默认先进先出，简单排队                  |
|   **htb**    | 分层令牌桶，**最常用带宽限速**，支持分层带宽、子分类限速 |
|  **netem**   |            网络仿真：加延迟、丢包、抖动、乱序            |

## 三、实操示例（eth0 为网卡）

### 0. 基础操作：清空已有规则

运行

```
# 清空网卡所有tc配置
tc qdisc del dev eth0 root
```

### 示例 1：整机出口全局限速（HTB，网卡总带宽 100Mbps）

运行

```
# 1. 挂载根队列htb
tc qdisc add dev eth0 root handle 1: htb default 10

# 2. 总带宽上限100Mbit
tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit ceil 100mbit

# 3. 默认流量全部分到1:10，限速100M
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 100mbit ceil 100mbit
```

- `rate`：保障最小带宽；`ceil`：峰值最大带宽

### 示例 2：按 IP 限速：192.168.1.10 限速 10Mbps，其余不限速

运行

```
# 根队列
tc qdisc add dev eth0 root handle 1: htb default 20
# 总带宽100M
tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit ceil 100mbit
# 分类1：受限IP 10M
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 10mbit ceil 10mbit
# 分类2：其他主机
tc class add dev eth0 parent 1:1 classid 1:20 htb rate 90mbit ceil 100mbit

# 配置filter：匹配源IP 192.168.1.10 归入1:10队列
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 match ip src 192.168.1.10 flowid 1:10
```

### 示例 3：按端口限速：80 端口（HTTP）限速 20M

运行

```
tc filter add dev eth0 parent 1: protocol ip prio 2 u32 match ip dport 80 0xffff flowid 1:10
```

### 示例 4：netem 模拟网络故障（测试必备）

#### 4.1 出口所有流量增加固定延迟 200ms

运行

```
tc qdisc add dev eth0 root netem delay 200ms
```

#### 4.2 随机丢包 5% 报文

运行

```
tc qdisc add dev eth0 root netem loss 5%
```

#### 4.3 延迟 + 抖动：200ms±50ms 波动

运行

```
tc qdisc add dev eth0 root netem delay 200ms 50ms
```

#### 4.4 报文乱序

运行

```
tc qdisc add dev eth0 root netem reorder 25% gap 3
```

### 示例 5：入方向限速（tc 不能直接控 ingress，用 ifb 虚拟网卡）

运行

```
# 加载ifb模块
modprobe ifb numifbs=1
ip link set dev ifb0 up
# 把eth0入流量重定向到ifb0
tc qdisc add dev eth0 ingress
tc filter add dev eth0 parent ingress protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0
# 在ifb0做限速（同出口tc规则）
tc qdisc add dev ifb0 root handle 1: htb
tc class add dev ifb0 parent 1: classid 1:1 htb rate 50mbit
```

## 四、查看规则

运行

```
# 查看队列
tc qdisc show dev eth0
# 查看分类
tc class show dev eth0
# 查看过滤规则
tc filter show dev eth0
```

## 五、单位说明

`bit` 带宽单位（小写比特）：`kbit/mbit/gbit`；

文件下载 `KB` 是字节，`1Byte=8bit`。
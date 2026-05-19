# iptables 与 ufw 对比 + 常用封禁 IP、网段放行端口策略

## 一、iptables 和 ufw 核心对比

表格

|    维度    |                           iptables                           |                      ufw                      |
| :--------: | :----------------------------------------------------------: | :-------------------------------------------: |
|    定位    |       Linux **底层原生防火墙**，内核级 netfilter 框架        | **iptables 前端简化工具**，Ubuntu/Debian 默认 |
|   复杂度   | 语法复杂、规则链多（INPUT/OUTPUT/FORWARD）、表多（filter/nat/mangle） |     语法极简、上手快，屏蔽底层复杂链和表      |
|  适用场景  |      服务器精细管控、路由转发、端口映射、复杂 ACL 规则       |    单机快速防护、个人 / 小型服务器极简配置    |
| 规则持久化 |    需手动保存（iptables-save/iptables-restore）或配置文件    |     自带 `enable/disable`，规则自动持久化     |
|  系统支持  |                    所有 Linux 发行版通用                     |      主打 Debian/Ubuntu，CentOS 也可安装      |
| 转发 / NAT |            完美支持端口转发、内网穿透、SNAT/DNAT             |    仅基础放行封禁，**不适合复杂 NAT 转发**    |

**总结**：

- 追求**简单快速、单机防护** → 用 `ufw`
- 需要**精细过滤、路由转发、端口映射、复杂网段策略** → 用 `iptables`

------

## 二、通用前置说明

默认业务端口示例：

- SSH：22

- HTTP：80

- HTTPS：443

  

  示例网段：

- 信任网段：`192.168.1.0/24`

- 封禁单个 IP：`1.2.3.4`

------

# 三、ufw 常用实战策略（简单推荐）

## 1. 基础开关

运行

```
# 重置所有规则
ufw reset

# 默认拒绝所有入站，允许出站
ufw default deny incoming
ufw default allow outgoing

# 开启防火墙
ufw enable

# 关闭防火墙
ufw disable

# 查看规则
ufw status verbose
```

## 2. 封禁单个 IP 所有访问

运行

```
ufw deny from 1.2.3.4
```

## 3. 封禁整个网段

运行

```
ufw deny from 10.0.0.0/24
```

## 4. 仅允许指定网段访问 SSH (22)

运行

```
# 只允许 192.168.1.0/24 连22端口
ufw allow from 192.168.1.0/24 to any port 22
```

## 5. 允许指定网段访问 80/443

运行

```
ufw allow from 192.168.1.0/24 to any port 80,443 proto tcp
```

## 6. 允许单个 IP 访问所有端口

运行

```
ufw allow from 5.6.7.8
```

## 7. 删除某条规则

先 `ufw status numbered` 看序号，再：

运行

```
ufw delete 规则序号
```

------

# 四、iptables 常用实战策略（通用全 Linux）

## 1. 清空旧规则、设置默认策略

运行

```
# 清空所有规则
iptables -F
iptables -X

# 默认入站拒绝、出站允许、转发拒绝
iptables -P INPUT DROP
iptables -P OUTPUT ACCEPT
iptables -P FORWARD DROP
```

## 2. 放行本地回环（必加，否则本机异常）

运行

```
iptables -A INPUT -i lo -j ACCEPT
```

## 3. 封禁单个 IP 所有入站

运行

```
iptables -A INPUT -s 1.2.3.4 -j DROP
```

## 4. 封禁整个网段

运行

```
iptables -A INPUT -s 10.0.0.0/24 -j DROP
```

## 5. 仅允许指定网段访问 SSH 22 端口

运行

```
iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 22 -j ACCEPT
```

## 6. 允许指定网段访问 80/443

运行

```
iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 443 -j ACCEPT
```

## 7. 允许某个公网 IP 全部访问

运行

```
iptables -A INPUT -s 5.6.7.8 -j ACCEPT
```

## 8. 允许已建立的连接通行（必备，避免断连）

运行

```
iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
```

## 9. 查看 iptables 规则

运行

```
iptables -L -n --line-numbers
```

## 10. 删除指定序号规则

运行

```
iptables -D INPUT 规则序号
```

## 11. iptables 规则持久化

运行

```
# CentOS
service iptables save

# Debian/Ubuntu
iptables-save > /etc/iptables/rules.v4
```

------

# 五、选型建议

1. 新手、Ubuntu 单机、只做简单封禁 / 放行 → **直接用 ufw**
2. 生产服务器、需要端口转发、多网段复杂权限、CentOS → **用 iptables**
3. 原则：先默认拒绝所有入站，再逐条按需放行最小权限端口和网段。


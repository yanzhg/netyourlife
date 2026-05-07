# 生产服务器标准防火墙规则模板（iptables + ufw 两套可直接复制）

## 前置统一规划

- 信任内网网段：`192.168.1.0/24`
- 业务端口：80、443 对外公开
- 管理端口：22 仅内网信任网段可连，外网禁止
- 默认策略：**所有入站默认拒绝，仅放行指定 IP / 端口**
- 已建立相关连接全部放行，不打断正常业务

------

# 一、UFW 完整版生产模板（Ubuntu/Debian 直接用）

## 1. 一键初始化 + 配置

运行

```
# 重置清空旧规则
ufw reset

# 默认入站拒绝、出站允许
ufw default deny incoming
ufw default allow outgoing

# 放行本地回环
ufw allow in on lo

# 仅允许内网 192.168.1.0/24 访问 SSH 22
ufw allow from 192.168.1.0/24 to any port 22 proto tcp

# 对外开放 80、443 所有IP可访问
ufw allow 80/tcp
ufw allow 443/tcp

# 【可选】单独封禁恶意IP示例
ufw deny from 111.222.33.44

# 启用防火墙
ufw enable

# 查看规则
ufw status verbose
```

## 2. 常用运维命令

运行

```
# 带序号查看，方便删规则
ufw status numbered

# 删除第N条规则
ufw delete N

# 关闭防火墙
ufw disable
```

------

# 二、iptables 完整版生产模板（CentOS / 所有 Linux 通用）

## 1. 逐条复制执行（标准生产基线）

运行

```
# 清空所有旧规则、自定义链
iptables -F
iptables -X

# 设置默认策略
iptables -P INPUT DROP
iptables -P OUTPUT ACCEPT
iptables -P FORWARD DROP

# 放行本地回环lo（必须）
iptables -A INPUT -i lo -j ACCEPT

# 放行已建立/相关连接（必须，防止业务断连）
iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT

# 仅内网 192.168.1.0/24 允许SSH 22
iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 22 -j ACCEPT

# 对外开放80、443
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 封禁单个恶意IP 全网拦截
iptables -A INPUT -s 111.222.33.44 -j DROP

# 封禁恶意整个网段示例
# iptables -A INPUT -s 222.111.0.0/16 -j DROP
```

## 2. 查看 & 删除规则

运行

```
# 查看带序号规则
iptables -L -n --line-numbers

# 删除INPUT链第N条规则
iptables -D INPUT N
```

## 3. iptables 规则持久化

### CentOS 6/7

运行

```
service iptables save
systemctl enable iptables
```

### Ubuntu/Debian

运行

```
iptables-save > /etc/iptables/rules.v4
```

------

# 三、扩展常用场景（直接加规则即可）

1. 额外放行一个公网 IP 单独连 SSH

   UFW：

运行

```
ufw allow from 22.33.44.55 to any port 22 proto tcp
```

iptables：

运行

```
iptables -A INPUT -s 22.33.44.55 -p tcp --dport 22 -j ACCEPT
```

1. **封禁某 IP 所有端口**

   UFW：`ufw deny from 1.2.3.4`

   iptables：`iptables -A INPUT -s 1.2.3.4 -j DROP`

   

2. **只允许指定网段访问 3306 数据库**

   UFW：

   运行

```
ufw allow from 192.168.1.0/24 to any port 3306 proto tcp
```

iptables：

运行

```
iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 3306 -j ACCEPT
```

------

# 四、使用建议

- Ubuntu/Debian 直接用 **UFW 模板**，简单省心、自动持久化；
- CentOS、需要做端口转发 / 内网隔离、复杂 ACL，用 **iptables 模板**；
- 永远遵循：**默认拒绝，按需最小放行**，不要全开再拦。
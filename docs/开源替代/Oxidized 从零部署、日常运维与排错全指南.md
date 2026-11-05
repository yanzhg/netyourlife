# Zabbix 6.0 LTS 从零部署、日常运维与排错全指南

> 标签：#Zabbix #监控运维 #告警 #自动化运维 简介：Zabbix 是业界最主流开源监控平台，支持主机、网络设备、数据库、中间件、自定义脚本、SNMP、IPMI 等采集，自带告警、图形、自动发现、分布式 Proxy。本文基于长期生产实践，覆盖**完整安装部署、日常运维规范、高频故障排错**，适合内网服务器、IDC、企业网络落地。

> 版本说明：本文选用 **Zabbix 6.0 LTS（长期支持版）**，稳定、BUG 少，适合生产环境；操作系统：Ubuntu 22.04 + MySQL8.0 + Nginx + PHP。

## 一、Zabbix 基础概念

- Zabbix Server：核心服务，负责拉取数据、计算触发器、生成告警、写入数据库，端口 **10051**
- Zabbix Agent：被监控端采集程序，分被动模式 / 主动模式，端口 10050
  - 被动：Server 连 Agent 10050 拉取数据，适合服务器可直连场景
  - 主动：Agent 主动连 Server 10051 上报数据，适合 NAT、跨网、大量节点，减轻 server 压力
- Zabbix Proxy：分布式代理，跨机房、跨网段场景分担采集压力
- Web UI：PHP 前端，配置管理、看图、告警查看
- Database：MySQL / PostgreSQL，存储所有配置、历史指标、事件告警（Zabbix 性能瓶颈大多在数据库）

## 二、安装部署（Ubuntu 22.04 + MySQL8.0 + Nginx）

### 2.1 前置准备

服务器建议配置（参考）

- 小规模（<100 台）：4C 8G，SSD 优先
- 中规模（100~500 台）：8C 16G，独立 SSD 给 MySQL

> 必须关闭 swap 或者调低 swapiness，数据库对 swap 非常敏感

### 2.2 添加官方源

```
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-4+ubuntu22.04_all.deb
dpkg -i zabbix-release_6.0-4+ubuntu22.04_all.deb
apt update
```

### 2.3 安装组件

```
apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf zabbix-sql-scripts zabbix-agent
```

### 2.4 初始化 MySQL 数据库

登录 mysql

```
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER zabbix@localhost IDENTIFIED BY 'Zabbix@Passw0rd';
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
SET GLOBAL log_bin_trust_function_creators = 1;
FLUSH PRIVILEGES;
```

导入初始 schema

```
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbix -p zabbix
```

### 2.5 修改 zabbix-server 配置

```
vim /etc/zabbix/zabbix_server.conf
```

关键参数：

```
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=Zabbix@Passw0rd
# 调优参数，根据机器配置修改
StartPollers=10
StartPollersUnreachable=5
StartTrappers=5
StartPingers=5
StartDiscoverers=2
CacheSize=512M
HistoryCacheSize=256M
TrendCacheSize=128M
ValueCacheSize=256M
LogFile=/var/log/zabbix/zabbix_server.log
LogFileSize=100
```

### 2.6 Nginx + PHP 前端配置

官方包自带 nginx 配置：`/etc/nginx/conf.d/zabbix.conf` 修改监听、域名，PHP 时区必须改成 Asia/Shanghai

```
vim /etc/php/8.1/fpm/php.ini
date.timezone = Asia/Shanghai
memory_limit = 256M
post_max_size = 32M
max_execution_time = 300
```

### 2.7 启动服务并开机自启

```
systemctl restart zabbix-server zabbix-agent nginx php8.1-fpm
systemctl enable zabbix-server zabbix-agent nginx php8.1-fpm
```

### 2.8 Web 初始化向导

浏览器访问 `http://服务器IP:8080`（默认端口 8080，可自行修改 nginx）

1. 检查 PHP 依赖全部 OK
2. 填写数据库信息（zabbix 库、账号密码）
3. Zabbix server 地址默认`localhost`，端口 10051
4. 设置管理员账号密码，完成安装

> 默认管理员账号：`Admin` 密码：`zabbix`，上线第一件事修改密码

### 2.9 客户端 Agent 安装（被监控 Linux 主机）

```
apt install zabbix-agent
vim /etc/zabbix/zabbix_agentd.conf
Server=ZABBIX_SERVER_IP          # 被动模式，允许server来拉取
ServerActive=ZABBIX_SERVER_IP    # 主动模式，agent上报
Hostname=web01                   # 主机名，必须和Web界面主机名称完全一致
```

放行防火墙：**10050/tcp**

```
systemctl restart zabbix-agent
systemctl enable zabbix-agent
```

## 三、日常运维规范（生产必看）

### 3.1 常用运维命令

```
# 查看服务状态
systemctl status zabbix-server
systemctl status zabbix-agent

# 实时日志（排错第一优先）
tail -f /var/log/zabbix/zabbix_server.log
tail -f /var/log/zabbix/zabbix_agentd.log

# 测试agent本地采集（在被监控机执行）
zabbix_get -s 127.0.0.1 -p 10050 -k system.cpu.load[percpu,avg1]
```

### 3.2 数据库备份策略（重中之重）

Zabbix 所有配置、历史数据都在 MySQL，**定时全量备份** 示例脚本：

```
mysqldump -uzabbix -p zabbix --single-transaction --quick --lock-tables=false > /data/backup/zabbix_$(date +%F).sql
```

- 小规模：每日全量备份，保留 7 天
- 大规模：全量 + binlog 增量，或者使用 xtrabackup
- 升级前、大变更前**必须先备份数据库**

### 3.3 历史数据清理与 Housekeeping

Zabbix 最容易踩坑：history、history_str、trends 表持续膨胀，数据库越来越慢。 Web 界面：【管理】→【维护】

- 历史数据：默认 90 天，业务不需要可以缩短到 7~30 天
- 趋势数据：可以保留 365 天（聚合数据，体积小）

> 高级优化方案：MySQL 分区表 / 迁移 PostgreSQL+TimescaleDB 时序压缩，替代原生 housekeeping（delete 删除极慢）

### 3.4 告警治理（降噪）

很多 Zabbix 上线后告警轰炸，核心原则：

1. 触发器必须设置合理阈值、故障持续时间（不要瞬时告警）
2. 配置告警收敛、告警升级、故障确认
3. 无用触发器直接关闭，区分【P0 紧急、P1 重要、P2 提示】
4. 告警渠道：钉钉 / 企业微信 / 邮件，不要所有告警都发手机

### 3.5 权限与安全加固

1. 修改 Admin 默认密码，新建普通运维账号，最小权限
2. Web 前端不要直接暴露公网，Nginx 增加 IP 白名单、Basic Auth、HTTPS
3. zabbix 数据库账号只允许本地 / 指定 IP 访问，禁止远程通配符
4. Agent 不要开启不安全参数，禁止自定义命令高危权限
5. 定期升级 Zabbix 补丁，修复安全漏洞

### 3.6 版本升级注意事项

Zabbix 跨版本升级只能顺序升级，LTS 版本优先：

- 步骤：停止 zabbix-server → 数据库全量备份 → 更新包 → 启动服务，自动升级表结构
- 升级期间不要操作 Web，等待数据库迁移完成，日志无报错再使用

## 四、高频排错指南（按故障场景分类）

> 排错通用顺序：先看日志 → 网络连通性 → 端口 → 配置一致性 → 权限 → 数据库

### 故障 1：Web 页面提示：Zabbix server is not running

现象：页面顶部红色提示，但是 systemctl 看服务是 running 排查：

1. 检查`zabbix_server.conf`日志，大概率数据库连接失败、密码错误、数据库没起来
2. 检查 10051 端口是否监听：`ss -tlnp | grep 10051`
3. 检查 web 配置文件 `/etc/zabbix/web/zabbix.conf.php` 里的 server 地址是否正确
4. MySQL 8.0 权限、密码加密方式问题，旧密码插件不兼容

### 故障 2：主机状态灰色，监控无数据（最常见）

分被动模式和主动模式分开排查： ✅ 被动模式（server 连 agent 10050）

1. Server 上测试连通：`telnet 主机IP 10050` 或者 `nc -zv IP 10050`
2. 被监控机防火墙 / 安全组放行 10050
3. agent 配置里 `Server=` 填写的 IP，是否包含 zabbix server 地址
4. 使用`zabbix_get`在 server 上测试 key，看是否返回数据

✅ 主动模式（agent 上报 10051）

1. agent 日志：`tail -f /var/log/zabbix/zabbix_agentd.log`，看是否连接 server 10051 成功
2. agent 配置 `ServerActive=` 正确，防火墙放行 agent 出站到 10051
3. **Hostname 严格匹配**：agent 内 Hostname 和 Web 页面主机名称一字不差（大小写敏感）
4. 主机模板是否添加、监控项类型是不是【Zabbix agent 主动】

### 故障 3：图形中文乱码，方框□□□

原因：缺少中文字体 解决：

```
# Ubuntu
apt install fonts-wqy-microhei
# 复制字体到zabbix字体目录，修改默认字体
cp /usr/share/fonts/truetype/wqy/wqy-microhei.ttc /usr/share/zabbix/assets/fonts/
# 修改 /usr/share/zabbix/include/defines.inc.php 里字体名称
```

也可以直接上传 Windows 微软雅黑字体到 fonts 目录，修改配置文件指定字体腾讯云

### 故障 4：数据库慢、页面卡顿、加载超时

1. 优先看 history 大表，维护周期太长，数据堆积
2. MySQL 优化：调整 innodb_buffer_pool_size（物理内存 50%~70%），关闭不需要的查询日志
3. 检查 housekeeping 进程卡死，大量 delete 阻塞
4. 大规模集群：上 Proxy 分流采集压力、数据库分区、迁移 TimescaleDB

### 故障 5：SNMP 设备采集不到数据（交换机、防火墙）

1. 连通性：UDP 161 端口，很多防火墙默认拦截 UDP
2. SNMP v2c：community 团体字一致；v3：账号、认证加密算法、密码严格一致
3. OID 是否正确，部分厂商私有 OID 不兼容通用模板
4. 调大 `StartSNMPPollers` 并发采集进程

### 故障 6：触发器不触发，明明指标已经超过阈值

1. 监控项本身没有正常采集，无最新数据
2. 触发器表达式写错、单位不匹配（字节 vs 兆）
3. 触发器设置了【持续多久】，没有达到时间条件不会告警
4. 主机 / 监控项被禁用、模板没有关联成功

### 故障 7：zabbix-server 启动失败，直接 crash

1. 查看日志第一行报错：
   - 数据库密码错误、库不存在：DB connect failed
   - YAML / 配置文件语法错误：读取配置失败
   - 共享内存信号量问题：`cannot create semaphore set`，修改 systemd RemoveIPC 参数
2. 磁盘满了：/var/log 或者数据库分区 100%，直接无法启动

### 故障 8：告警收不到（钉钉 / 企业微信脚本不推送）

1. 脚本权限：zabbix 用户是否有执行权限，脚本路径在配置里正确填写
2. zabbix 用户是否可以外网连通 webhook 地址（代理、白名单）
3. 脚本内日志输出，在动作里看【通知日志】，查看返回码
4. 动作条件、用户媒介、告警时间段配置错误

### 故障 9：时间不同步，监控时序错乱

Zabbix 极度依赖时间同步，所有 Server、Proxy、Agent 必须统一 NTP 时间

```
apt install chrony
systemctl enable --now chronyd
```

时间差超过 30s 很容易出现数据丢包、图形错位、主动模式异常

## 五、进阶扩展方向

1. Zabbix Proxy 分布式部署，跨机房、跨专线隔离采集
2. 自动发现 + 自动注册，批量上线主机
3. 自定义模板、自定义脚本、监控硬件、日志监控（logrt）
4. Grafana 对接 Zabbix 数据源，做大屏展示
5. 结合 Ansible 批量推送 Agent、自动添加主机
6. 监控 Zabbix 自身：监控 zabbix 进程、队列、数据库慢查询，防止监控平台失明

## 六、总结

Zabbix 功能强大，但瓶颈往往在**数据库、时间同步、网络策略、告警治理**。小规模直接单机部署；超过 200 台设备建议引入 Proxy 分担采集压力；长期保存指标一定要提前规划时序存储方案，不要等数据库爆掉再处理。 上线三件套：**定时数据库备份、NTP 时间同步、先收敛告警**，这三点做好，Zabbix 稳定性提升一大截。
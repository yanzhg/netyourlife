# 云原生运维实战｜Prometheus\+Grafana 全套监控指南

传统运维依赖 Zabbix、Nagios 监控，部署笨重、耦合度高、适配云原生能力差、扩展困难。现如今 **Prometheus\+Grafana** 已经成为现代化运维、云原生、容器化、微服务架构的**标准监控方案**。

这套架构轻量、解耦、可扩展、可视化极强，不仅可以监控服务器、业务服务，还可以监控交换机、路由器等网络设备，完全覆盖机房运维、云主机、容器集群、网络设备全场景。

**博文定位**：现代化运维监控可视化一站式实战教程，从零搭建生产级监控平台，包含服务器\+网络设备监控、仪表盘模板复用、告警规则配置全流程落地

## 一、Prometheus\+Grafana 核心组件详解

整套监控体系并非单一工具，而是**多组件协同架构**，新手必须先理清组件分工，才能理解监控完整流程。

### 1\. Prometheus（时序数据库\+采集服务）

Prometheus 是整套架构的核心，负责 **定时采集指标、存储时序数据、执行告警规则**。区别于传统监控，它采用「主动拉取（Pull）」模式，稳定性更高、适配容器动态扩缩容场景。

**核心能力**：指标采集、时序数据存储、PromQL 数据查询、告警规则计算、监控目标管理。

### 2\. Grafana（可视化展示面板）

Grafana 专注**数据可视化与展示**，不存储数据，仅对接 Prometheus 数据源，将枯燥的监控指标转化为曲线图、仪表盘、热力图、拓扑图，支持自定义大屏、模板导入、权限管理。

### 3\. Exporter（指标采集客户端）

各类设备、服务无法直接被 Prometheus 识别，需要通过 Exporter 暴露 metrics 指标：

- **node\_exporter**：服务器CPU、内存、磁盘、网络监控

- **snmp\_exporter**：交换机、路由器、防火墙等网络设备监控

- **mysql\_exporter**：数据库监控

- **nginx\_exporter**：Web服务监控

### 4\. Alertmanager（告警分发组件）

独立告警组件，负责接收 Prometheus 告警、去重、降噪、分组，支持推送钉钉、企业微信、邮件、短信，实现故障及时通知。

### 5\. 完整监控流程

Exporter 暴露指标 → Prometheus 定时拉取存储 → Grafana 可视化展示 → 触发阈值后 Alertmanager 推送告警

## 二、生产环境从零部署完整流程

本次采用 **二进制生产部署方式**，稳定无容器依赖、便于开机自启、适配所有Linux服务器，适合企业线上环境。

### 1\. Prometheus 部署安装

```plaintext
# 下载解压
wget https://github.com/prometheus/prometheus/releases/download/v2.53.1/prometheus-2.53.1.linux-amd64.tar.gz
tar -zxvf prometheus-2.53.1.linux-amd64.tar.gz
mv prometheus-2.53.1.linux-amd64 /usr/local/prometheus

# 创建数据目录与规则目录
mkdir -p /data/prometheus
mkdir -p /usr/local/prometheus/rules
```

### 2\. 核心配置 prometheus\.yml

配置全局抓取周期、告警组件、监控任务，适配服务器与网络设备监控：

```plaintext
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "rules/*.yml"

alerting:
  alertmanagers:
  - static_configs:
    - targets:
        - 127.0.0.1:9093

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9090']

  - job_name: 'linux-server'
    static_configs:
      - targets: ['192.168.1.100:9100']

  - job_name: 'network-device'
    static_configs:
      - targets: ['127.0.0.1:9116']
```

### 3\. 配置系统服务开机自启

创建 systemd 服务，实现常驻运行、开机自启、异常重启：

```plaintext
[Unit]
Description=Prometheus
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/prometheus/prometheus \
--config.file=/usr/local/prometheus/prometheus.yml \
--storage.tsdb.path=/data/prometheus

[Install]
WantedBy=multi-user.target
```

### 4\. Grafana 部署安装

```plaintext
yum install grafana -y
systemctl start grafana-server
systemctl enable grafana-server
```

默认端口 3000，初始账号密码：**admin / admin**，首次登录强制修改密码。

### 5\. 对接 Prometheus 数据源

1\. 登录 Grafana 后台 → Configuration → Data sources

2\. 添加 Prometheus 数据源，填写地址：`http://127.0.0.1:9090`

3\. 点击 Save \& test，显示成功即对接完成。

## 三、网络设备监控实战（交换机/路由器）

传统监控很难适配网络设备，Prometheus 通过 **SNMP Exporter** 实现全网设备统一监控，支持华为、华三、锐捷、思科全品牌设备。

### 1\. SNMP Exporter 部署

安装启动监听 9116 端口，用于采集网络设备 SNMP 指标：

```plaintext
yum install snmp_exporter -y
systemctl start snmp_exporter
systemctl enable snmp_exporter
```

### 2\. 网络设备配置（关键）

交换机/路由器开启 SNMP 服务，配置只读团体字，允许监控服务器抓取设备数据。

### 3\. Prometheus 动态relabel配置

支持批量监控多台网络设备，无需逐个新增任务：

```plaintext
relabel_configs:
- source_labels: [__param_target]
  target_label: instance
- target_label: __address__
  replacement: 127.0.0.1:9116
```

### 4\. 监控覆盖指标

设备在线状态、端口UP/DOWN、端口出入流量、带宽利用率、设备CPU/温度、报文错包丢包，完全替代传统网管平台。

## 四、Grafana 仪表盘模板一键复用（生产必备）

无需手动画图，Grafana 官方提供海量成熟仪表盘模板，**输入ID一键导入**，直接生成高颜值监控大屏。

### 1\. 常用生产模板ID（直接抄作业）

- **Linux服务器监控**：1860（最全节点监控模板，CPU/内存/磁盘/网络全覆盖）

- **网络设备SNMP监控**：11190、15170

- **MySQL数据库监控**：12900

- **Nginx业务监控**：8919

### 2\. 导入步骤

1\. 左侧菜单 Dashboards → Import

2\. 输入模板ID，点击 Load

3\. 选择 Prometheus 数据源，确认导入

4\. 秒级生成完整监控大屏，支持自定义配色、告警阈值、时间粒度

### 3\. 自定义仪表盘导出复用

自己调整好的大屏可直接导出 JSON 文件，新环境一键导入，实现监控模板标准化复用。

## 五、企业级告警规则完整配置

监控可视化只是基础，**及时告警才是运维核心**。本节落地服务器、网络设备高频告警规则。

### 1\. 编写告警规则文件

新建 `/usr/local/prometheus/rules/node_alerts.yml`

```plaintext
groups:
- name: node_alerts
  rules:
  # CPU过高告警
  - alert: HighCpuUsage
    expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[1m])) * 100) > 90
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "服务器CPU使用率过高"

  # 内存过高告警
  - alert: HighMemoryUsage
    expr: 100 - (node_memory_MemFree_bytes/node_memory_MemTotal_bytes * 100) > 90
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "服务器内存占用过高"

  # 磁盘使用率过高
  - alert: HighDiskUsage
    expr: 100 - (node_filesystem_free_bytes / node_filesystem_size_bytes * 100) > 90
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "磁盘空间即将占满"

  # 设备离线告警
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "监控设备离线故障"
```

### 2\. Alertmanager 告警推送配置

支持企业微信、钉钉、邮件推送，配置完成后故障自动推送运维人员，无需人工盯屏。

### 3\. Grafana 可视化告警

支持面板直接创建告警规则、配置告警阈值、查看告警历史、告警状态可视化，兼顾简易运维与精准通知。

## 六、生产环境最佳实践与运维规范

- **统一模板规范**：全网服务器、网络设备统一使用标准仪表盘，避免样式混乱

- **告警降噪处理**：配置持续时间 for 2m，杜绝瞬时波动误告警

- **分层监控**：硬件资源监控、网络设备监控、业务监控分层管理

- **定期备份数据**：定时备份 TSDB 时序数据与告警规则文件

- **权限分级**：Grafana 区分管理员、只读用户，防止误删监控配置

- **网络设备常态化巡检**：依托SNMP监控，实现端口故障、带宽拥堵、设备温度异常提前预警

## 七、全文总结

Prometheus\+Grafana 是**现代化运维的监控基石**，对比传统监控具备轻量、灵活、可扩展、云原生适配、可视化极强的核心优势。

本文完整落地了 **组件原理、从零部署、网络设备监控、官方仪表盘模板复用、企业级告警规则配置** 全流程，一套架构可同时覆盖服务器、网络设备、业务服务监控，完全满足中小企业生产环境监控刚需，是运维工程师进阶云原生的必备核心技能。



# **踩坑实录：Nautobot + Golden Config 容器化部署与 Celery 版本冲突排查**

在自动化运维体系中，Nautobot 作为新一代的网络事实单一来源（SSOT），配合 Golden Config 插件可以实现强大的网络配置合规检查。本文记录了在 Debian 11 环境下，使用 Docker Compose 部署 Nautobot 2.3.10 及 Golden Config 插件的完整过程，重点复盘了部署过程中遇到的 **Celery 与 Redis 版本冲突** 以及 **超级用户缺失** 两个典型坑点。

## **一、 环境准备与基础架构**

本次部署基于 Docker Compose 编排，核心组件包括：

- **Nautobot**: 2.3.10 (基于 Python 3.11)
- **PostgreSQL**: 16 (数据库)
- **Redis**: 7-alpine (消息队列与缓存)
- **插件**: nautobot-plugin-nornir, nautobot-golden-config

工作目录规划在 `/opt/nautobot-docker`。

## **二、 定制 Dockerfile：引入插件与依赖**

官方提供的 Nautobot 镜像默认不包含 Golden Config 和 Nornir 插件。我们需要通过自定义 `Dockerfile` 来扩展基础镜像。

**初始的 Dockerfile 如下：**

dockerfile



```
FROM networktocode/nautobot:2.3.10-py3.11
RUN pip install --no-cache-dir --timeout 120 --retries 10 \
    -i https://mirrors.aliyun.com/pypi/simple/ \
    "nautobot-plugin-nornir>=2.0,<3.0" \
    "nautobot-golden-config>=2.0,<3.0"
```

> **避坑指南**：在 Nautobot 的 Docker 构建机制中，依赖必须写死在 Dockerfile 的 `RUN` 命令中。如果仅仅将依赖写入 `local_requirements.txt`，构建系统并不会自动读取，导致插件安装失败。

## **三、 核心故障排查：Celery Worker 崩溃**

执行 `docker compose up -d` 后，通过 `docker compose ps` 发现 `nautobot-worker` 容器状态异常，不断重启，日志中抛出 `VersionMismatch` 错误。

### **1. 现象分析**

查看 Worker 日志，发现其无法正常连接到 Redis Broker。通过进入容器检查依赖版本：

bash



```
docker exec -it nautobot-worker pip show redis | grep -i version
```

输出显示 `Version: 5.0.7`。

### **2. 根因定位**

Nautobot 2.3.10 基础镜像内置了 Celery 5.6.3 和 Kombu 5.5。这套较新的 Celery 组件要求 `redis-py` 版本必须 `>=6.1.0`。然而，基础镜像预装的 `redis-py` 版本被锁定在 5.0.7，导致 Kombu 在初始化 Broker 时触发版本校验失败。

### **3. 解决方案**

修改 Dockerfile，在 pip install 阶段显式升级 redis-py：

dockerfile



```
FROM networktocode/nautobot:2.3.10-py3.11
RUN pip install --no-cache-dir --timeout 120 --retries 10 \
    -i https://mirrors.aliyun.com/pypi/simple/ \
    "nautobot-plugin-nornir>=2.0,<3.0" \
    "nautobot-golden-config>=2.0,<3.0" \
    "redis>=6.1.0"
```

**强制重建并重启：**

bash



```
docker compose build --no-cache celery_worker celery_beat
docker compose up -d --force-recreate celery_worker celery_beat
```

重建后再次检查，`redis-py` 版本已升级至 `8.1.0`，Worker 日志出现 `celery@... ready.`，表明已成功连接至 Redis。

## **四、 登录故障：超级用户缺失**

部署完成后，访问 Web UI 尝试使用 `admin` 登录，提示 `Please enter a correct username and password`。

### **1. 根因定位**

检查 `docker-compose.yml`，发现未配置 `NAUTOBOT_CREATE_SUPERUSER=true` 及相关的环境变量。Nautobot 的 entrypoint 脚本在缺少这些变量时，会跳过超级用户的初始化步骤，导致数据库中没有任何管理员账号。

### **2. 解决方案**

直接通过 Django 的管理命令手动创建超级用户：

bash



```
docker exec -it nautobot nautobot-server createsuperuser
```

按照提示输入用户名（如 `admin`）、邮箱和强密码。创建成功后，即可正常登录 Web 界面。

## **五、 端到端验证**

为了确认 Celery 任务队列（Beat -> Broker -> Worker）完全打通，我们在 Web UI 中触发了一次 Golden Config 的合规检查任务。

同时，在后台实时监控 Worker 日志：

bash



```
docker logs -f nautobot-worker
```

当日志中出现以下输出时，标志着整个自动化链路已完全就绪：

text



```
[2026-11-19 09:44:40,613: INFO/MainProcess] Connected to redis://:**@redis:6379/4
[2026-11-19 09:44:41,747: INFO/MainProcess] celery@02d3ba4bfe42 ready.
Task nautobot.extras.jobs.run_job[<uuid>] received
Task nautobot.extras.jobs.run_job[<uuid>] succeeded in 1.234s
```

## **六、 总结**

Nautobot 的容器化部署虽然看似只是拉起几个容器，但在引入第三方插件时，极易遇到底层依赖库（如 Celery 与 Redis 客户端）的版本冲突。

**本次部署的核心经验：**

1. **依赖显式声明**：自定义镜像时，务必检查基础镜像的依赖树，避免版本降级或冲突。
2. **环境变量驱动**：Nautobot 的初始化高度依赖环境变量，部署前需仔细核对 `docker-compose.yml` 中的超级用户配置。
3. **日志是最好的老师**：遇到容器反复重启，第一时间 `docker logs` 查看 Python 异常栈，通常能直接定位到缺失的依赖或配置。
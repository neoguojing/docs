
# OpenClaw Docker 使用指南（中文整理版）

> 本文是对 OpenClaw 官方 Docker 安装文档的整理版，方便快速阅读与部署。

---

# 1 Docker 是否适合你

Docker 是 **可选的**。只有在以下情况下建议使用。

## 适合使用 Docker

- 希望运行 **隔离的网关环境**
- 想快速创建 **可丢弃的实验环境**
- 在没有本地依赖的机器上运行 OpenClaw
- 需要 **容器化部署**

## 不适合使用 Docker

- 只是在本机开发
- 希望 **最快开发循环**
- 不需要环境隔离

建议直接使用 **普通安装流程**。

---

# 2 两种 Docker 使用方式

OpenClaw Docker 有两种模式。

## 1 Containerized Gateway（完整 Docker）

整个 OpenClaw 运行在 Docker 中

```
Host
 └── Docker
      ├── openclaw-gateway
      └── openclaw-cli
```

适合：

- VPS
- 服务器部署
- 完全隔离环境

---

## 2 Agent Sandbox（推荐开发模式）

网关在 **主机运行**  
Agent 工具在 **Docker 中运行**

```
Host
 ├── OpenClaw Gateway
 └── Docker Sandbox
       └── Agent tools
```

优点：

- 主程序性能更好
- Agent 工具安全隔离

---

# 3 系统要求

最低要求：

| 资源 | 要求 |
|---|---|
| Docker | Docker Desktop / Docker Engine |
| Compose | Docker Compose v2 |
| 内存 | ≥ 2GB |
| 磁盘 | 足够存储镜像与日志 |

注意：

如果只有 **1GB 内存**，构建镜像可能会出现：

```
exit code 137
```

表示 **OOM（内存不足）**。

---

# 4 Docker 快速安装（推荐）

在项目根目录执行：

```bash
./docker-setup.sh
```

脚本会自动完成：

1 构建 Docker 镜像  
2 运行安装向导  
3 启动 gateway  
4 生成 token  
5 写入 `.env`

完成后打开：

```
http://127.0.0.1:18789
```

然后在 UI 中输入 token。

---

# 5 常用环境变量

| 变量 | 作用 |
|---|---|
| OPENCLAW_IMAGE | 使用远程镜像 |
| OPENCLAW_SANDBOX | 启用 Agent sandbox |
| OPENCLAW_HOME_VOLUME | 持久化 home 目录 |
| OPENCLAW_EXTRA_MOUNTS | 添加额外挂载 |
| OPENCLAW_DOCKER_APT_PACKAGES | 安装 apt 软件包 |
| OPENCLAW_DOCKER_SOCKET | 自定义 docker socket |

示例：

```bash
export OPENCLAW_SANDBOX=1
./docker-setup.sh
```

---

# 6 使用官方镜像

官方镜像地址：

```
ghcr.io/openclaw/openclaw
```

常用标签：

| Tag | 说明 |
|---|---|
| latest | 最新稳定版 |
| main | 主分支 |
| 版本号 | 发布版本 |

示例：

```bash
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
./docker-setup.sh
```

这样可以 **跳过本地构建**。

---

# 7 手动 Docker 部署

如果不使用自动脚本：

构建镜像

```bash
docker build -t openclaw:local -f Dockerfile .
```

运行初始化

```bash
docker compose run --rm openclaw-cli onboard
```

启动服务

```bash
docker compose up -d openclaw-gateway
```

---

# 8 挂载额外目录

通过变量：

```
OPENCLAW_EXTRA_MOUNTS
```

示例：

```bash
export OPENCLAW_EXTRA_MOUNTS="$HOME/github:/home/node/github:rw"
./docker-setup.sh
```

格式：

```
source:target:mode
```

例如：

```
~/github:/home/node/github:rw
```

---

# 9 持久化容器 home

默认容器删除后 `/home/node` 会丢失。

可以使用 Docker volume。

示例：

```bash
export OPENCLAW_HOME_VOLUME="openclaw_home"
./docker-setup.sh
```

数据将保存在：

```
docker volume ls
```

---

# 10 安装系统依赖

需要在容器内安装工具：

例如：

- ffmpeg
- build-essential
- git

示例：

```bash
export OPENCLAW_DOCKER_APT_PACKAGES="git curl jq"
./docker-setup.sh
```

---

# 11 Health Check

容器健康检测：

```
/healthz
/readyz
```

测试：

```bash
curl http://127.0.0.1:18789/healthz
```

---

# 12 Agent Sandbox 介绍

Sandbox 会将 Agent 工具运行在 Docker 中。

默认配置：

| 参数 | 默认 |
|---|---|
| scope | agent |
| workspace | ~/.openclaw/sandboxes |
| network | none |
| memory | 1G |
| cpus | 1 |

---

# 13 Sandbox 隔离级别

| scope | 说明 |
|---|---|
| session | 每个 session 一个容器 |
| agent | 每个 agent 一个容器 |
| shared | 所有 session 共用 |

推荐：

```
scope: agent
```

---

# 14 默认允许工具

Sandbox 默认允许：

- exec
- process
- read
- write
- edit
- session 管理

默认禁止：

- browser
- canvas
- discord
- gateway

---

# 15 构建 Sandbox 镜像

执行：

```bash
scripts/sandbox-setup.sh
```

生成镜像：

```
openclaw-sandbox:bookworm-slim
```

---

# 16 构建开发工具 Sandbox

如果需要编译环境：

```
scripts/sandbox-common-setup.sh
```

镜像：

```
openclaw-sandbox-common:bookworm-slim
```

包含：

- Node
- Go
- Rust

---

# 17 浏览器 Sandbox

如果需要 Browser Tool：

```bash
scripts/sandbox-browser-setup.sh
```

特点：

- Chromium
- CDP 调试
- Xvfb 显示

---

# 18 Docker 网络模式

默认：

```
network: none
```

表示 **无网络访问**。

如果需要联网：

```
network: bridge
```

注意安全风险。

---

# 19 Sandbox 资源限制

示例配置：

```json
{
  "memory": "1g",
  "cpus": 1,
  "pidsLimit": 256
}
```

防止 Agent 占满系统资源。

---

# 20 自动清理

Sandbox 会自动删除旧容器：

| 条件 | 默认 |
|---|---|
| idle | 24h |
| 最大生命周期 | 7 天 |

---

# 总结

OpenClaw Docker 架构：

```
Host
 ├── openclaw-gateway
 ├── openclaw-cli
 └── sandbox containers
```

核心特点：

- Gateway 控制
- CLI 管理
- Sandbox 隔离 Agent

推荐模式：

```
Gateway + Docker Sandbox
```

既安全又高性能。

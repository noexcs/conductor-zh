---
description: "安装并使用 Conductor CLI：在终端中管理工作流、任务、计划、密钥、Webhook，并运行本地 Conductor 服务器。"
---

# Conductor CLI

Conductor CLI（`conductor`）用于管理 Conductor 资源——工作流、任务、计划、密钥、Webhook——还可以在终端中运行本地 Conductor 服务器用于开发。

源码与问题反馈：[conductor-oss/conductor-cli](https://github.com/conductor-oss/conductor-cli)。

## 安装

### npm

```bash
npm install -g @conductor-oss/conductor-cli
```

该命令会下载并安装适用于你平台的对应二进制文件。

### macOS / Linux

```bash
curl -fsSL https://raw.githubusercontent.com/conductor-oss/conductor-cli/main/install.sh | sh
```

该脚本会检测你的操作系统和架构，下载最新的发行版本并安装到 `/usr/local/bin`。若要安装到其他位置：

```bash
INSTALL_DIR=$HOME/.local/bin curl -fsSL https://raw.githubusercontent.com/conductor-oss/conductor-cli/main/install.sh | sh
```

### Windows

```powershell
irm https://raw.githubusercontent.com/conductor-oss/conductor-cli/main/install.ps1 | iex
```

### 验证

```bash
conductor --version
```

## 命令

```text
Conductor Management:
  api-gateway             API Gateway management commands (Orkes Conductor only)
  schedule                Schedule management
  secret                  Secret management
  task                    Task definition and execution management
  webhook                 Webhook management
  workflow                Workflow definition and execution management

CLI Configuration:
  completion              Generate the autocompletion script for the specified shell
  config                  CLI configuration management
  update                  Update the CLI to the latest version
  whoami                  Display information about the current user

Development:
  code                    Generate projects from templates
  server                  Local Conductor server management
  worker                  Task worker management
```

运行 `conductor [command] --help` 可查看任意命令组的标志和子命令——例如 `conductor workflow --help` 或 `conductor server --help`。

## 常见任务

启动本地 Conductor 服务器：

```bash
conductor server start
```

注册工作流定义并运行它：

```bash
conductor workflow create --file my_workflow.json
conductor workflow start --name my_workflow --input '{}'
```

保持 CLI 为最新版本：

```bash
conductor update
```

## 连接服务器

默认情况下，CLI 指向本地 OSS 服务器。可以通过标志或环境变量将其指向其他服务器：

| 标志 | 环境变量 | 用途 |
|---|---|---|
| `--server` | `CONDUCTOR_SERVER_URL` | Conductor 服务器 URL |
| `--server-type` | `CONDUCTOR_SERVER_TYPE` | `OSS`（默认）或 `Enterprise` |
| `--auth-key` / `--auth-secret` | `CONDUCTOR_AUTH_KEY` / `CONDUCTOR_AUTH_SECRET` | API 凭据 |
| `--auth-token` | `CONDUCTOR_AUTH_TOKEN` | Token 认证 |
| `--profile` | `CONDUCTOR_PROFILE` | 命名配置文件（`config-<profile>.yaml`） |

配置文件通过 `conductor config` 管理。

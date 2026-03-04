# OpenClaw 完整安装指南

> **目标**：从零开始，在 Azure VM 上部署 OpenClaw 个人 AI 助手，并连接飞书（Feishu）作为消息通道。  
> **预计用时**：1–2 小时  
> **前置要求**：Azure 账号、飞书管理员权限（用于创建应用）  
> **官方文档**：<https://docs.openclaw.ai/>

---

## 目录

1. [创建 Azure VM](#1-创建-azure-vm)
2. [配置 VM 基础环境](#2-配置-vm-基础环境)
3. [安装 OpenClaw](#3-安装-openclaw)
4. [配置 AI Provider（模型）](#4-配置-ai-provider模型)
5. [配置 Nginx 反向代理](#5-配置-nginx-反向代理)
6. [配置 HTTPS（Let's Encrypt）](#6-配置-httpslets-encrypt)
7. [启用 systemd 守护进程](#7-启用-systemd-守护进程)
8. [设备配对（Device Pairing）](#8-设备配对device-pairing)
9. [连接飞书（Feishu Channel）](#9-连接飞书feishu-channel)
10. [验证 & 常见问题](#10-验证--常见问题)

---

## 1. 创建 Azure VM

### 1.1 通过 Azure CLI 创建资源组 + VM

> 如果你更习惯 Azure Portal 图形界面，可以跳到 [1.2 节](#12-通过-azure-portal-创建可选)。

在本地 PowerShell 中执行（需先安装 [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)）：

```powershell
# 登录 Azure
az login

# 创建资源组（选择你需要的区域）
az group create --name openclaw-rg --location eastasia

# 创建 VM（Ubuntu 24.04, B1ms 规格, SSH 密钥认证）
az vm create `
  --resource-group openclaw-rg `
  --name openclawUbuntu `
  --image Canonical:ubuntu-24_04-lts:server:latest `
  --size Standard_B1ms `
  --admin-username aristo7298 `
  --generate-ssh-keys `
  --public-ip-sku Standard `
  --os-disk-size-gb 128
```

> **规格说明**：`Standard_B1ms`（1 vCPU, 2 GB RAM）足以运行 OpenClaw Gateway。如需同时运行 FutuOpenD 或其他服务，建议升级至 `Standard_B2s`（2 vCPU, 4 GB RAM）。

### 1.2 开放必要的网络端口

```powershell
# 开放 HTTP (80) — 用于 HTTPS 301 重定向和 Certbot 验证
az vm open-port --resource-group openclaw-rg --name openclawUbuntu --port 80 --priority 1010

# 开放 HTTPS (443) — 外部访问
az vm open-port --resource-group openclaw-rg --name openclawUbuntu --port 443 --priority 1020
```

> **安全提示**：不要开放 18789 端口到公网。OpenClaw Gateway 配合 Nginx 反向代理使用时，仅监听 `127.0.0.1:18789`。

### 1.3 配置 DNS 域名标签（可选但推荐）

为 VM 的公网 IP 添加 DNS 标签，这样可以获得一个稳定的域名（格式：`<label>.<region>.cloudapp.azure.com`）：

```powershell
# 查看 VM 公网 IP 资源名称
$ipName = az vm show -g openclaw-rg -n openclawUbuntu -d --query publicIps -o tsv

# 设置 DNS 标签
az network public-ip update `
  --resource-group openclaw-rg `
  --name openclawUbuntuPublicIP `
  --dns-name openclaw
```

> 实际的公网 IP 资源名称可能不同，请使用 `az network public-ip list -g openclaw-rg -o table` 查看。设置后域名为 `openclaw.<region>.cloudapp.azure.com`。

### 1.4 （通过 Azure Portal 创建）可选

1. 登录 <https://portal.azure.com>
2. 搜索 **Virtual machines** → **Create**
3. 选择 **Ubuntu Server 24.04 LTS**
4. 规格选 **Standard_B1ms**
5. 认证方式选 **SSH public key**
6. 网络 → NSG 规则开放 22, 80, 443
7. 创建完成后，在 **公网 IP 资源** → **Configuration** 中设置 DNS name label

### 1.5 测试 SSH 连接

```powershell
ssh <你的用户名>@<你的公网IP或域名>
# 例如:
# ssh aristo7298@openclaw.eastasia.cloudapp.azure.com
```

---

## 2. 配置 VM 基础环境

以下所有命令在 **VM 上（SSH 连入后）** 执行。

### 2.1 更新系统

```bash
sudo apt update && sudo apt upgrade -y
```

### 2.2 安装 Node.js 22

OpenClaw 要求 Node.js ≥ 22。推荐使用 NodeSource 官方仓库：

```bash
# 添加 NodeSource 仓库
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -

# 安装 Node.js
sudo apt install -y nodejs

# 验证版本
node --version   # 应显示 v22.x.x
npm --version
```

> **参考**：<https://github.com/nodesource/distributions#installation-instructions>

---

## 3. 安装 OpenClaw

### 3.1 全局安装

```bash
sudo npm install -g openclaw@latest
```

验证安装：

```bash
openclaw --version
```

### 3.2 运行 Onboarding Wizard

```bash
openclaw onboard --install-daemon
```

Wizard 会引导你完成以下步骤：

1. **选择运行模式** — 选 `local`
2. **Gateway 端口** — 默认 `18789`，直接回车
3. **Gateway 认证模式** — 选 `token`（自动生成一个 48 字符的 Token）
4. **主模型配置** — 如果你已有 API Key，可以在此步配置；也可以稍后手动编辑配置文件
5. **安装 systemd 服务** — `--install-daemon` 标志会自动创建 systemd user service

> **配置文件路径**：`~/.openclaw/openclaw.json`  
> **官方文档**：<https://docs.openclaw.ai/start/getting-started>

### 3.3 确认 Gateway 运行

```bash
openclaw gateway status
```

如果显示 `running`，说明安装成功。你可以通过以下命令查看日志：

```bash
openclaw logs --follow
```

或直接通过 systemd 查看：

```bash
journalctl --user -u openclaw-gateway -f
```

---

## 4. 配置 AI Provider（模型）

OpenClaw 支持多种 AI 模型提供商。以下以 **Azure OpenAI** 和 **Azure AI Claude** 为例。

> **如果你使用 OpenAI 官方 API 或 Anthropic 官方 API**，只需在 `openclaw onboard` 时提供 API Key 即可，无需手动配置 `models.providers`。

### 4.1 编辑配置文件

```bash
nano ~/.openclaw/openclaw.json
```

### 4.2 Azure OpenAI 配置示例

在 `models.providers` 中添加：

```jsonc
{
  "models": {
    "mode": "merge",
    "providers": {
      "azure-openai": {
        "baseUrl": "https://<你的资源名>.openai.azure.com/openai/v1",
        "apiKey": "<你的 API Key>",
        "api": "openai-responses",
        "authHeader": false,
        "headers": {
          "api-key": "<你的 API Key>"
        },
        "models": [
          {
            "id": "<部署名>",
            "name": "GPT-5.2 (Azure)",
            "reasoning": true,
            "input": ["text", "image"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 272000,
            "maxTokens": 16384,
            "compat": { "supportsStore": true }
          }
        ]
      }
    }
  }
}
```

**Azure OpenAI 要点**：
- `baseUrl` 格式：`https://<resource>.openai.azure.com/openai/v1`
- 认证头使用 `api-key`（而非 `Authorization: Bearer`），所以 `authHeader` 设为 `false`，手动在 `headers` 中指定
- `compat.supportsStore` — 推理模型（reasoning: true）需要设为 `true`

### 4.3 Azure AI Claude 配置示例

```jsonc
{
  "models": {
    "mode": "merge",
    "providers": {
      "azure-claude": {
        "baseUrl": "https://<你的资源名>.cognitiveservices.azure.com/anthropic",
        "apiKey": "<你的 API Key>",
        "api": "anthropic-messages",
        "authHeader": false,
        "headers": {
          "x-api-key": "<你的 API Key>",
          "anthropic-version": "2023-06-01"
        },
        "models": [
          {
            "id": "claude-sonnet-4-5",
            "name": "Claude Sonnet 4.5 (Azure AI)",
            "reasoning": true,
            "input": ["text", "image"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

**Azure AI Claude 要点**：

- ⚠️ **`baseUrl` 不要带 `/v1`** — Anthropic SDK 会自动追加 `/v1/messages`，如果你写了 `/v1`，最终请求会变成 `/v1/v1/messages` 导致 404 错误
- 认证头使用 `x-api-key`（而非 `api-key`），需附带 `anthropic-version: 2023-06-01`
- `authHeader` 设为 `false`，避免框架自动添加 `Authorization` 头

### 4.4 设置主模型

在 `agents.defaults.model` 中指定主模型：

```jsonc
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "azure-claude/claude-sonnet-4-5"
      },
      "models": {
        "azure-claude/claude-sonnet-4-5": {},
        "azure-openai/gpt-5.2": {}
      }
    }
  }
}
```

> **格式**：`<provider-id>/<model-id>`  
> **官方文档**：<https://docs.openclaw.ai/concepts/models>

### 4.5 重启 Gateway 使配置生效

```bash
openclaw gateway restart
```

---

## 5. 配置 Nginx 反向代理

Nginx 负责将外部 HTTP/HTTPS 请求转发到 OpenClaw Gateway（`127.0.0.1:18789`），同时处理 SSL 终止。

### 5.1 安装 Nginx

```bash
sudo apt install -y nginx
```

### 5.2 创建站点配置

```bash
sudo nano /etc/nginx/sites-available/openclaw
```

粘贴以下内容（替换 `server_name` 为你自己的域名）：

```nginx
server {
    listen 80;
    server_name openclaw.eastasia.cloudapp.azure.com;

    location / {
        proxy_pass http://127.0.0.1:18789;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }
}
```

> **关键配置**：`proxy_http_version 1.1` + `Upgrade`/`Connection` headers 是 WebSocket 反向代理的必需项。`proxy_read_timeout 86400` 避免长连接超时断开。

### 5.3 启用站点

```bash
# 启用配置
sudo ln -s /etc/nginx/sites-available/openclaw /etc/nginx/sites-enabled/

# 删除默认站点（可选）
sudo rm /etc/nginx/sites-enabled/default

# 测试配置语法
sudo nginx -t

# 重载 Nginx
sudo systemctl reload nginx
```

---

## 6. 配置 HTTPS（Let's Encrypt）

### 6.1 安装 Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
```

### 6.2 申请 SSL 证书

```bash
sudo certbot --nginx -d openclaw.eastasia.cloudapp.azure.com
```

> 将域名替换为你自己的域名。Certbot 会：
> 1. 自动验证域名所有权（通过 HTTP-01 challenge）
> 2. 获取 SSL 证书
> 3. 自动修改 Nginx 配置，添加 SSL 监听和 HTTP → HTTPS 重定向
> 4. 配置自动续期定时任务

### 6.3 验证 HTTPS

```bash
# 测试自动续期
sudo certbot renew --dry-run

# 查看证书状态
sudo certbot certificates
```

在浏览器中访问 `https://你的域名`，应该能看到 OpenClaw 的 Control UI。

> **官方文档**：<https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal>

---

## 7. 启用 systemd 守护进程

如果你在 `openclaw onboard` 时使用了 `--install-daemon`，服务应该已经配置好了。以下是手动配置或检查的方法。

### 7.1 检查现有服务

```bash
systemctl --user status openclaw-gateway
```

### 7.2 手动创建服务（如果需要）

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/openclaw-gateway.service << 'EOF'
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/openclaw gateway --port 18789
Restart=on-failure
RestartSec=10
Environment=NODE_ENV=production

[Install]
WantedBy=default.target
EOF
```

### 7.3 启用并启动服务

```bash
# 重载 systemd
systemctl --user daemon-reload

# 启用开机自启
systemctl --user enable openclaw-gateway

# 启动服务
systemctl --user start openclaw-gateway

# 允许用户服务在未登录时运行
sudo loginctl enable-linger $(whoami)
```

> **`enable-linger` 非常重要**：没有它，当 SSH 断开后，用户级 systemd 服务会被停止。

### 7.4 管理服务

```bash
systemctl --user status openclaw-gateway          # 查看状态
systemctl --user restart openclaw-gateway         # 重启
systemctl --user stop openclaw-gateway            # 停止
journalctl --user -u openclaw-gateway -f          # 实时日志
```

---

## 8. 设备配对（Device Pairing）

当你通过**非 localhost** 地址（如域名或公网 IP）访问 OpenClaw 的 Control UI（WebChat）时，需要进行设备配对。

### 8.1 配对流程

1. 在浏览器中打开 `https://你的域名`
2. 页面会提示你输入 **Gateway Token**
3. 输入 Token 后，在 VM 上执行：

```bash
# 查看待批准的设备
openclaw devices list

# 批准配对请求
openclaw devices approve <请求ID>
```

4. 配对成功后，该设备会被记住，后续无需重复操作

> **Gateway Token** 可以在 `~/.openclaw/openclaw.json` 的 `gateway.auth.token` 字段中找到。

---

## 9. 连接飞书（Feishu Channel）

这是最后一步，让你的 AI 助手可以通过飞书接收和回复消息。

> **官方文档**：<https://docs.openclaw.ai/channels/feishu>

### 9.1 安装飞书插件

在 VM 上执行：

```bash
openclaw plugins install @openclaw/feishu
```

### 9.2 在飞书开放平台创建应用

#### Step 1 — 打开飞书开放平台

访问 <https://open.feishu.cn/app> 并登录。

> 如果你使用的是 Lark（国际版），请访问 <https://open.larksuite.com/app>，并在后续配置中设置 `domain: "lark"`。

#### Step 2 — 创建企业自建应用

1. 点击 **创建企业自建应用**
2. 填写应用名称和描述
3. 选择应用图标

#### Step 3 — 复制凭据

在 **凭证与基础信息** 页面，复制：
- **App ID**（格式：`cli_xxx`）
- **App Secret**

> ⚠️ App Secret 为敏感信息，请妥善保管。

#### Step 4 — 配置权限

在 **权限管理** 页面，点击 **批量导入权限**（Batch import），粘贴以下 JSON：

```json
{
  "scopes": {
    "tenant": [
      "aily:file:read",
      "aily:file:write",
      "application:application.app_message_stats.overview:readonly",
      "application:application:self_manage",
      "application:bot.menu:write",
      "cardkit:card:read",
      "cardkit:card:write",
      "contact:user.employee_id:readonly",
      "corehr:file:download",
      "event:ip_list",
      "im:chat.access_event.bot_p2p_chat:read",
      "im:chat.members:bot_access",
      "im:message",
      "im:message.group_at_msg:readonly",
      "im:message.p2p_msg:readonly",
      "im:message:readonly",
      "im:message:send_as_bot",
      "im:resource"
    ],
    "user": [
      "aily:file:read",
      "aily:file:write",
      "im:chat.access_event.bot_p2p_chat:read"
    ]
  }
}
```

#### Step 5 — 启用机器人能力

在 **应用能力** → **机器人** 中：
1. 启用 **机器人** 能力
2. 设置机器人名称

#### Step 6 — 配置事件订阅

> ⚠️ **重要**：在设置事件订阅之前，确保你已完成 [9.3 节](#93-在-openclaw-中配置飞书-channel) 的 OpenClaw 配置，并且 Gateway 正在运行。

在 **事件订阅** 页面：
1. 选择 **使用长连接接收事件**（WebSocket 模式）
2. 添加事件：`im.message.receive_v1`

> WebSocket 长连接模式不需要公网 webhook URL，飞书 SDK 会主动建立连接。如果 Gateway 未运行，长连接设置可能无法保存。

#### Step 7 — 发布应用

1. 在 **版本管理与发布** 中创建版本
2. 提交审核并发布
3. 等待管理员审批（企业自建应用通常会自动审批）

### 9.3 在 OpenClaw 中配置飞书 Channel

#### 方式一：使用 CLI（推荐）

```bash
openclaw channels add
```

选择 **Feishu**，然后输入你的 App ID 和 App Secret。

#### 方式二：手动编辑配置文件

```bash
nano ~/.openclaw/openclaw.json
```

添加 `channels.feishu` 配置：

```jsonc
{
  "channels": {
    "feishu": {
      "enabled": true,
      "dmPolicy": "pairing",
      "accounts": {
        "main": {
          "appId": "cli_xxxxxxxxxxxxx",
          "appSecret": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
          "botName": "我的AI助手"
        }
      }
    }
  }
}
```

**配置说明**：
- `dmPolicy: "pairing"` — 默认值，未知用户会收到配对码，需要你在服务器端批准
- `connectionMode` — 默认 `websocket`（长连接），不需要公网 webhook
- 如果使用 Lark 国际版，添加 `"domain": "lark"`

### 9.4 重启 Gateway

```bash
openclaw gateway restart
```

检查日志确认飞书连接成功：

```bash
openclaw logs --follow
```

你应该能看到类似 `feishu channel connected` 的日志。

### 9.5 配对飞书用户

1. 在飞书中找到你刚创建的机器人，发送一条消息
2. 机器人会回复一个 **配对码**（pairing code）
3. 在 VM 上执行：

```bash
# 查看配对请求
openclaw pairing list feishu

# 批准配对
openclaw pairing approve feishu <配对码>
```

4. 配对完成后，你就可以正常和 AI 助手对话了！

### 9.6 飞书群聊配置（可选）

如果你希望在群聊中使用机器人：

```jsonc
{
  "channels": {
    "feishu": {
      "groupPolicy": "open",
      "groups": {
        "oc_xxxxxx": {
          "requireMention": true
        }
      }
    }
  }
}
```

**群聊策略**：
- `"open"` — 允许所有群（默认）
- `"allowlist"` — 只允许 `groupAllowFrom` 里列出的群
- `"disabled"` — 禁用群聊

**获取群 ID（chat_id）**：
1. 启动 Gateway 并在群中 @机器人
2. 运行 `openclaw logs --follow` 查看日志中的 `chat_id`

---

## 10. 验证 & 常见问题

### 10.1 完整验证清单

| 步骤 | 验证命令 / 方法 | 预期结果 |
|------|----------------|---------|
| SSH | `ssh <user>@<域名>` | 成功连入 |
| Node.js | `node --version` | v22.x.x |
| OpenClaw | `openclaw --version` | 显示版本号 |
| Gateway | `openclaw gateway status` | running |
| Nginx | `sudo nginx -t` | syntax is ok |
| HTTPS | 浏览器访问 `https://你的域名` | 显示 Control UI |
| 飞书连接 | `openclaw logs --follow` | feishu connected |
| 飞书对话 | 在飞书中给机器人发消息 | 收到 AI 回复 |

### 10.2 常见问题

#### Claude API 返回 404

**原因**：`baseUrl` 末尾多了 `/v1`。Anthropic SDK 会自动追加 `/v1/messages`。

**修复**：将 `baseUrl` 从
```
https://xxx.cognitiveservices.azure.com/anthropic/v1
```
改为
```
https://xxx.cognitiveservices.azure.com/anthropic
```

#### Azure OpenAI 返回 400 Bad Request

**原因**：推理模型需要 `compat.supportsStore` 设置。

**修复**：在模型配置中添加 `"compat": { "supportsStore": true }`

#### 飞书机器人不回复消息

检查步骤：
1. 应用是否已发布并审批通过
2. 事件订阅是否包含 `im.message.receive_v1`
3. 是否启用了长连接（WebSocket）
4. 应用权限是否完整（参考 Step 4 的权限列表）
5. Gateway 是否正在运行：`openclaw gateway status`
6. 查看日志：`openclaw logs --follow`

#### 飞书群聊中机器人不响应

1. 确认机器人已被添加到群中
2. 默认需要 @机器人 才会触发回复
3. 检查 `groupPolicy` 不是 `"disabled"`

#### Gateway 在 SSH 断开后停止运行

**原因**：未启用 `loginctl enable-linger`。

**修复**：
```bash
sudo loginctl enable-linger $(whoami)
```

#### Nginx 502 Bad Gateway

**原因**：OpenClaw Gateway 未运行或未监听 18789 端口。

**修复**：
```bash
systemctl --user restart openclaw-gateway
# 确认端口监听
ss -tlnp | grep 18789
```

---

## 附录 A：完整的 openclaw.json 参考配置

以下是一个包含 Azure AI Provider + 飞书的完整配置示例：

```jsonc
{
  "models": {
    "mode": "merge",
    "providers": {
      "azure-openai": {
        "baseUrl": "https://<resource>.openai.azure.com/openai/v1",
        "apiKey": "<API_KEY>",
        "api": "openai-responses",
        "authHeader": false,
        "headers": {
          "api-key": "<API_KEY>"
        },
        "models": [
          {
            "id": "gpt-5.2",
            "name": "GPT-5.2 (Azure)",
            "reasoning": true,
            "input": ["text", "image"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 272000,
            "maxTokens": 16384,
            "compat": { "supportsStore": true }
          }
        ]
      },
      "azure-claude": {
        "baseUrl": "https://<resource>.cognitiveservices.azure.com/anthropic",
        "apiKey": "<API_KEY>",
        "api": "anthropic-messages",
        "authHeader": false,
        "headers": {
          "x-api-key": "<API_KEY>",
          "anthropic-version": "2023-06-01"
        },
        "models": [
          {
            "id": "claude-sonnet-4-5",
            "name": "Claude Sonnet 4.5 (Azure AI)",
            "reasoning": true,
            "input": ["text", "image"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "azure-claude/claude-sonnet-4-5"
      },
      "models": {
        "azure-claude/claude-sonnet-4-5": {},
        "azure-openai/gpt-5.2": {}
      },
      "workspace": "/home/<user>/.openclaw/workspace"
    }
  },
  "channels": {
    "feishu": {
      "enabled": true,
      "dmPolicy": "pairing",
      "accounts": {
        "main": {
          "appId": "cli_xxxxxxxxxxxxx",
          "appSecret": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
          "botName": "我的AI助手"
        }
      }
    }
  },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "auto",
    "auth": {
      "mode": "token",
      "token": "<自动生成的 Gateway Token>"
    }
  }
}
```

---

## 附录 B：Nginx 完整配置参考

Certbot 自动修改后的完整配置（`/etc/nginx/sites-available/openclaw`）：

```nginx
server {
    server_name openclaw.eastasia.cloudapp.azure.com;

    location / {
        proxy_pass http://127.0.0.1:18789;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/openclaw.eastasia.cloudapp.azure.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/openclaw.eastasia.cloudapp.azure.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    if ($host = openclaw.eastasia.cloudapp.azure.com) {
        return 301 https://$host$request_uri;
    }

    listen 80;
    server_name openclaw.eastasia.cloudapp.azure.com;
    return 404;
}
```

---

## 附录 C：常用命令速查

### VM 管理（本地 PowerShell）

```powershell
az vm start -g openclaw-rg -n openclawUbuntu                              # 启动 VM
az vm stop -g openclaw-rg -n openclawUbuntu                               # 停止 VM
az vm deallocate -g openclaw-rg -n openclawUbuntu                         # 释放（停止计费）
az vm show -g openclaw-rg -n openclawUbuntu -d --query publicIps -o tsv   # 查看公网 IP
```

### OpenClaw 服务管理（VM 上）

```bash
openclaw gateway status                           # Gateway 状态
openclaw gateway restart                          # 重启 Gateway
openclaw logs --follow                            # 实时日志
openclaw devices list                             # 查看设备列表
openclaw devices approve <ID>                     # 批准设备配对
openclaw pairing list feishu                      # 查看飞书配对请求
openclaw pairing approve feishu <CODE>            # 批准飞书用户
openclaw models status                            # 查看模型状态
openclaw doctor                                   # 诊断检查
```

### SSL 证书管理

```bash
sudo certbot renew --dry-run                      # 测试自动续期
sudo certbot certificates                         # 查看证书信息
```

---

## 附录 D：相关文档链接

| 资源 | 链接 |
|------|------|
| OpenClaw 官方文档 | <https://docs.openclaw.ai/> |
| OpenClaw Getting Started | <https://docs.openclaw.ai/start/getting-started> |
| OpenClaw 模型配置 | <https://docs.openclaw.ai/concepts/models> |
| OpenClaw Gateway 配置（全参数） | <https://docs.openclaw.ai/gateway/configuration> |
| OpenClaw 飞书通道 | <https://docs.openclaw.ai/channels/feishu> |
| OpenClaw 安全指南 | <https://docs.openclaw.ai/gateway/security> |
| OpenClaw Docker 部署 | <https://docs.openclaw.ai/install/docker> |
| OpenClaw Linux 平台 | <https://docs.openclaw.ai/platforms/linux> |
| 飞书开放平台 | <https://open.feishu.cn/app> |
| Lark（国际版）开放平台 | <https://open.larksuite.com/app> |
| Azure CLI 安装 | <https://learn.microsoft.com/cli/azure/install-azure-cli> |
| NodeSource Node.js 安装 | <https://github.com/nodesource/distributions> |
| Certbot 安装指南 | <https://certbot.eff.org/> |
| Let's Encrypt 文档 | <https://letsencrypt.org/docs/> |


# OpenClaw Chat Channels（聊天渠道说明）

OpenClaw 可以通过你已经在使用的聊天应用与你交互。  
每个 **Channel（聊天渠道）** 都通过 **Gateway（网关）** 连接。

- 所有渠道都支持 **文本消息**
- **媒体、反应、编辑等功能** 会根据不同渠道有所不同

---

# 支持的聊天渠道

## 推荐渠道

### BlueBubbles（推荐用于 iMessage）
- 使用 **BlueBubbles macOS Server REST API**
- 支持完整功能：
  - 消息编辑
  - 撤回消息
  - 消息效果
  - 表情反应
  - 群组管理
- 注意：在 **macOS 26 Tahoe** 上编辑功能目前存在问题

---

### Discord
- 使用 **Discord Bot API + Gateway**
- 支持：
  - 服务器（Server）
  - 频道（Channel）
  - 私聊（DM）

---

### Telegram
- 使用 **Telegram Bot API（grammY）**
- 设置简单，只需要 **Bot Token**
- 支持：
  - 私聊
  - 群组

---

### WhatsApp
- 最常用的渠道之一
- 使用 **Baileys**
- 需要 **扫码登录（QR Pairing）**
- 会在本地磁盘保存会话状态

---

# 其他支持的渠道

### Feishu（飞书 / Lark）
- 通过 **WebSocket Bot**
- 需要单独安装插件

### Google Chat
- 使用 **Google Chat API**
- 通过 **HTTP Webhook** 接入

### IRC
- 经典 **IRC 协议**
- 支持：
  - Channel
  - 私聊
- 支持 **配对（pairing）** 与 **allowlist 控制**

### LINE
- 使用 **LINE Messaging API**
- 需要插件

### Matrix
- 基于 **Matrix 协议**
- 需要插件

### Mattermost
- 使用 **Bot API + WebSocket**
- 支持：
  - Channels
  - Groups
  - Direct Messages
- 需要插件

### Microsoft Teams
- 使用 **Microsoft Bot Framework**
- 企业级支持
- 需要插件

### Nextcloud Talk
- 自托管聊天系统
- 需要插件

### Nostr
- 去中心化聊天
- 使用 **NIP-04 DMs**
- 需要插件

### Signal
- 使用 **signal-cli**
- 注重隐私安全

### Synology Chat
- 使用 **Synology NAS Chat**
- 通过 **Outgoing + Incoming Webhooks**
- 需要插件

### Slack
- 使用 **Bolt SDK**
- 支持 **Workspace App**

### Tlon
- 基于 **Urbit**
- 需要插件

### Twitch
- 使用 **IRC 连接**
- 需要插件

### WebChat
- Gateway 内置的 **WebChat UI**
- 使用 **WebSocket** 连接

### Zalo
- 使用 **Zalo Bot API**
- 越南流行的聊天软件
- 需要插件

### Zalo Personal
- 使用 **Zalo 个人账号**
- 通过 **二维码登录**
- 需要插件

---

# 使用说明

## 多渠道同时运行

OpenClaw 支持同时配置多个 Channel：

- Telegram
- Discord
- WhatsApp
- Slack
- 等等

系统会根据 **聊天来源自动路由消息**。

---

## 推荐的快速部署渠道

通常建议优先使用：

| 渠道 | 原因 |
|-----|-----|
| Telegram | 配置简单，只需要 Bot Token |
| Discord | API 完整，功能丰富 |
| WhatsApp | 用户最多 |
| BlueBubbles | iMessage 支持最好 |

---

# 注意事项

## 群组行为

不同渠道对 **群组功能** 的支持不同，例如：

- 群组提及
- 机器人权限
- 消息编辑
- 反应

具体请参考：

**Groups 文档**

---

## 安全机制

OpenClaw 对聊天连接做了安全控制：

- **DM Pairing（私聊配对）**
- **Allowlist（允许列表）**

用于防止未知用户访问你的 Agent。

详情见：

**Security 文档**

---

## 故障排查

如果某个 Channel 无法正常工作，可以参考：

**Channel troubleshooting 文档**

---

# 相关文档

- Model Providers（模型提供商）
- Gateway 配置
- Agents 配置
- Sandboxing


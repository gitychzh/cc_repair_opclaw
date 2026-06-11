# CHANGELOG

## v0.2.0 (2026-06-11) — Fix OpenClaw Control UI & Feishu Group Chat Auto-Reply

### 概述
修复 OpenClaw Control UI 可访问性确认、飞书群聊免@自动回复、DM策略改为open方便操作、清理无效session。

### 详细变更

#### 1. 飞书群聊免@自动回复 (requireMention: true → false)
- **之前**: `channels.feishu.requireMention = true` — 群聊中必须@机器人才能触发回复
- **之后**: `channels.feishu.requireMention = false` — 群聊中任何消息都会触发机器人回复
- 目的：方便群聊使用，不需要每次@机器人
- ⚠️ 注意：机器人会对群聊中所有消息回复，可能较活跃。如需改为只回复@消息，设置 `requireMention: true`
- 修改方式: `openclaw config set channels.feishu.requireMention false`

#### 2. 飞书DM策略改为open (dmPolicy: pairing → open)
- **之前**: `channels.feishu.dmPolicy = "pairing"` — 需要pairing确认才能私聊
- **之后**: `channels.feishu.dmPolicy = "open"` — 任何人私聊机器人都会回复
- 目的：方便操作，不需要额外确认步骤
- 修改方式: `openclaw config set channels.feishu.dmPolicy open`

#### 3. Control UI 可访问性确认
- Gateway 绑定 `0.0.0.0:18789`，从 localhost (127.0.0.1) 和 LAN IP (192.168.1.107) 都能 HTTP 200 正常访问
- `controlUi.dangerouslyDisableDeviceAuth = true` — 禁用设备认证，方便从不同设备访问
- `controlUi.allowInsecureAuth = true` — 允许不安全认证，方便快速访问
- `controlUi.allowedOrigins = ["*"]` — 允许任何来源访问
- Auth token: `opclaw123` — 简单的认证token，方便记忆和输入
- 结论：Control UI 当前配置已满足方便操作的需求，无需额外修改

#### 4. Session 清理
- 清理了 1 个缺失 transcript 的 session（从 5 → 4）
- 命令: `openclaw sessions cleanup --enforce --fix-missing`

#### 5. Gateway 重启
- 修改 feishu 配置后重启 gateway 使配置生效
- 新 pid: 2224428, 状态 active
- 验证：openclaw status 显示所有组件正常，Feishu Channel ON/OK

### 当前状态
- OpenClaw Gateway: running (pid 2224428, 0.0.0.0:18789)
- Feishu Channel: ON/OK (WebSocket connected)
- Model: proxy40002/dsv4p (anthropic-messages API)
- Feishu: requireMention=false, dmPolicy=open, groupPolicy=open

### 验证方式
```bash
# 检查 feishu requireMention 配置
openclaw config get channels.feishu.requireMention  # 应输出 false

# 检查 Control UI
curl -s http://192.168.1.107:18789/ -o /dev/null -w "%{http_code}"  # 应输出 200

# 测试飞书消息发送（在飞书群聊中发消息，不用@机器人，观察是否自动回复）
```

### 下一步优化方向
- 在飞书群中实际测试免@回复是否生效
- 监控群聊免@后的回复频率和质量
- 优化 LiteLLM routing strategy
- 探索 OpenClaw 升级到 2026.6.5

## v0.1.0 (2026-06-05) — Initial Setup & Configuration

### 概述
初始版本：克隆仓库、分析本地 OpenClaw 完整链路、重新配置模型指向 proxy40002 的 dsv4p、修改 gateway bind 为 0.0.0.0、启用飞书插件、建立工程化文档。

### 详细变更

#### 1. 模型配置更新
- **之前**: OpenClaw 指向 `litellm41001/dsv4p_uni41001` (port 41001, openai-completions 格式)
  - 问题：port 41001 只有 glm5.1 模型，dsv4p 模型在 42001；配置指向了错误的 litellm 容器
- **之后**: OpenClaw 指向 `proxy40002/dsv4p` (port 40002, anthropic-messages 格式)
  - proxy40002 提供 Anthropic ↔ OpenAI 格式转换，路由 dsv4p 到 42001 litellm 容器
  - anthropic-messages 格式更适合 OpenClaw 的原生 API 通信
- 修改文件: `~/.openclaw/openclaw.json`, `~/.openclaw/agents/main/agent/models.json`

#### 2. Gateway 绑定地址更新
- **之前**: `gateway.bind = "lan"` → 绑定到 192.168.1.103 (LAN IP)
- **之后**: `gateway.bind = "custom", customBindHost = "0.0.0.0"` → 绑定所有接口
- 目的：方便从任何网络位置访问 Control UI (0.0.0.0:18789)
- 修改文件: `~/.openclaw/openclaw.json`

#### 3. 飞书插件启用
- **之前**: feishu plugin `enabled = false`
- **之后**: feishu plugin `enabled = true`
- 验证：openclaw status 显示 Feishu Channel ON/OK，WebSocket 已连接
- 修改文件: `~/.openclaw/openclaw.json`

#### 4. Docker 网络架构确认
- proxy40002: Anthropic 格式转换 + 输入安全 + 指标记录 → 路由 dsv4p 到 42001, glm5.1 到 41003
- proxy40001: 同 proxy40002 但端口 40001（给 Claude Code 用）
- dsv4p_uni42001: LiteLLM 11 variants × 7 keys = 77 deployments, latency-based routing
- glm5.1_test41003: LiteLLM 1000 variants × 7 keys = 7000 deployments (临时测试配额探索)

#### 5. 文档建立
- 创建 CLAUDE.md：完整架构文档、命令手册、工作流说明
- 创建 CHANGELOG.md：版本变更详细记录
- 创建 README.md 已有（仓库初始内容）

### 当前状态
- OpenClaw Gateway: running (pid 173040, 0.0.0.0:18789)
- Feishu Channel: ON/OK (WebSocket connected)
- Model: proxy40002/dsv4p (anthropic-messages API)
- 注意：ModelScope dsv4p 配额可能暂时耗尽 (429 rate limit)，这是 ModelScope 平台限制，非配置问题

### 下一步优化方向
- 监控 ModelScope 配额恢复情况
- 优化 proxy.py 的 Anthropic 流式转换性能
- 优化 LiteLLM routing strategy（当前 latency-based-routing）
- 探索增加更多模型源（OpenRouter 等）
- 优化飞书消息处理质量
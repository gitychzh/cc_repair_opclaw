# CHANGELOG

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
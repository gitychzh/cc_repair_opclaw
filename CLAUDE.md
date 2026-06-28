# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

cc_repair_opclaw — 长期优化 OpenClaw 龙虾机器人。每轮优化完成后建立版本号，push 到仓库，详细说明更新内容。工作流文档化方便后续接手者继续优化。

## Architecture Overview

### Full Chain: OpenClaw → passthrough-proxy → ms-gateway → ModelScope (v0.4.0, 本机)

```
User/Feishu → OpenClaw Gateway (0.0.0.0:18789, api: openai-completions, model glm5.1_ol)
    → /v1/chat/completions → proxy40003 (127.0.0.1:40003, passthrough-proxy, OpenAI format)
        [非流式请求强制 stream=true 上游, 流式收集后合成非流式 OpenAI 响应 — v0.4.0]
        [v×k 2D round-robin + burst 429 per-(variant,key) 冷却 + 全局出站节流 1.8s]
        → /v1/chat/completions → ms_uni41001 (127.0.0.1:41001, ms-gateway, 纯 Python 直连)
            → https://api-inference.modelscope.cn/v1/chat/completions (GLM-5.2, 10 variants × 7 keys = 70 deployments)
```

> v0.4.0 修正：`ms_uni41001` 已不是 LiteLLM，是 ~50MB 的 Python 直连 ModelScope 网关（`/opt/cc-infra/proxy/ms-gateway`），不依赖 DB。CLAUDE.md 早期版本里"41001→41002 双 LiteLLM 兜底"在本机部署中不存在（41002 容器未起）。GLM-5.2 与 GLM-5.1 在 ModelScope 均可用，链路实际打的是 GLM-5.2；模型名 `glm5.1_*` 仅为历史命名遗留。passthrough config 里 `glm5.2→glm5.1` 的"ModelScope dropped GLM-5.2"注释为过时错误信息。

### Key Components (本机实际部署)

| Component | Port | Role |
|-----------|------|------|
| OpenClaw Gateway | 18789 | AI agent platform, Feishu integration, Control UI |
| proxy40003 (auth_to_api_40003) | 40003 | **openclaw 主入口** passthrough proxy, OpenAI format, v×k cycling + burst 429 冷却 + 强制流式 |
| ms_uni41001 (ms-gateway) | 41001 | 纯 Python 直连 ModelScope, 70 deployments (GLM-5.2), 无路由/无重试/无 DB |
| cc_postgres | 5432 | LiteLLM 历史 DB (ms-gateway 不使用) |

> 其它容器（40001 cc-proxy / 40002 codex / 40005 cc-experiment / 40000 dispatcher / 40006 hm-proxy / 41101-41105 LiteLLM）为历史/备用链路，v0.4.0 未改动、未启用在 openclaw 主链路中。

### ModelScope 限速（实测）
- `Model-Requests-Limit: 200`（每 variant/天，独立配额）
- `Requests-Limit: 2000`（账户级/天总配额）
- 突发 429：`"Request rate increased too quickly"` — 短时并发限速，与日配额无关，是 429 主因
- 治理：全局出站节流 `MIN_OUTBOUND_INTERVAL_S` + per-(variant,key) `BURST_COOLDOWN_S` + v×k 轮询绕开冷却 slot

### OpenClaw Configuration

- Config: `~/.openclaw/openclaw.json`
- Agent models: `~/.openclaw/agents/main/agent/models.json`
- Workspace: `~/.openclaw/workspace/`
- Gateway systemd service: `~/.config/systemd/user/openclaw-gateway.service`
- Models provider: `litellm`, `api: openai-completions`, `baseUrl: http://127.0.0.1:40003/v1`, model id `glm5.1_ol` (contextWindow 170000)

### Proxy (passthrough-proxy, 40003) Key Features

1. **OpenAI passthrough** — 只服务 `/v1/chat/completions`（openclaw 直连，无 Anthropic 转换）
2. **Force-stream fix (v0.4.0 扩展到 OpenAI 路径)** — 非流式请求强制 `stream=true` 上游，`collect_stream_to_openai` 收集 SSE 后合成非流式 OpenAI 响应（ModelScope 非流式返回 `choices:null` 空响应）
3. **v×k 2D round-robin** — request N → variant_idx=(N//NUM_KEYS)%NUM_VARIANTS, key_idx=N%NUM_KEYS；429 时同 variant 内 key 循环；counter 持久化到 `rr_counter.json` 防重启回 v1k1
4. **Burst 429 冷却 (v0.4.0)** — burst 429 标记 (variant,key) 冷却 8s，轮询自动绕开；quota 耗尽不冷却
5. **Outbound throttle** — `MIN_OUTBOUND_INTERVAL_S=1.8` 全局出站节流，平滑突发
6. **Tool description truncation** — MAX_TOOL_DESC=2000 chars, MAX_SCHEMA_DESC=600 chars
7. **Input token safety** — estimates tokens from text content, rejects if over model limit
8. **Messages sequence fix** — messages 以 assistant 结尾时自动 append `{"role":"user","content":"Continue."}`（GLM 拒绝 assistant 结尾序列）

### Feishu Channel

- App ID: `cli_a9690bef46389cd4`
- Plugin: `@openclaw/feishu` (installed at `~/.openclaw/npm/node_modules/@openclaw/feishu/`)
- WebSocket long-connection for event receiving (connectionMode: websocket)
- Features: feishu_chat, feishu_doc, feishu_drive, feishu_perm, feishu_wiki, feishu_bitable
- **requireMention: false** — 群聊无需@机器人即可触发回复 (v0.2.0 起)
- **dmPolicy: open** — 任何人私聊都会回复，无需pairing确认 (v0.2.0 起)
- **groupPolicy: open** — 群聊开放，任何成员消息都会触发
- Auth token: `opclaw123` — Gateway 简单认证

### Docker Infrastructure

- Docker Compose: `/opt/cc-infra/docker-compose.yml`
- Proxy source: `/opt/cc-infra/proxy/proxy.py` + `/opt/cc-infra/proxy/Dockerfile`
- LiteLLM configs: `/opt/cc-infra/litellm-dsv4p/config.yaml`, `/opt/cc-infra/litellm-glm51/config.yaml`, `/opt/cc-infra/litellm-glm51-test/config.yaml`
- Logs: `/opt/cc-infra/logs/` (proxy, litellm-glm51, litellm-dsv4p, litellm-glm51-test)

## Commands

### OpenClaw Management

```bash
openclaw status                     # Check gateway, channels, models, sessions
openclaw status --deep              # Full probe including channel connectivity
openclaw models status              # Show configured model auth health
openclaw gateway restart            # Restart the gateway (pick up config changes)
openclaw doctor --fix               # Auto-fix config issues
openclaw config validate            # Validate openclaw.json
openclaw config get <path>          # Get a config value
openclaw config set <path> <value>  # Set a config value
openclaw agent --agent main --message "test"  # Test agent turn
openclaw channels status            # Show connected channels
openclaw logs --follow              # Live tail gateway logs
```

### Docker Infrastructure

```bash
cd /opt/cc-infra
docker compose ps                   # Check all containers
docker compose logs -f auth_to_api_40002  # Tail proxy logs
docker compose logs -f dsv4p_uni42001     # Tail dsv4p litellm logs
docker compose restart auth_to_api_40002  # Restart proxy
docker compose restart dsv4p_uni42001     # Restart dsv4p litellm
```

### Testing Model Connectivity

```bash
# Test dsv4p direct (streaming)
curl -s http://127.0.0.1:42001/v1/chat/completions \
  -H "Authorization: Bearer sk-litellm-local" \
  -H "Content-Type: application/json" \
  -d '{"model":"dsv4p","messages":[{"role":"user","content":"hello"}],"max_tokens":10,"stream":true}'

# Test via proxy (Anthropic format)
curl -s http://127.0.0.1:40002/v1/messages \
  -H "x-api-key: sk-litellm-local" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"dsv4p","max_tokens":10,"messages":[{"role":"user","content":"hello"}],"stream":true}'
```

## Version Control Workflow

### Rules

1. **每轮优化完必须建立版本号** — 用 git tag 标记版本 (v0.1.0, v0.2.0, ...)
2. **Push 到仓库** — 每次版本更新 push 到 `git@github.com:gitychzh/cc_repair_opclaw.git`
3. **详细说明更新内容** — commit message 和 CHANGELOG.md 中详细记录
4. **工作流写清楚** — 方便别人接手继续优化

### Workflow Per Optimization Round

1. Make changes (config, code, docs)
2. Update CHANGELOG.md with detailed changes
3. Git commit with detailed message
4. Git tag with version number
5. Git push + push tags
6. Record in this repo what changed, why, and how to verify

### Git Commands

```bash
git add -A
git commit -m "v0.X.0: 详细描述更新内容"
git tag v0.X.0
git push origin main
git push origin v0.X.0
```

## Important Notes

- **ModelScope quota limits**: dsv4p has ~200/id/day per variant, glm5.1 similar. Rate limits (429) are expected when quotas exhausted.
- **dsv4p non-stream bug**: ModelScope returns invalid `delta` field in non-stream responses → LiteLLM crashes. The proxy force-stream fix resolves this.
- **Proxy does NOT retry** — all retry/fallback/routing is delegated to LiteLLM upstream.
- **OpenClaw uses `anthropic-messages` API** to connect to proxy40002, proxy converts to OpenAI format for LiteLLM.
- **SSH clone required**: HTTPS clone fails (GitHub access issues), use `git@github.com:gitychzh/cc_repair_opclaw.git`
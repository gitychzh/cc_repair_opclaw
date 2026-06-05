# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

cc_repair_opclaw — 长期优化 OpenClaw 龙虾机器人。每轮优化完成后建立版本号，push 到仓库，详细说明更新内容。工作流文档化方便后续接手者继续优化。

## Architecture Overview

### Full Chain: OpenClaw → Proxy → LiteLLM → ModelScope

```
User/Feishu → OpenClaw Gateway (0.0.0.0:18789)
    → Anthropic API → proxy40002 (127.0.0.1:40002, format conversion)
        → OpenAI API → dsv4p_uni42001 LiteLLM (127.0.0.1:42001, routing/fallback/retry)
            → ModelScope API (dsv4p, 11 variants × 7 keys = 77 deployments)
        → OpenAI API → glm5.1_test41003 LiteLLM (127.0.0.1:41003, routing/fallback/retry)
            → ModelScope API (glm5.1, 1000 variants × 7 keys = 7000 deployments)
```

### Key Components

| Component | Port | Role |
|-----------|------|------|
| OpenClaw Gateway | 18789 | AI agent platform, Feishu integration, Control UI |
| proxy40002 (auth_to_api_40002) | 40002 | Anthropic ↔ OpenAI format conversion, metrics logging, input safety |
| proxy40001 (auth_to_api_40001) | 40001 | Same proxy, port 40001 (for Claude Code) |
| dsv4p_uni42001 (LiteLLM) | 42001 | DeepSeek V4 Pro routing, 77 deployments |
| glm5.1_uni41001 (LiteLLM) | 41001 | GLM-5.1 routing, 1792 deployments (legacy, not primary routing) |
| glm5.1_test41003 (LiteLLM) | 41003 | GLM-5.1 test routing, 7000 deployments (primary glm5.1 routing) |
| cc_postgres | 5432 | LiteLLM persistence DB (3 DBs: litellm_glm51, litellm_dsv4p, litellm_glm51_test) |

### OpenClaw Configuration

- Config: `~/.openclaw/openclaw.json`
- Agent models: `~/.openclaw/agents/main/agent/models.json`
- Workspace: `~/.openclaw/workspace/`
- Gateway systemd service: `~/.config/systemd/user/openclaw-gateway.service`

### Proxy (proxy.py) Key Features

1. **Anthropic ↔ OpenAI format conversion** — converts `/v1/messages` (Anthropic) to `/v1/chat/completions` (OpenAI)
2. **Force-stream fix** — all non-stream requests forced to `stream=true` upstream (ModelScope non-stream returns invalid `delta` field)
3. **Model name mapping** — Claude model names (claude-opus-4-8, etc.) mapped to glm5.1/dsv4p
4. **Tool description truncation** — MAX_TOOL_DESC=2000 chars, MAX_SCHEMA_DESC=600 chars
5. **Input token safety** — estimates tokens from text content, rejects if over model limit

### Feishu Channel

- App ID: `cli_a9690bef46389cd4`
- Plugin: `@openclaw/feishu` (installed at `~/.openclaw/npm/node_modules/@openclaw/feishu/`)
- WebSocket long-connection for event receiving
- Features: feishu_chat, feishu_doc, feishu_drive, feishu_perm, feishu_wiki, feishu_bitable

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
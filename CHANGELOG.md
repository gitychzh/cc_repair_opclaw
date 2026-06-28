# CHANGELOG

## v0.4.0 (2026-06-28) — passthrough OpenAI 路径强制流式修空响应 + 突发 429 冷却治理

### 概述
本机 openclaw 模型链路 `openclaw → 40003 (passthrough) → ms_uni41001 (ms-gateway) → ModelScope GLM-5.2` 的两项关键修复：
1. **修复空响应 bug**：openclaw 发非流式 `/v1/chat/completions` 请求时，passthrough proxy 原先直接透传 ModelScope 非流式响应，而 ModelScope 非流式返回 `choices:null / completion_tokens:0`（"delta bug"），导致 openclaw 拿到空回复。现已对 OpenAI 路径也强制 `stream=true` 上游、流式收集后合成真正的非流式 OpenAI 响应。
2. **突发 429 治理**：ModelScope 的 429 主要是 `"Request rate increased too quickly"`（短时并发突发限速），而非日配额耗尽。新增 per-(variant,key) 突发冷却：burst 429 时标记该 slot 冷却 8s，轮询自动绕开；全局出站间隔 1.5→1.8s 平滑突发。

### 背景（实测结论，修正 CLAUDE.md 旧描述）
- `ms_uni41001` 已不是 LiteLLM，是 ~50MB Python 直连 ModelScope 的轻量网关（`/opt/cc-infra/proxy/ms-gateway`）。10 variants × 7 keys = 70 deployments，variant ID 全是 `ZHIPUAI/*-5.2` 大小写变体，每个 variant 独立 200/天 配额。
- CLAUDE.md 里"41001→41002 双 LiteLLM 兜底"在本机不存在（41002 容器未起，ms-gateway 不依赖 DB）。
- passthrough config 里 `glm5.2→glm5.1` 的注释"ModelScope dropped GLM-5.2"是过时错误信息——实测 GLM-5.2 和 GLM-5.1 都在 ModelScope 可用，链路实际打到的就是 GLM-5.2。模型名 `glm5.1_*` 仅为历史命名遗留。
- ModelScope 限速头实测：`Model-Requests-Limit: 200`（每 variant/天）、`Requests-Limit: 2000`（账户级/天）、突发 429 与日配额无关。
- openclaw 配置 `api: openai-completions` 直连 40003 `/v1/chat/completions`，不经 cc-proxy 的 Anthropic 转换。

### 详细变更

#### 1. `proxy/passthrough-proxy/gateway/stream.py` — 新增 `collect_stream_to_openai()`
- 镜像已有 `collect_stream_to_anth` 的 SSE 解析逻辑，合成 **OpenAI** 非流式响应：`choices[0].message{role,content,tool_calls,reasoning_content}` + `usage` + `finish_reason`。
- 对 `delta`/`tool_calls`/`choices` 为 `None` 的 chunk 做防御（ModelScope 流式 chunk 里这些字段常显式为 null）。

#### 2. `proxy/passthrough-proxy/gateway/handlers.py` — OpenAI 路径强制流式
- `_handle_chat_completions`：非流式请求设 `force_stream_for_nonstream=True`，`body["stream"]=True`，加 `stream_options.include_usage`。
- 成功路径新增 `elif force_stream_for_nonstream:` 分支调用 `collect_stream_to_openai`（原先非流式直接 `resp.read()` 透传空响应）。
- 导入 `collect_stream_to_openai`。

#### 3. `proxy/passthrough-proxy/gateway/config.py` — per-(variant,key) 突发冷却
- 新增 `BURST_COOLDOWN_S`（默认 8s，env 可调）、`_ms_cooldown` 表、`mark_ms_cooldown(v,k)`、`_ms_slot_cooled_down()`。
- `_next_variant_key_pair` 返回前：若选中的 MS slot 处于冷却期，推进 round-robin 直到找到未冷却 slot（或扫遍整轮 70 slot）。

#### 4. `proxy/passthrough-proxy/gateway/upstream.py` — burst 429 标记冷却
- MS 429 处理：`cycle_reason="429_rate_limit"`（非 quota）时调用 `mark_ms_cooldown(start_variant_idx, current_key_idx)`。quota 耗尽型不冷却（冷却无意义，靠 key-cycling 解决）。
- 导入 `mark_ms_cooldown`。

#### 5. `docker-compose.yml` — 40003 env 调整
- `MIN_OUTBOUND_INTERVAL_S`: 1.5 → **1.8**（平滑突发）。
- 新增 `BURST_COOLDOWN_S: "8"`。

### 不动的部分（按用户决策）
- 不动 openclaw 配置（仍 `openai-completions` → 40003）。
- 不删/不改其它容器（40001/40002/40005/40000/40006/41101-41105 全部保留）。
- 不改 ms-gateway（已是纯直连，无 429 日志，工作正常）。

### 验证（本机实测通过）
- 非流式：`curl 40003 ... stream:false` → `content="OK"`, `usage` 正常（修复前为空）。
- 流式：`stream:true` → 正常 SSE。
- 连发 15 次非流式：15/15 成功，0 空响应、0 个 429。
- `openclaw agent --agent main --message "回复两个字：你好"` → 真实回复"你好"。
- metrics.jsonl：`force_stream_collect_success=True`、v×k 轮询正常推进、`finish_reason=stop`、`output_tokens` 正常。

### 回滚
- `git revert` + `cd /opt/cc-infra && docker compose up -d --build auth_to_api_40003`。仅影响 40003 单容器。


## v0.3.0 (2026-06-25) — 远程模型链路改为主备: openclaw → 40003 → 41001(默认) + 41002(兜底)，仅用 ModelScope GLM-5.2

### 概述
远程主机 `opc2sname-tailscale` (用户 opc2_uname) 的 openclaw 模型链路从「单 LiteLLM」改为「主备双 LiteLLM」：
- **openclaw → 40003 (passthrough-proxy) → 41001 (默认) → 41001 全 key 耗尽时兜底到 41002**
- 41002 = 41001 的 config copy（独立目录/容器/DB），作为「旧版本兜底」——后续持续优化 41001 时不影响 41002
- 上下游**只用 ModelScope GLM-5.2**（不走 NV）
- **仅改远程部署，本地仓库不动部署逻辑**

### 背景
- 远程 `ms_uni41001` 的 config.yaml 已先期改为 GLM-5.2（70 dep，case-permutation variant × 7 keys），这是「相关代码已改」的部分
- passthrough-proxy (40003) 历史上 (R29 之前) 曾有 `ms_uni41002` 兜底，R29 移除了；本版本针对 GLM-5.2 重新恢复并明确主备语义

### 详细变更（均在远程 `/opt/cc-infra`）

#### 1. 新建 41002 独立 config 目录
- `mkdir -p /opt/cc-infra/litellm-glm51-fb` + `logs/litellm-glm51-fb`
- `cp litellm-glm51/config.yaml litellm-glm51-fb/config.yaml`（与 41001 完全一致，GLM-5.2，70 dep）
- 头注释改为标明 41002 fallback 身份：`# 41002 LiteLLM Config — glm5.2 (70 dep), rpm=1 [FALLBACK copy of 41001, R46]`
- 设计意图：41002 文件独立，后续优化 41001 的 config 时此兜底版本保持不动

#### 2. docker-compose.yml: 新增 `ms_uni41002` 服务
- 复制 `ms_uni41001` 块，改动：
  - `container_name: ms_uni41002`，`ports: 41002:4000`
  - `DATABASE_URL` → 独立 DB `litellm_glm51_fallback`（复用历史遗留的空 DB，与 41001 router state 隔离）
  - volumes: `./litellm-glm51-fb/config.yaml` + `./logs/litellm-glm51-fb`
  - env (MS_KEY1-7 / MS_BASEURL / LITELLM_MASTER_KEY) 与 41001 一致
  - `depends_on: cc_postgres (service_healthy)`
- 备份: `docker-compose.yml.bak.R46-<ts>`

#### 3. passthrough-proxy config.py: 新增 fallback 环境变量
- `LITELLM_URL_GLM51_FB` / `LITELLM_MODELS_URL_GLM51_FB` / `LITELLM_FB_ENABLED`
- 空字符串 → 禁用（向后兼容其他 proxy role / 其他部署，已用 docker import 测试验证）
- 关键修正：`_ensure_url_path("")` 会返回 `/v1/chat/completions`（非空）导致误启用，故用 `LITELLM_FB_RAW = env.strip()` 先判空再 `_ensure_url_path`

#### 4. passthrough-proxy upstream.py: 插入 41002 兜底逻辑
- import 新增 `LITELLM_URL_GLM51_FB, LITELLM_FB_ENABLED`
- 在 MS primary `_try_ms_keys` 全部失败后、NV 不可用的 `return result` 之前插入：
  ```
  if LITELLM_FB_ENABLED and result.all_keys_exhausted:
      _log("MS-FB-FALLTHROUGH", "All 41001 MS keys exhausted → trying 41002 fallback")
      result = _try_ms_keys(..., LITELLM_URL_GLM51_FB, litellm_model_base)
      ...
      return result
  ```
- 复用现有 `_try_ms_keys`（其 `litellm_url` 参数控制上游），41002 config 与 41001 model_name 一致（`glm5.1v{V}k{K}`），可直接复用
- 更新文件头 R29 注释为「R46: Restored 41002 fallback (GLM-5.2 copy of 41001)」
- 备份: `config.py.bak.R46-<ts>` / `upstream.py.bak.R46-<ts>`

#### 5. docker-compose.yml: 给 40003 加 fallback env + depends_on
- `LITELLM_URL_GLM51_FB: http://ms_uni41002:4000/v1/chat/completions`
- `LITELLM_MODELS_URL_GLM51_FB: http://ms_uni41002:4000/v1/models`
- `depends_on` 追加 `ms_uni41002`

### 兜底触发语义（已与用户确认）
- **全 key 耗尽才兜底**：41001 内部 v×k 轮转 + 429/conn-error cycling 把 7 个 key 全部试完后，才切到 41002 再跑一轮
- 41001 与 41002 共用同一组 ModelScope key+variant → **配额共享**，兜底主要价值是「41001 容器/配置故障或全部 429 时保活」，不是额外配额
- 已知权衡：41001 容器整体宕机时，需走完 7 key × ~7-8s 超时 ≈ 54s 才切到 41002（既有 cycling 设计；后续可优化为「连接 refused 快速判定容器死」）

### 部署
```bash
cd /opt/cc-infra
docker compose up -d ms_uni41002          # 新建并启动 41002
docker compose up -d --build auth_to_api_40003  # 重建 40003（代码改了）
```

### 测试验证（全部通过）
1. **41002 独立健康**：`/health/liveliness` 200；`/v1/models` 返回 70 个 model
2. **41002 直连推理**：`curl 41002/v1/chat/completions -d '{"model":"glm5.1v1k1",...,"stream":true}'` 流式返回 GLM-5.2 内容 ✓
3. **40003 正常路径**（走 41001）：`curl 40003 -d '{"model":"glm5.1_ol",...}'` 流式返回，日志显示 `glm5.1v5k5`（41001）✓
4. **40003 兜底路径**（模拟 41001 故障）：`docker stop ms_uni41001` → 请求 40003 → 日志出现 `[MS-FB-FALLTHROUGH] All 41001 MS keys exhausted → trying 41002 fallback` → 从 41002 (`glm5.1v5k6`) 成功返回 ✓；之后 `docker start ms_uni41001` 恢复
5. **openclaw 端到端**：`openclaw agent --agent main --message "..."` 回复「我现在能正常工作，当前运行的模型是 proxy-gateway/glm5.1_ol」✓
6. **openclaw status --deep**：Gateway reachable 196ms，飞书 Channel ON/OK ✓

### 当前状态
- 三个容器均 healthy：`ms_uni41001` (41001, 默认) / `ms_uni41002` (41002, 兜底) / `auth_to_api_40003` (40003, passthrough)
- openclaw: `providers.proxy-gateway.baseUrl=http://127.0.0.1:40003/v1`，model id `glm5.1_ol` → 40003 → 41001 → ModelScope GLM-5.2
- 链路：openclaw → 40003 → 41001(默认) + 41002(全 key 耗尽兜底)

### 回滚
- compose: `cp docker-compose.yml.bak.R46-<ts> docker-compose.yml && docker compose up -d`
- 代码: `cp proxy/passthrough-proxy/gateway/{config,upstream}.py.bak.R46-<ts> ...` 后 `docker compose up -d --build auth_to_api_40003`
- 41002 服务: `docker compose stop ms_uni41002`（保留容器/config 以便再启用）

### 下一步优化方向
- 41001 容器整体宕机时快速判定（连接 refused → 立即切 41002，不逐 key 超时）
- 持续优化 41001 的 config（variant/key 调度、rpm），41002 作为稳定兜底不动
- 监控 41002 实际被触发的频率（应接近 0，仅在 41001 故障时）

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
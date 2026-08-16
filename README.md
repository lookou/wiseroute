# Wisro · 微路由

**Wisro — the wise router for every model. 见微知著，智能选路。**
**私有化 AI 网关。协议兼容、路由稳态、记账清晰。**

为企业与代理运营设计：五分钟起最小可用；Codex / Claude / OpenAI SDK / Ark 改地址即可接入；渠道故障自动切流并按冷却恢复；用量与毛利一本账。未登录打开控制台根路径即为宣传页。

> 品牌名 **Wisro（微路由）**；代码内部标识符、二进制名（`xapi-gateway` 等）、环境变量（`XAPI_*`）、配置文件名与 Redis 频道仍沿用 `xapi`，属运行时契约，未随品牌重命名。

- 最小可用：[docs/QUICKSTART.md](docs/QUICKSTART.md)
- 路由与切流：[docs/ROUTING.md](docs/ROUTING.md)
- Codex / Claude 直连：[docs/PASSTHROUGH.md](docs/PASSTHROUGH.md)

C++20 高并发低时延：OpenAI / Claude 转发、多账号选路、Token 计费、Seedance / Hailuo（MiniMax）视频任务。

## 支持平台

| 平台 | 状态 | 说明 |
|------|------|------|
| Ubuntu 22.04 / 24.04 | 一等支持 | `scripts/install-deps-ubuntu.sh` |
| Rocky / Alma / CentOS Stream 9 | 一等支持 | `scripts/install-deps-centos.sh`，镜像见 `Dockerfile.centos` |
| Rocky / Alma / CentOS Stream 8 | 支持 | 需 `gcc-toolset-12` + Boost ≥ 1.70 |
| CentOS 7 | 不支持 | 默认 GCC 过旧，无官方 C++20 工具链 |

依赖：CMake ≥ 3.16，GCC ≥ 10 或 Clang ≥ 12，OpenSSL，Boost.Asio/Beast（≥ 1.70 推荐）。

## 架构

- `xapi-gateway`：热路径转发（鉴权、内存选路、SSE、计费预留）
- `xapi-control`：管理 API（用户/Token/渠道/价格/别名）
- `xapi-worker`：视频任务轮询与失败退款

热路径使用不可变 `RouteSnapshot`。账本支持三种 SQL 后端（互斥，选一个）：

| 后端 | `store_backend` | 构建开关 | 配置 | 迁移目录 |
|------|-----------------|----------|------|----------|
| SQLite（本地默认） | `sqlite` | `-DXAPI_WITH_SQLITE=ON`（默认） | `sqlite_path` | `migrations/sqlite` |
| MySQL 8+ | `mysql` | `-DXAPI_WITH_MYSQL=ON` | `mysql_dsn` | `migrations/mysql` |
| PostgreSQL | `postgres` | `-DXAPI_WITH_POSTGRES=ON` | `postgres_dsn` | `migrations/` |

JSON 文件账本（`store.json` / `store_backend=file` / `memory`+非空 `data_dir`）**永久禁用**。  
gateway/control/worker 轮询 `config_versions.route` 刷新 Snapshot。  
纯内存 `memory` 仅单测（空 `data_dir` + `allow_memory_store` / `XAPI_ALLOW_MEMORY_STORE=1`）。  
三 SQL 后端共用 `sql_common.hpp`（解析 / migrations / mirror 原子写）与 `sql_row_mappers.hpp`（行→实体）；方言 SQL 仍分文件。

### 协议 / 能力切片（相对 new-api）

| 能力 | 状态 |
|------|------|
| OpenAI chat/responses/embeddings/audio/moderations/images | 有 |
| Claude messages / `/anthropic` | 有；渠道 `type=claude` 或 `anthropic` |
| Seedance / Hailuo 视频任务 | 有（Hailuo 模型如 `MiniMax-Hailuo-2.3`；无独立「H3」适配器） |
| Rerank `POST /v1/rerank` | 有（透传） |
| Realtime | `POST /v1/realtime/sessions`（及 `client_secrets`）HTTP 透传上游；`GET /v1/realtime` WS 双向代理；结算解析 `response.done` usage（无则 per-call） |
| Gemini / Vertex | 渠道 `gemini`/`google`/`vertex`/`vertex_ai`：聊天走 `/v1beta/openai/chat/completions`；原生 `POST /v1beta/models/…:generateContent`（`stream=true` 改打 `streamGenerateContent`）；AI Studio 用 `x-goog-api-key`；Vertex 渠道 Key 可为 **GCP SA JSON**（RS256 JWT → OAuth access_token，约 30min 缓存）或普通 Bearer；请求带 `x-goog-user-project`；Veo 走 `projects/…/locations/…/publishers/google/models/…` |
| Kling | TaskAdaptor + 原生 `POST /kling/v1/videos/text2video|image2video`、`GET /kling/v1/videos/{id}`；`mock://` 可用 |
| Midjourney | `midjourney_proxy` / `midjourney_discord` → proxy-ac；客户端可走原生 `/mj/submit/*`、`/mj/task/{id}/fetch`（或 `/v1/videos` + `metadata.mj`） |
| Suno | TaskAdaptor + 原生 `/suno/submit/music|lyrics`、`/suno/fetch/{id}`（需兼容中转） |
| Sora | `type=sora`/`openai_video`：`POST /v1/videos`、`POST /v1/videos/{id}/remix` |
| 即梦 Jimeng | `type=jimeng`（`access\|secret` HMAC 或 `sk-` Bearer）；原生 `/jimeng/submit` |
| Vidu | `type=vidu`：`Authorization: Token …`；`POST /ent/v2/text2video|img2video|start-end2video|reference2video`（按图数量），轮询 `GET /ent/v2/tasks/{id}/creations`；默认 `base_url=https://api.vidu.cn`；`mock://vidu` 可用 |
| 通义万相 Ali Wan | `type=ali`/`dashscope`/`wan`：DashScope 异步 `video-synthesis` + `GET /api/v1/tasks/{id}`；`X-DashScope-Async: enable`；`mock://ali` 可用 |
| 通义千问 Chat | `type=qwen` / `dashscope_chat`：DashScope OpenAI 兼容 `https://dashscope.aliyuncs.com/compatible-mode`；与 `openai_compatible` 同一上游 adaptor |
| Gemini Veo | 渠道 `gemini`/`google`/`vertex` 视频走 `predictLongRunning`（chat 仍走 GeminiUpstream） |
| 兑换码 | Admin `/api/redemption/`；用户 `POST /api/user/topup` `{"key":…}`（控制台「兑换码」「流水」） |
| 分组倍率 | settings.`group_ratio`（如 `{"default":1,"vip":0.5}`）乘到预扣/结算；`GET /api/group/` |
| 支付 | Admin `/api/billing/topup`；用户自助 Stripe `POST /api/user/stripe/pay` + webhook；控制台流水页 **Stripe Elements** 收银台（需 `STRIPE_PUBLISHABLE_KEY`）；易支付 Epay `POST /api/user/epay/pay` + `GET\|POST /api/user/epay/notify` |
| Azure OpenAI | 渠道 `type=azure`：`api-key` + deployments Chat/Embeddings；**Responses** `/openai/v1/responses`、**compact** `/openai/v1/responses/compact`（`cognitiveservices.azure.com` 无 `v1` 段）；Key 可 `key\|api-version` |
| AWS Bedrock | 渠道 `bedrock`/`aws`：默认 OpenAI `POST /v1/chat/completions`（SigV4/Bearer；OpenAI 透传与 **Claude↔OpenAI 协议转换** 流式均可真增量 flush；Responses 仍整段缓冲）；**Nova** 走 Invoke `messages-v1`（非流式）；**Embeddings** Titan/Cohere 走 Invoke → OpenAI `/v1/embeddings` 形状；跨区自动加 `us.`/`eu.`/`apac.`；Key=`AK\|SK\|region` 或 `APIKEY\|region` |
| 渠道亲和 / RPM / 自动恢复 | settings：`channel_affinity_*`（默认开）；`rate_limit_*`（成功次数窗口，超限 429）；`channel_auto_recover_seconds`（默认 300，自动摘除后冷却再启用；0=只许人工恢复）。说明见 [docs/ROUTING.md](docs/ROUTING.md) |
| Playground | 控制台「试用」页；`POST /pg/chat/completions` 用 dashboard JWT（`aud=xapi-dashboard`），计费扣用户首个可用 API 令牌（无则自动建 `playground` 令牌）；control 同进程转发同路径以便同源 |
| 两步验证 | 控制台账户页绑定 TOTP；登录密码通过后需 `/api/user/login/2fa`；无第三方 OAuth |
| Redis | 启动 `PING`；配置变更 `PUBLISH xapi:config:route`；gateway/control/worker `SUBSCRIBE` 后立刻刷 RouteSnapshot；仍保留 `snapshot_reload_ms` 轮询兜底 |

本地起服务：`./scripts/run_local.sh`（始终用 `configs/xapi.local.yaml`；8080/8081 被占时用环境变量改端口为 18080/18081，不另写 yaml）。然后  
`XAPI_GATEWAY=… XAPI_CONTROL=… ./scripts/smoke_live.sh`。  
**产品化**（真上游门禁，跳过 `mock://`）：`./scripts/smoke_live.sh --with-live-channels`（说明见 [docs/PRODUCTIZATION.md](docs/PRODUCTIZATION.md)）。

### Midjourney / Suno 渠道

渠道、密钥、模型与卖价写在 **SQL ledger**（控制台或 `/api/channels`），**不要**写进 yaml。

**Midjourney**（推荐 `type=midjourney_proxy`；`midjourney_discord` / 旧别名 `midjourney`|`mj` 同 adaptor）：

- 上游以本地 **midjourney-proxy-ac** 为准（`context-path=/mj`），不是早期 GitHub 单 `base64` 版。
- `base_url`：proxy 主机根（如 `http://127.0.0.1:8080`）；本地 mock：`mock://mj`。
- `api_key`：proxy 的 **`mj-api-secret`**（请求头同名；勿把 Discord user-token 填进 xapi）。
- Discord 的 guild/channel/token 只配在 proxy-ac。
- 客户端 Imagine：`POST /v1/videos`，`{"model":"midjourney","prompt":"a cat --ar 16:9"}`；垫图用 `metadata.base64Array` / `input_reference`。
- 进阶（对齐 proxy-ac）：在 `metadata.mj` 里设 `op`：
  - `action`：`taskId` + `customId`（或 `parent_task_id`=xapi 任务 id 自动换成上游 taskId）
  - `change`：`taskId` + `action`=`UPSCALE|VARIATION|REROLL` + `index`
  - `simple-change`：`content`（如 `"<id> U1"`）
  - `describe`：`base64`
  - `blend`：`base64Array` + `dimensions`（`SQUARE` 等）

**Suno**（`type=suno`）：上游兼容 `/suno/submit/music` + `/suno/fetch/{id}`；`base_url` 主机根或 `mock://suno`；`models` 如 `chirp-v4`。客户端：`POST /v1/videos`，`{"model":"chirp-v4","prompt":"...","metadata":{"title":"...","tags":"indie"}}`。

### Stripe 充值

环境变量（勿写入仓库）：

| 变量 | 用途 |
|------|------|
| `STRIPE_SECRET_KEY` | 创建 PaymentIntent / confirm 拉取 |
| `STRIPE_WEBHOOK_SECRET` | 校验 `Stripe-Signature`（Dashboard 或 `stripe listen` 给出的 `whsec_…`） |

流程（API；控制台尚未内嵌 Stripe Elements）：

1. 管理端 `POST /api/billing/topup`，body 含 `"provider":"stripe","user_id":…,"delta":…`（可选 `amount_cents` / `currency`）→ 返回 `client_secret`。
2. 自有前端或脚本用 Stripe.js / Payment Element，以 `client_secret` 完成支付。
3. Stripe 回调 control：`POST /api/billing/stripe/webhook`（事件 `payment_intent.succeeded`）→ 按 metadata `user_id` / `delta_quota` 入账；ledger `reason=stripe:<pi_id>` 防重复。
4. 本地无公网隧道时：管理端 `POST /api/billing/stripe/confirm`，`{"payment_intent_id":"pi_…"}` 向 Stripe 拉 PI 后同样入账。

Webhook URL 指向 **control** 端口，例如 `https://<host>:<control>/api/billing/stripe/webhook`。

### 两步验证（TOTP）

不做第三方 OAuth。用户在控制台「账户」页绑定认证器：

1. `POST /api/user/2fa/setup` → `secret` + `otpauth_url`
2. `POST /api/user/2fa/enable` `{"code":"123456"}` → 启用
3. 之后 `POST /api/user/login` 在密码正确时返回 `requires_2fa` + `pending_token`（无 access_token）
4. `POST /api/user/login/2fa` `{"pending_token":"...","code":"..."}` → 正式会话
5. 关闭：`POST /api/user/2fa/disable`（需密码 + 当前验证码）

## 首 Token（TTFT）指标

- `ingress_us`：鉴权 + 选路 + 额度预留
- `upstream_us`：供应商等待
- `flush_us`：首字节转发

市场对比只使用网关增量：`ingress_us + flush_us`。

## Ubuntu 快速开始

```bash
./scripts/install-deps-ubuntu.sh
# 本地默认 SQLite（无需 Docker）。生产/多机再用 Postgres：
# WITH_POSTGRES=1 ./scripts/install-deps-ubuntu.sh && docker compose up -d postgres
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release   # SQLite ON by default
# 可选: -DXAPI_WITH_POSTGRES=ON
cmake --build build -j
./build/xapi-gateway -c configs/xapi.local.yaml   # 本地默认 sqlite
```

## CentOS / Rocky / Alma 快速开始

```bash
sudo ./scripts/install-deps-centos.sh
# Rocky/Alma 8 如提示 toolset：
#   source /opt/rh/gcc-toolset-12/enable
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
./build/xapi-gateway -c configs/xapi.example.yaml
```

默认 admin/token：`sk-xapi-admin`

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H "Authorization: Bearer sk-xapi-admin" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","stream":true,"messages":[{"role":"user","content":"hi"}]}'
```

视频（OpenAI 风格）：

```bash
curl http://127.0.0.1:8080/v1/videos \
  -H "Authorization: Bearer sk-xapi-admin" \
  -H "Content-Type: application/json" \
  -d '{"model":"SeeDance","prompt":"a cat","seconds":5}'
```

视频（[火山方舟原生接口](https://www.volcengine.com/docs/82379/1520757?lang=zh)）：

```bash
# 创建
curl http://127.0.0.1:8080/api/v3/contents/generations/tasks \
  -H "Authorization: Bearer sk-xapi-admin" \
  -H "Content-Type: application/json" \
  -d '{
    "model":"doubao-seedance-2-0-260128",
    "content":[{"type":"text","text":"一只柯基在草地上奔跑"}],
    "ratio":"16:9",
    "duration":5,
    "resolution":"720p",
    "watermark":false
  }'

# 查询单个
curl "http://127.0.0.1:8080/api/v3/contents/generations/tasks/<task_id>" \
  -H "Authorization: Bearer sk-xapi-admin"

# 列表筛选（最近 7 天）
curl "http://127.0.0.1:8080/api/v3/contents/generations/tasks?page_num=1&page_size=20&filter.status=succeeded" \
  -H "Authorization: Bearer sk-xapi-admin"

# 取消排队中任务 / 删除已结束任务记录
curl -X DELETE "http://127.0.0.1:8080/api/v3/contents/generations/tasks/<task_id>" \
  -H "Authorization: Bearer sk-xapi-admin"
```

上游渠道 `type=volcengine`（或 `doubao_video`），`base_url=https://ark.cn-beijing.volces.com`。

图片（[火山方舟 Seedream](https://www.volcengine.com/docs/82379/1541523)）：

```bash
curl http://127.0.0.1:8080/api/v3/images/generations \
  -H "Authorization: Bearer sk-xapi-admin" \
  -H "Content-Type: application/json" \
  -d '{
    "model":"Seedream",
    "prompt":"星际穿越，黑洞，电影级光影，2K",
    "size":"2K",
    "watermark":false,
    "response_format":"url"
  }'
```

别名默认走低成本 `seedream-4.0` → `doubao-seedream-4-0-250828`；另有 `seedream-4.5` / `seedream-5.0-*`。兼容路径：`POST /v1/images/generations`。

真机联调（Key 用环境变量，勿写入仓库）：

```bash
export ARK_API_KEY='ark-xxx'
./scripts/live-ark-smoke.sh   # Seedream 4.0 + size=1K 单图
```

## OpenAI + Anthropic 双协议（Claude Code / Codex）

网关同时兼容 OpenAI Chat Completions、OpenAI Responses，以及 Anthropic Messages（对齐 [DeepSeek Anthropic API](https://api-docs.deepseek.com/guides/anthropic_api/)）。

| 协议 | 路径 | 鉴权 |
|------|------|------|
| OpenAI Chat | `POST /v1/chat/completions` | `Authorization: Bearer sk-…` |
| OpenAI Responses（Codex） | `POST /v1/responses` | Bearer |
| OpenAI Responses compact | `POST /v1/responses/compact` | Bearer；Azure/OpenAI 原生透传 |
| OpenAI Embeddings | `POST /v1/embeddings` | Bearer |
| OpenAI Audio | `POST /v1/audio/speech\|transcriptions\|translations` | Bearer（JSON 或 multipart） |
| OpenAI Moderations | `POST /v1/moderations` | Bearer |
| OpenAI Images | `POST /v1/images/generations\|edits\|variations`（及 Ark `/api/v3/images/generations`） | Bearer |
| Anthropic Messages | `POST /v1/messages` 或 `POST /anthropic/v1/messages` | `x-api-key` 或 Bearer |
| Anthropic count_tokens | `POST /v1/messages/count_tokens`（含 `/anthropic` 前缀） | 同上 |
| Claude Code Hello | `HEAD/GET /api/hello`（含 `/anthropic/api/hello`） | 无需鉴权 |

### Claude Code

```bash
export ANTHROPIC_BASE_URL=http://127.0.0.1:8080/anthropic
export ANTHROPIC_AUTH_TOKEN=sk-xapi-admin
unset ANTHROPIC_API_KEY

# 连接预热（Claude Code 会发 HEAD）
curl -I "$ANTHROPIC_BASE_URL/api/hello"

curl "$ANTHROPIC_BASE_URL/v1/messages" \
  -H "x-api-key: $ANTHROPIC_AUTH_TOKEN" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{
    "model":"claude-sonnet-4-0",
    "max_tokens":64,
    "messages":[{"role":"user","content":"hi"}]
  }'
```

### Codex CLI

`~/.codex/config.toml` 示例（`wire_api = "responses"`）：

完整示例见 [`configs/codex.xapi.example.toml`](configs/codex.xapi.example.toml) + [`configs/codex_model_catalog.xapi.example.json`](configs/codex_model_catalog.xapi.example.json)（`model_catalog_json`，避免 CLI 报 Model metadata not found）。

```toml
model_provider = "xapi"
model = "gpt-4o-mini"
model_context_window = 128000
model_max_output_tokens = 16384
model_supports_reasoning_summaries = false
# 改成你机器上的绝对路径：
model_catalog_json = "/ABS/PATH/to/xapi/configs/codex_model_catalog.xapi.example.json"

[model_providers.xapi]
name = "xapi"
base_url = "http://127.0.0.1:8080/v1"
env_key = "OPENAI_API_KEY"
wire_api = "responses"
supports_websockets = false
```

```bash
# 推荐：脚本已处理 NO_PROXY + ephemeral metadata
./scripts/codex_via_xapi.sh

# 或手动：若本机开了 HTTP 代理，务必绕过 loopback
export NO_PROXY=127.0.0.1,localhost,::1
export no_proxy="$NO_PROXY"
unset HTTP_PROXY HTTPS_PROXY ALL_PROXY http_proxy https_proxy all_proxy
unset OPENAI_BASE_URL
export OPENAI_API_KEY=sk-xapi-admin

curl http://127.0.0.1:8080/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","input":"hi","max_output_tokens":64}'
```

说明：
- 上游为 OpenAI 兼容渠道时，网关自动做 Messages/Responses ↔ Chat Completions 往返转换（含流式事件、tools、**image**、**document/input_file**）。
- Codex 发往 OpenAI 兼容上游时会剥离 `reasoning` 等 Chat Completions 不认的字段；发往 Claude 时仅在显式 `reasoning.effort` 时开启 extended thinking。
- `claude-opus*` / `claude-sonnet*` / `claude-haiku*` / `claude-*` 可通过模型别名映射到实际对外模型（控制台「别名」可改）。
- `anthropic-version` / `anthropic-beta`：发往 Claude 上游时原样转发（MCP beta 依赖此头）。
- **MCP**：Claude→Claude 透传 `mcp_servers` / `container`；若客户端 Messages 打到 OpenAI 兼容上游则返回 400（无法等价转换）。
- `count_tokens`：上游为 Claude 渠道时优先透传 `/v1/messages/count_tokens`；否则按 messages/system/tools 文本本地估算（image≈1600、document≈4000 tokens/块）。

### 分档 Token 计费

结算从上游 usage 解析结构化明细，并按分档单价计费：

| 字段 | 含义 | 售价倍率（ModelPrice） | 成本（ChannelCost） |
|------|------|------------------------|---------------------|
| input | 非缓存输入 | `model_ratio` | `input_per_1k` |
| cached_input | 读缓存 | `cache_ratio`（默认 0.1） | `cached_input_per_1k`（0=回退 input） |
| cache_write | 建缓存（Anthropic） | `cache_write_ratio`（默认 1.25） | `cache_write_per_1k`（0=回退 input） |
| output | 输出（含 reasoning） | `completion_ratio` | `output_per_1k` |
| reasoning | 推理子集 | 仅记录，不另加价 | 同上 |

OpenAI/Ark 读 `prompt_tokens_details.cached_tokens` / `completion_tokens_details.reasoning_tokens`；Anthropic 读 `cache_read_input_tokens` / `cache_creation_input_tokens`。UsageLog 保留 `prompt_tokens`/`completion_tokens` 总量，并新增 `cached_tokens` / `cache_write_tokens` / `reasoning_tokens`。

## 协议适配器（可扩展）

不同厂商/客户端协议以**静态注册的 Adapter** 接入（非动态插件），分两层：

| 层 | 接口 | 已注册 |
|----|------|--------|
| 客户端 | `ClientProtocolAdapter` | `openai_chat`、`openai_responses`、`openai_embeddings`、`openai_audio`、`openai_moderations`、`openai_rerank`、`openai_realtime`、`gemini_generate`、`anthropic_messages`、`anthropic_hello`、`models`、`ark_v3_chat`、`ark_v3_media` |
| 上游 | `UpstreamProtocolAdapter` | `openai_compat`、`claude`（含 `anthropic`）、`gemini`（含 `google`/`vertex`/`vertex_ai`）、`ark`（`/api/v3/chat/completions`） |
| 视频任务 | `TaskAdaptor` | `doubao_video`（`volcengine`/`ark`/`volcengine_ark`）、`hailuo`/`minimax`、`kling`、`midjourney_*`、`suno`、`sora`/`openai_video`、`jimeng`、`vidu`、`gemini`/`google`/`vertex`（Veo）、`mock` |

聊天上游 `type=volcengine|ark` 走 Ark chat；同 type 的视频走 Seedance TaskAdaptor。新增厂商：实现 adapter → 在 `register_builtin_protocol_adapters()` / `TaskService` 注册即可。

### 火山方舟 Python SDK

```bash
pip install 'volcengine-python-sdk[ark]'
```

```python
from volcenginesdkarkruntime import Ark

client = Ark(
    api_key="sk-xapi-admin",
    base_url="http://127.0.0.1:8080/api/v3",
)
completion = client.chat.completions.create(
    model="gpt-4o-mini",  # 或渠道里的 doubao / ep-xxx
    messages=[{"role": "user", "content": "hi"}],
)
print(completion.choices[0].message.content)
```

网关已接受 `POST /api/v3/chat/completions`（及 bots 路径），与 OpenAI Chat 同一套计费/路由；上游渠道 `type=volcengine` 时出站也走 `/api/v3/chat/completions`。

## 控制台（Vite + React）

源码：`web/console-app/`（Vite + React + TS）。构建产物输出到 `web/console/dist/`，由 `xapi-control` **优先**托管。旧静态台在 `web/console/legacy/`（仅无 dist 时兜底，**不是**免构建主路径）。

```bash
cd web/console-app && npm install && npm run build
# 开发联调：npm run dev  （代理 /api → :8081；另开 xapi-control）
```

浏览器打开 `http://127.0.0.1:8081/`（或 Vite `http://127.0.0.1:5173/`）。演示账号 `demo` / `demo123456`（root），或管理员密钥 `sk-xapi-admin`。

| 能力 | 说明 |
|------|------|
| 中英文 | 侧栏切换，文案分语言 |
| 账户 / TOTP | 「账户」页：`/api/user/2fa/setup|enable|disable`；登录挑战见上文 |
| 渠道 | 拉取上游模型、勾选上线；类型含 openai / claude / volcengine / hailuo / kling / midjourney_* / suno / gemini / vertex 等 |
| 渠道测试 | `POST /api/channels/{id}/test`（连通 / 廉价请求） |
| 卖价 / 成本 | 下拉用 `/api/models/catalog` 统一模型名 |
| 系统设置 | `GET/PUT /api/settings`（root；含注册开关、默认额度、账本汇率） |
| 额度流水 | `GET /api/quota-ledger`；管理端调账 `POST /api/users/{id}/quota`（非 Stripe UI） |
| 角色 | user=1 / admin=10 / root=100；创建/提权 admin 需 root |

**gateway / control / worker 请共用同一 SQL 账本**：

- SQLite：`data/xapi.sqlite`（`configs/xapi.local.yaml`）
- MySQL：`mysql_dsn`（`configs/xapi.mysql.example.yaml`）
- Postgres：`postgres_dsn`（`configs/xapi.docker.yaml`）
- 可选灾备导出：`data/config-mirror.json`（不是运行时账本；勿再使用 `store.json`）

```bash
# 仓库根目录启动，保证能读到 ./web/console/dist 与 ./data
./build/xapi-control -c configs/xapi.local.yaml &
./build/xapi-gateway -c configs/xapi.local.yaml &
open http://127.0.0.1:8081/
```

验收：控制台加代理渠道后约 2s 内 gateway 可选路；打几笔 mock chat 后再改卖价，额度不回滚。Docker 用 [`configs/xapi.docker.yaml`](configs/xapi.docker.yaml)。冒烟：`./scripts/console-smoke.sh`。

## 用户自助（对齐 new-api）

控制面端口默认 `8081`。客户可注册/登录后自助创建 API Key（JWT 管控制台，`sk-` 调 gateway）：

```bash
# 注册
curl -X POST http://127.0.0.1:8081/api/user/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"alice","password":"alice1234"}'

# 登录 → data.access_token（若启用 2FA 则先返回 requires_2fa + pending_token）
curl -X POST http://127.0.0.1:8081/api/user/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"alice","password":"alice1234"}'

# 创建 Key（不回证明文）
curl -X POST http://127.0.0.1:8081/api/token/ \
  -H "Authorization: Bearer <access_token>" \
  -H 'Content-Type: application/json' \
  -d '{"name":"default","remain_quota":100000,"expired_time":-1}'

# 列表后取明文
curl http://127.0.0.1:8081/api/token/?p=1&size=10 \
  -H "Authorization: Bearer <access_token>"
curl -X POST http://127.0.0.1:8081/api/token/<id>/key \
  -H "Authorization: Bearer <access_token>"

# 用 sk- 调网关
curl http://127.0.0.1:8080/v1/chat/completions \
  -H "Authorization: Bearer sk-..." \
  -H 'Content-Type: application/json' \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"hi"}]}'
```

Demo 账号：`demo` / `demo123456`（role=root）。REST 路径与 new-api 对齐，便于后续换更完整前端。

## 运营：代理渠道 / 卖价 / 成本 / 故障切换

客户只看到对外模型名与卖价；供应商渠道（直连或 OpenAI 兼容代理）在控制面配置，对客户无感。

### 挂 Partner 代理（OpenAI 兼容）

`type` 可用：`openai` / `openai_compatible` / `proxy` / `openrouter`（任意 `base_url` + Key + models 即可进池）；`azure`（deployment + `api-key`）；`bedrock`/`aws`（Claude invoke）；Gemini 用 `gemini`/`google`，Vertex 用 `vertex`/`vertex_ai`（Bearer 或 GCP SA JSON）。`model_mapping` 把对外名映射成上游名，**计费仍用对外名**。

```bash
# 1) 渠道
curl -X POST http://127.0.0.1:8081/api/channels \
  -H "Authorization: Bearer sk-xapi-admin" \
  -H 'Content-Type: application/json' \
  -d '{
    "name":"partner-proxy",
    "type":"openai_compatible",
    "base_url":"https://partner.example/v1",
    "api_key":"<PARTNER_KEY>",
    "models":["gpt-4o-mini"],
    "model_mapping":{"gpt-4o-mini":"partner/gpt-4o-mini-fast"},
    "priority":80,
    "weight":1,
    "auto_ban":true
  }'

# 2) Ability（对外模型 → 渠道）
curl -X POST http://127.0.0.1:8081/api/abilities \
  -H "Authorization: Bearer sk-xapi-admin" \
  -H 'Content-Type: application/json' \
  -d '{"group":"default","model":"gpt-4o-mini","channel_id":<id>,"priority":80,"weight":1}'
```

同模型可挂多条渠道（直连 + 代理）。上游返回 402/403/429/5xx 或余额类错误体时会 exclude 并重试；`auto_ban` 在硬失败（额度耗尽等）或连续软失败后将渠道置为 `auto_disabled`。恢复：

```bash
curl -X POST http://127.0.0.1:8081/api/channels/<id>/enable \
  -H "Authorization: Bearer sk-xapi-admin"
```

### 卖价 vs 渠道成本

| 概念 | API | 含义 |
|------|-----|------|
| 卖价 `ModelPrice` | `GET/POST /api/prices` | 客户扣额度（`model_ratio` / `completion_ratio` / `per_call_quota`） |
| 成本 `ChannelCost` | `GET/POST /api/channel-costs` | 进货价（元/1K in·out、元/张、元/秒、或 `per_call`） |

每次请求的 `UsageLog` 同时记 `sell_quota` 与 `cost_milli`（金额×1000），可用 `sell_quota - cost_milli` 粗算毛利（单位需自行对齐）。

```bash
curl -X POST http://127.0.0.1:8081/api/prices \
  -H "Authorization: Bearer sk-xapi-admin" \
  -H 'Content-Type: application/json' \
  -d '{"model":"gpt-4o-mini","model_ratio":1.0,"completion_ratio":2.0}'

curl -X POST http://127.0.0.1:8081/api/channel-costs \
  -H "Authorization: Bearer sk-xapi-admin" \
  -H 'Content-Type: application/json' \
  -d '{"channel_id":1,"upstream_model":"*","input_per_1k":0.002,"output_per_1k":0.008}'

curl http://127.0.0.1:8081/api/logs -H "Authorization: Bearer sk-xapi-admin"
```

更多字段说明见 `configs/xapi.example.yaml` 末尾注释。

## 测试与基准

```bash
make test
make bench
```

门槛（mock 热连接）：overhead P50 ≤ 1ms，P99 ≤ 3ms；flush P99 ≤ 2ms。

### 本地 live 冒烟（只读配置）

前提：Postgres + `configs/xapi.local.yaml` 三进程已起（`gateway:8080` / `control:8081`）。脚本**不会**改渠道/密钥/价格。

| 用例 | 默认 | 说明 |
|------|------|------|
| OpenAI chat `/v1/chat/completions` | ✓ | `gpt-4o-mini` |
| OpenAI Responses `/v1/responses` | ✓ | 含 Codex 字段 `reasoning.summary` |
| Claude `/v1/messages` | ✓ | 含 `context_management` 剥离 |
| Embeddings `/v1/embeddings` | ✓ | `text-embedding-3-small` |
| Codex CLI | `--with-codex` / `--full` | 临时 `CODEX_HOME` + model metadata，强制 `NO_PROXY` |
| Claude Code CLI | `--with-claude-cli` / `--full` | `--bare`，走 `/anthropic` |
| Ark Seedream | `--with-image` / `--full` | `POST /api/v3/images/generations`（OpenAI 渠道则出站 `/v1/images/generations`） |
| Ark Seedance | `--with-video` / `--full` | `scripts/seedance_test.py` |
| Hailuo 取消 | 单测 / 手工 | `DELETE /v1/videos/{id}` 或 Ark `DELETE .../tasks/{id}`（含退款） |

```bash
# 协议层（日常）
./scripts/smoke_live.sh

# 协议 + 两个 CLI（含 Codex metadata 校验）
./scripts/smoke_live.sh --with-codex --with-claude-cli

# 媒体（按需；有成本）
./scripts/smoke_live.sh --with-image
./scripts/smoke_live.sh --with-video

# 产品化：ledger 里非 mock:// 渠道经网关打真上游
./scripts/smoke_live.sh --with-live-channels

# 全量（CLI + Seedream + Seedance + live-channels）
./scripts/smoke_live.sh --full

# 单独跑 CLI（同样绕过系统代理）
./scripts/codex_via_xapi.sh
XAPI_CODEX_MODEL=claude-haiku-4-5-20251001 ./scripts/codex_via_xapi.sh 'Reply with exactly: HAIKU_OK'
./scripts/claude_via_xapi.sh
```

结果可落盘：`./scripts/smoke_live.sh --with-codex --json data/smoke-last.json`（只写 JSON，不碰 DB）。

## 运维要点

- **Metrics**：业务口仍有 `GET /metrics`；另按 `metrics_listen`（默认 `0.0.0.0:9090`）起独立监听，仅暴露 `/metrics` 与 `/healthz`。
- **控制面多实例**：登录 refresh session 存 Store（`dashboard_sessions`）；多 `xapi-control` 需共享 Postgres（或同机共享同一 `xapi.sqlite`）。渠道快照：`PUBLISH/SUBSCRIBE xapi:config:route` 即时刷新，并用 `snapshot_reload_ms` + `config_versions.route` 轮询兜底。`redis_url` 支持 `redis://[:password@]host:port[/db]`。
- **官价 sync**：`POST /api/channels/{id}/sync-quotes` 使用 builtin ∪ settings.`official_prices`（含 Seedream / Hailuo 行）；`hailuo`/`minimax` 渠道走共享官价目录。

## Docker / MySQL / Postgres（可选）

本地开发默认 [`configs/xapi.local.yaml`](configs/xapi.local.yaml) 为 **sqlite**，无需 Docker。

```bash
# MySQL
# macOS: brew install mysql-client
# Ubuntu: sudo apt install libmysqlclient-dev
docker compose --profile mysql up -d mysql
cmake -S . -B build -DXAPI_WITH_MYSQL=ON -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
./build/xapi-gateway -c configs/xapi.mysql.example.yaml
# 单测：XAPI_TEST_MYSQL_DSN='mysql://xapi:xapi@127.0.0.1:3306/xapi' ./build/xapi_tests

# Postgres
# macOS: brew install libpq && export CMAKE_PREFIX_PATH="$(brew --prefix libpq):$CMAKE_PREFIX_PATH"
docker compose up -d postgres
cmake -S . -B build -DXAPI_WITH_POSTGRES=ON -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
# 单测：XAPI_TEST_PG_DSN='postgresql://xapi:xapi@127.0.0.1:5432/xapi' ./build/xapi_tests
```

`configs/xapi.docker.yaml` 仍为 Postgres 三进程编排；MySQL 示例见 `configs/xapi.mysql.example.yaml`。

```bash
# Ubuntu（默认）
docker build -t xapi:ubuntu -f Dockerfile.ubuntu .
# CentOS 兼容（Rocky Linux 9）
docker build -t xapi:centos -f Dockerfile.centos .

docker compose up --build
```

可选静态链 libstdc++（便于拷贝到同 glibc 系列机器）：

```bash
cmake -S . -B build -DXAPI_STATIC_LINK=ON -DCMAKE_BUILD_TYPE=Release
```
# WiseRoute · Wisro (微路由)

**A self-hosted AI gateway: one address for every model provider, a key for every person, and an exact record of what each one spends.**

[中文](README.md) · [Documentation](https://route.ifootoo.com/docs/en/) · [Website](https://route.ifootoo.com/) · [Docker Hub](https://hub.docker.com/r/ifootoo/wiseroute)

> This repository publishes downloads, the feature overview and install instructions. Wisro ships as container images; the source code is not public.

## What it does

- **One endpoint, two APIs.** OpenAI (Chat Completions, Responses) and Anthropic Messages, with conversion between them, so Codex, Claude Code and the OpenAI SDKs work with a base-URL change. Also Embeddings, images, audio, Realtime, Rerank, the native Gemini API and an MCP gateway.
- **Many upstreams.** OpenAI, Anthropic, AWS Bedrock, Vertex AI, Azure OpenAI and any OpenAI- or Claude-shaped provider (36 presets in the console). Priority and weight routing, failover between channels with cool-down recovery, fallback to another model.
- **Spend under control.** Departments with inherited budgets, per-person limits, request tags with tag budgets, rate limits (RPM / TPM / concurrency), approvals. Every request is reserved and then settled against a ledger. Prices in any currency, with daily exchange rates.
- **Security and compliance.** SSO (Feishu, WeCom, DingTalk, OIDC), outbound redaction of secrets and personal data, prompt-injection guardrails, streaming answer checks, per-team MCP tool lists, an audit log of every admin change.
- **Request log.** Request, answer, attachments and cost of every call; usage, cost and margin day by day.
- **Runs anywhere.** One image for every role; SQLite on a single box, PostgreSQL or MySQL for more. Docker Compose files and a Helm chart ship inside the image. `linux/amd64` and `linux/arm64`.

## Two editions, one image

| | Enterprise | Reseller |
|---|---|---|
| For | Your own employees | Buying upstream access and selling it on |
| Money | Department budgets, personal limits, approvals | Customer sign-up, top-ups, redemption codes |
| Reports | Usage and spend per department | Revenue, upstream cost, margin |

Requests behave identically in both. See [Editions](https://route.ifootoo.com/docs/en/editions).

## Install

A Linux server (x86_64 or ARM64; 2 cores, 4 GB RAM and 5 GB free disk or more) with ports **8080** (model API) and **8081** (console) open.

### One command (recommended)

```bash
curl -fsSL https://route.ifootoo.com/install.sh | bash
```

The script picks Docker Hub or the Aliyun mirror (whichever is reachable and faster), installs Docker if it is missing, writes a `.env` with random secrets, starts the services and prints the console address and setup code.

Common options:

```bash
curl -fsSL https://route.ifootoo.com/install.sh | bash -s -- --yes                 # no questions, defaults
curl -fsSL https://route.ifootoo.com/install.sh | bash -s -- --edition reseller    # reseller edition
curl -fsSL https://route.ifootoo.com/install.sh | bash -s -- --registry hub        # pull from Docker Hub
curl -fsSL https://route.ifootoo.com/install.sh | bash -s -- --help                # every option
```

### Single container (SQLite, quickest to try)

```bash
docker run -d --name wisro -p 8080:8080 -p 8081:8081 -v wisro-data:/data \
  ifootoo/wiseroute:0.0.3
docker logs wisro | grep "setup code"
```

Open `http://<host>:8081`, claim the install with the setup code, add a channel and a key. Clients use `http://<host>:8080/v1` (OpenAI) or `http://<host>:8080/anthropic` (Claude).

### Production (PostgreSQL; separate gateway, console and worker)

```bash
docker run --rm ifootoo/wiseroute:0.0.3 deploy-bundle | tar xz
cd wisro-deploy-0.0.3
./scripts/init.sh
docker compose up -d
```

Domains, HTTPS, reverse proxies (nginx, OpenResty, 1Panel, BT Panel), backups and upgrades: [full install guide](https://route.ifootoo.com/docs/en/install).

## Downloads

| Source | Image |
|---|---|
| Docker Hub | `ifootoo/wiseroute` |
| Aliyun (mainland China) | `crpi-qecnvpqw46ro2why.cn-shanghai.personal.cr.aliyuncs.com/footoo/wiseroute` |
| Install script | `https://route.ifootoo.com/install.sh` |

| Tag | Notes |
|---|---|
| `0.0.3`, `latest` | Third preview; one-command install |
| `0.0.2` | Second preview |
| `0.0.1` | First public preview |

Ports: `8080` model API, `8081` console, admin API and documentation, `9090` Prometheus metrics.

## Documentation

- [Install](https://route.ifootoo.com/docs/en/install) · [Quick start](https://route.ifootoo.com/docs/en/quickstart) · [Deployment](https://route.ifootoo.com/docs/en/deployment)
- [API](https://route.ifootoo.com/docs/en/api) · [Routing](https://route.ifootoo.com/docs/en/routing) · [Codex / Claude passthrough](https://route.ifootoo.com/docs/en/passthrough)
- [Reseller guide](https://route.ifootoo.com/docs/en/reseller) · [Terms](https://route.ifootoo.com/docs/terms) · [Privacy](https://route.ifootoo.com/docs/privacy)

Every image also serves the full documentation at `http://<host>:8081/docs/`.

## Feedback

Open an [issue](https://github.com/lookou/wiseroute/issues) or write to biz@ifootoo.com.

## Licence

Proprietary. Free to run on your own infrastructure as a single-tenant deployment for your own use. Hosting it for other customers or reselling model access needs a commercial licence: biz@ifootoo.com. Binaries and images only; no source code is distributed.

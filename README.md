# Wisro · 微路由 (WiseRoute)

**私有化 AI 网关：一个地址接入所有模型，每人一把密钥，每笔花费记清楚。**

[English](README.en.md) · [在线文档](https://route.ifootoo.com/docs/) · [官网](https://route.ifootoo.com/) · [Docker Hub](https://hub.docker.com/r/ifootoo/wiseroute)

> 本仓库只发布下载方式、功能说明和安装方法。Wisro 以镜像形式分发，源代码不公开。

## 能做什么

- **一个地址，两种接口**：OpenAI（Chat Completions、Responses）和 Anthropic Messages，互相转换。Codex、Claude Code、OpenAI SDK 只需改接口地址。另有 Embeddings、图片、语音、Realtime、Rerank、Gemini 原生接口和 MCP 网关。
- **接多家上游**：OpenAI、Anthropic、AWS Bedrock、Vertex AI、Azure OpenAI，以及任何 OpenAI / Claude 格式的服务商（控制台内置 36 个预设）。按优先级与权重选路，渠道故障自动切换并按冷却恢复，可兜底到其他模型。
- **花费可控**：部门预算逐级继承、个人额度、请求标签与标签预算、限流（RPM / TPM / 并发）、审批。每次请求先预扣、再按实际用量结算到账本。任意货币计价，汇率每日自动更新。
- **安全合规**：单点登录（飞书、企业微信、钉钉、OIDC），发出前对密钥和个人信息脱敏，提示注入护栏，流式回答检查，MCP 工具按团队授权，管理操作全量审计。
- **调用记录**：每次调用的请求、回答、附件和费用都可查，按天看用量、成本和毛利。
- **哪里都能跑**：一个镜像承担所有角色；单机用 SQLite，多机用 PostgreSQL 或 MySQL。镜像内附 Docker Compose 文件和 Helm chart。支持 `linux/amd64` 和 `linux/arm64`。

## 两个版本，同一个镜像

| | 企业版（Enterprise） | 运营版（Reseller，转售） |
|---|---|---|
| 给谁用 | 公司自己的员工 | 采购上游、加价卖给客户 |
| 钱怎么管 | 部门预算、个人额度、审批 | 客户自助注册、充值、兑换码 |
| 看什么 | 各部门用量与花费 | 收入、上游成本、毛利 |

请求本身在两个版本里完全一样。详见[两个版本的区别](https://route.ifootoo.com/docs/editions)。

## 安装

准备一台 Linux 服务器（x86_64 或 ARM64，建议 2 核 4 GB 以上、5 GB 空闲磁盘），放行 **8080**（模型接口）和 **8081**（控制台）端口。

### 一条命令安装（推荐）

```bash
curl -fsSL https://route.ifootoo.com/install.sh | bash
```

脚本会自动选择 Docker Hub 或阿里云镜像（国内无需配置加速）、没有 Docker 时帮你装、生成带随机密钥的 `.env`、启动服务，最后打印控制台地址和设置码。

常用选项：

```bash
curl -fsSL https://route.ifootoo.com/install.sh | bash -s -- --yes                 # 不提问，全用默认值
curl -fsSL https://route.ifootoo.com/install.sh | bash -s -- --edition reseller    # 运营版
curl -fsSL https://route.ifootoo.com/install.sh | bash -s -- --registry aliyun     # 从阿里云拉取
curl -fsSL https://route.ifootoo.com/install.sh | bash -s -- --help                # 全部选项
```

### Docker 单容器（SQLite，最快试用）

```bash
docker run -d --name wisro -p 8080:8080 -p 8081:8081 -v wisro-data:/data \
  ifootoo/wiseroute:0.0.3
docker logs wisro | grep "setup code"
```

打开 `http://<服务器>:8081`，填入设置码认领，添加上游渠道和密钥。客户端接口地址：`http://<服务器>:8080/v1`（OpenAI 格式）或 `http://<服务器>:8080/anthropic`（Claude 格式）。

### 生产部署（PostgreSQL，网关 / 控制台 / 后台任务分开）

```bash
docker run --rm ifootoo/wiseroute:0.0.3 deploy-bundle | tar xz
cd wisro-deploy-0.0.3
./scripts/init.sh
docker compose up -d
```

域名、HTTPS、反向代理（nginx、OpenResty、1Panel、宝塔）、备份和升级见[完整部署文档](https://route.ifootoo.com/docs/install)。

## 下载

| 来源 | 镜像 |
|---|---|
| Docker Hub | `ifootoo/wiseroute` |
| 阿里云（国内） | `crpi-qecnvpqw46ro2why.cn-shanghai.personal.cr.aliyuncs.com/footoo/wiseroute` |
| 安装脚本 | `https://route.ifootoo.com/install.sh` |

| 版本 | 说明 |
|---|---|
| `0.0.3`, `latest` | 第三个预览版，一条命令安装 |
| `0.0.2` | 第二个预览版 |
| `0.0.1` | 首个公开预览版 |

端口：`8080` 模型接口，`8081` 控制台、管理接口和文档，`9090` Prometheus 指标。

## 文档

- [国内快速安装](https://route.ifootoo.com/docs/install_cn) · [完整部署](https://route.ifootoo.com/docs/install) · [快速上手](https://route.ifootoo.com/docs/quickstart)
- [接口说明](https://route.ifootoo.com/docs/api) · [选路与切换](https://route.ifootoo.com/docs/routing) · [Codex / Claude 直连](https://route.ifootoo.com/docs/passthrough)
- [运营版指南](https://route.ifootoo.com/docs/reseller) · [服务条款](https://route.ifootoo.com/docs/terms) · [隐私](https://route.ifootoo.com/docs/privacy)

镜像自带全部文档：`http://<服务器>:8081/docs/`。

## 问题反馈

在本仓库提 [Issue](https://github.com/lookou/wiseroute/issues)，或发邮件到 biz@ifootoo.com。

## 许可

专有软件。可在自己的基础设施上免费运行，供自己使用（单租户部署）。为其他客户托管或转售模型访问需要商业授权：biz@ifootoo.com。只分发二进制和镜像，不分发源代码。

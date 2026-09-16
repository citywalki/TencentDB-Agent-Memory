# Docker Hub 镜像发布

把三件套镜像构建并推送到 Docker Hub 的
[`agentmemory`](https://hub.docker.com/u/agentmemory) namespace。

`publish.sh` 是自包含的：只依赖各组件自己的 Dockerfile、
`deploy/panel-knowledge-combined/build.sh` 和 `MemoryPanel/scripts/secret-scan.sh`。

## 组件与镜像名

| 组件 | 构建上下文 | 镜像 |
|---|---|---|
| `memory-core` | `MemoryCore/` | `agentmemory/memory-core` |
| `memory-proxy` | `MemoryProxy/`（rsync 到临时 context） | `agentmemory/memory-proxy` |
| `memory-hub` | `MemoryPanel/` + `MemoryKnowledge/` 合并 | `agentmemory/memory-hub` |

## 前置

```bash
docker login docker.io          # 账号需有 agentmemory 推送权限
docker buildx version           # 需要 buildx（脚本会自动创建 builder）
```

## 使用

```bash
cd deploy/dockerhub

# 三件套一次发布
VERSION=1.0.0 ./publish.sh all

# 单个组件
VERSION=1.0.0 ./publish.sh memory-core
VERSION=1.0.0 ./publish.sh memory-proxy
VERSION=1.0.0 ./publish.sh memory-hub

# 干跑：只做 secret-scan 和 context 准备，不构建不推送
DRY_RUN=1 VERSION=1.0.0 ./publish.sh all

# 本地单架构构建并抽查镜像内容，不推送
PUSH=0 VERSION=1.0.0 ./publish.sh memory-core

# 同时更新 :latest
ALSO_LATEST=1 VERSION=1.0.0 ./publish.sh all
```

`VERSION` 必填，且不接受 `dev-` 开头的值，避免把开发 tag 推上公网。

## 环境变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `VERSION` | 无（必填） | 镜像 tag |
| `NAMESPACE` | `agentmemory` | Docker Hub namespace |
| `REGISTRY` | `docker.io` | 目标 registry |
| `PLATFORMS` | `linux/amd64,linux/arm64` | 多架构构建目标 |
| `ALSO_LATEST` | `0` | 是否同时推 `:latest` |
| `PUSH` | `1` | 置 `0` 则本地 `--load` 单架构，不推送 |
| `DRY_RUN` | `0` | 置 `1` 只跑扫描与 context 准备 |
| `LOAD_PLATFORM` | `linux/amd64` | `PUSH=0` 时本地构建的架构 |
| `KEEP_CTX` | `0` | 置 `1` 复用上次的临时 context |
| `APT_MIRROR` | `deb.debian.org` | 构建期 apt 源，内网可设为加速镜像 |

## 构建期 apt 加速

四个 Dockerfile 都通过 `APT_MIRROR` build-arg 控制 apt 源，默认走 Debian 官方，
公网环境开箱可用。内网构建想加速时统一传一个变量即可，镜像产物本身不受影响：

```bash
APT_MIRROR=<your-debian-mirror> VERSION=1.0.0 ./publish.sh all
```

## 关于可选私有模块

- `MemoryProxy/packages/cost-guard` 是可选扩展，不进公开镜像。`publish.sh` 会在
  临时 context 里生成一个 stub 包让依赖图能解析；运行时 `src/guard-adapter.ts`
  的动态 import 失败后自动降级为直通转发。
- `MemoryCore/src/integrations` 同理，已在 `MemoryCore/.dockerignore` 中排除，
  运行时走 fallback。


## GitHub Actions 发布到 GitHub Packages（GHCR）

[`.github/workflows/publish-ghcr.yml`](../../.github/workflows/publish-ghcr.yml)
复用同一个 `publish.sh`（`REGISTRY=ghcr.io`），把三件套发布到本仓库的
Packages，认证走内置 `GITHUB_TOKEN`，无需额外 Secret：

| 镜像 | 地址 |
|---|---|
| memory-core | `ghcr.io/<owner>/memory-core` |
| memory-proxy | `ghcr.io/<owner>/memory-proxy` |
| memory-hub | `ghcr.io/<owner>/memory-hub` |

触发方式与版本策略（版本号遵循官方上游
[TencentCloud/TencentDB-Agent-Memory](https://github.com/TencentCloud/TencentDB-Agent-Memory)
的 tag 命名，`vX.Y.Z` / `vX.Y.Z-beta.N`）：

- **推送 `v*` tag**（如 `git tag v2.0.2 && git push origin v2.0.2`）：
  发布全部三件套，tag 去 `v` 前缀作镜像版本号
- **`:latest` 只随稳定版移动**：`vX.Y.Z` 更新 `:latest`；`-beta.N` 视为
  预发布（对齐上游 GitHub Release 的 prerelease 语义），不动 `:latest`
- **手动 dispatch**：可指定版本号、单个镜像、平台；默认不推 `:latest`

首次推送生成的 package 默认 private，可见性在仓库 Packages 设置中调整。

### 与上游保持 tag 同步

本仓库是 [TencentCloud/TencentDB-Agent-Memory](https://github.com/TencentCloud/TencentDB-Agent-Memory)
的 fork，已配置 `upstream` remote（`git remote add upstream <url>`，一次性），
上游全部历史 tag 已镜像到本仓库。上游发布新版本（如 `v2.0.3`）后：

```bash
git fetch upstream --tags             # 拉取上游新代码与 tag
git merge upstream/feat/server_team   # 同步代码（上游默认分支即 feat/server_team）
git tag -f v2.0.3                     # 在 fork 合并后的 HEAD 上重打同名 tag
git push origin feat/server_team v2.0.3
```

为什么要在 fork HEAD 上重打 tag：上游 tag 指向上游提交，该提交不含本仓库
的 publish workflow（GitHub 按 tag 指向的提交取 workflow 定义），直接镜像
推送不会触发构建；把同名 tag 落在包含 workflow 的合并提交上，push 即自动
发布 `2.0.3` 镜像。已镜像的历史 tag 同理不会触发构建。

也可以不动 tag，直接在 Actions 页面运行 `Publish Docker Images (GHCR)`，
`version` 填 `2.0.3`，效果相同。

## 验证

```bash
docker pull agentmemory/memory-core:1.0.0
docker buildx imagetools inspect agentmemory/memory-core:1.0.0
```

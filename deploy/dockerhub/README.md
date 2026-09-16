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

- **Sync Upstream Release（推荐入口）**：选择上游 tag → 自动同步代码 →
  同一 run 内直接发布（见下节）
- **推送 `v*` tag**（如 `git tag v2.0.2 && git push origin v2.0.2`）：
  发布全部三件套，tag 去 `v` 前缀作镜像版本号
- **`:latest` 只随稳定版移动**：`vX.Y.Z` 更新 `:latest`；`-beta.N` 视为
  预发布（对齐上游 GitHub Release 的 prerelease 语义），不动 `:latest`
- **手动 dispatch**：可指定版本号、单个镜像、平台

首次推送生成的 package 默认 private，可见性在仓库 Packages 设置中调整。

### 同步上游发布（Action，推荐）

上游（[TencentCloud/TencentDB-Agent-Memory](https://github.com/TencentCloud/TencentDB-Agent-Memory)）
发新版本后，在 Actions 页面运行 **Sync Upstream Release**，`tag` 填上游
tag（如 `v2.0.3`）：

1. 拉取并校验上游 tag（`vX.Y.Z` / `vX.Y.Z-beta.N`）
2. `git merge` 到运行分支（fork 自己的 workflow 与文档全部保留；
   `CHANGELOG.md` 冲突自动按「fork 未发布章节 ⊕ 上游版本章节」拼接，
   其它文件冲突则中止并提示手工处理）
3. 推送同步结果，直接调用 `publish-ghcr.yml` 发布镜像 —— 版本号 = tag
   去 `v` 前缀，`:latest` 策略同上

本仓库镜像的上游历史 tag 与上游 SHA 保持一致；发布由本 Action 驱动，
不依赖 fork 上的 tag（GITHUB_TOKEN 的 push 不触发其它 workflow，故采用
workflow_call 直连发布）。

备选：本地手工同步（效果相同）：

```bash
git remote add upstream https://github.com/TencentCloud/TencentDB-Agent-Memory.git
git fetch upstream --tags && git merge v2.0.3 && git push origin HEAD
# 然后运行 Publish Docker Images (GHCR)，version 填 2.0.3
```

## 验证

```bash
docker pull agentmemory/memory-core:1.0.0
docker buildx imagetools inspect agentmemory/memory-core:1.0.0
```

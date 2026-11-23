# 饥荒管理平台（DMP）懒猫微服应用

把 [miracleEverywhere/dst-management-platform-api](https://github.com/miracleEverywhere/dst-management-platform-api) 打包成懒猫微服 LPK，并用 Gitea Actions 自动跟随上游镜像版本发布。

- 包名：`cloud.lazycat.app.dmp`
- 子域名：`dmp.<微服域名>`，例如 `https://dmp.lc03test.heiyu.space`
- 镜像：`ghcr.io/miracleeverywhere/dst-management-platform-api:v*`（上游只提供 linux/amd64）

## 目录

```text
package.yml                 LPK 元数据（包名、版本、多语言、权限）
lzc-manifest.yml            运行结构：HTTP 路由、UDP ingress、挂载
lzc-build.yml               构建配置
lzc-icon.png                应用图标（由上游 dmp.svg 转换）
.github/lazycat-action.yml  ca-x/lazycat-github-action 的版本跟踪配置
.gitea/workflows/build-lpk.yml  Gitea Actions 流水线
dist/                       构建产物（已 gitignore）
```

## 本地构建与安装

```bash
mkdir -p dist
lzc-cli project build -o dist/application.lpk
lzc-cli lpk lint dist/application.lpk
lzc-cli lpk install dist/application.lpk --apk n
```

也可以把 LPK 拷到微服上直接安装（无需开发者工具授权）：

```bash
scp dist/application.lpk <box>:/tmp/dmp.lpk
ssh <box> 'lpk-manager install /tmp/dmp.lpk'
```

## 运行结构与数据

数据与上游 `docker/docker-compose.yml` 的挂载一一对应，落在 `/lzcapp/var` 下，卸载应用不会删除：

| 容器路径 | 微服路径 | 内容 |
| --- | --- | --- |
| `/root/.klei` | `/lzcapp/var/klei` | Klei 集群配置与存档 |
| `/root/data` | `/lzcapp/var/data` | `dmp.db` 数据库 |
| `/root/dst` | `/lzcapp/var/dst` | SteamCMD 下载的饥荒服务端 |
| `/root/logs` | `/lzcapp/var/logs` | 平台日志 |
| `/root/dmp_files` | `/lzcapp/var/dmp_files` | LuaJIT、插件等运行时文件 |
| `/root/steamcmd` | `/lzcapp/var/steamcmd` | SteamCMD，持久化后不每次重下 |

### 网络端口

- HTTP：走懒猫路由，`/=http://dmp:80`，由微服做鉴权与 HTTPS。
- UDP：`application.ingress` 把外部 `11000-12000`、`21000-21100` 原样转发到容器内同名端口，对应 DMP 的世界端口（默认 11001 起）和主服务器端口（默认 21001 起）。

对外地址形如 `dmp.<微服域名>:11001`。注意两点：

1. **同一端口只能给一个应用/实例用**：端口号按应用子域名分发，若同一微服上还有别的应用声明了同一段 UDP 端口，会冲突。
2. **Klei 大厅联机**：DST 服务器向 Klei 注册的是它自己看到的公网出口 IP。微服在 NAT 后面时，想让公网玩家通过大厅加入，仍需要在路由器上把相同 UDP 端口映射到微服；否则只能局域网/直连（`dmp.<微服域名>:端口`）玩。

## 自动化（Gitea Actions）

`.gitea/workflows/build-lpk.yml`：

1. 每天 03:17（cron）或手动触发。
2. 用 `ca-x/lazycat-github-action@v1` 的 composite Action 执行 `operation: check`：
   - 从 GHCR 读取 `v*` 正式版本 tag；
   - 选出比当前 `package.yml` 新的版本，改写 `package.yml` 与 `lzc-manifest.yml` 的镜像 tag；
   - 顺带构建一次 LPK 做校验。
3. 有变更时提交回默认分支。
4. 执行 `operation: build` 产出 LPK，复制为 `dist/cloud.lazycat.app.dmp-v<version>.lpk`。
5. 用 `akkuman/gitea-release-action` 创建 `v<version>` Release 并上传 LPK 与 `.sha256`。
6. 若配置了微服 SSH Secrets，则用 `lzc-cli lpk install` 自动升级到目标微服。

版本发现规则在 `.github/lazycat-action.yml`：`channel: stable` + `tag_regex: '^v\d+\.\d+\.\d+$'` + `delivery.mode: direct`（清单直接引用 `ghcr.io`，不需要任何镜像仓库凭据），`target_arch: amd64`。

### 需要配置的 Secrets（都可选）

| Secret | 用途 |
| --- | --- |
| `LAZYCAT_BOX_HOST` | 目标微服的 SSH 地址，配置后才会自动安装 |
| `LAZYCAT_BOX_SSH_KEY` | 目标微服的部署私钥 |
| `LAZYCAT_BOX_USER` | SSH 用户，默认 `root` |
| `LAZYCAT_BOX_SSH_PORT` | SSH 端口 |

Release 只用 Gitea 自带的 `github.token`，不需要额外 token。

### 发布到商店的后续改动

`direct` 交付只适合自用/内测。要提交官方商店或喵喵商店，需要：

1. 把镜像复制到懒猫官方源（`lzc-cli appstore copy-image`），或把 `delivery.mode` 改成 `lazycat` 并给 Runner 配 `LZC_API_TOKEN`；
2. 官方商店要求 `lzc-cli lpk lint` 无 warning，且 `services.*.image` 以 `registry.lazycat.cloud` 开头；
3. 私有商店要求 LPK 下载地址是真实的 GitHub Release Asset，Gitea Release 地址会被拒绝 —— 这一步需要把发布仓放到 GitHub，或者在 GitHub 上再建一个发布仓。

## 在 GitHub 上跑

同一份 `package.yml` / `lzc-manifest.yml` / `.github/lazycat-action.yml` 可以直接用官方 reusable workflow：

```yaml
# .github/workflows/lazycat.yml
name: LazyCat LPK
on:
  schedule:
    - cron: "17 3 * * *"
  workflow_dispatch:
permissions:
  contents: write
  pull-requests: write
jobs:
  lazycat:
    uses: ca-x/lazycat-github-action/.github/workflows/lazycat.yml@v1
    with:
      operation: auto
      config: .github/lazycat-action.yml
      versioned-release-asset: true
```

## 许可

打包脚本遵循 MIT；应用本身来自上游 DMP，版权归 miracleEverywhere。

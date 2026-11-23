# 饥荒管理平台（DMP）懒猫微服应用

把 [miracleEverywhere/dst-management-platform-api](https://github.com/miracleEverywhere/dst-management-platform-api) 打包成懒猫微服 LPK，并用 GitHub Actions 自动跟随上游镜像版本发布。

- 包名：`cloud.lazycat.app.dmp`
- 仓库：<https://github.com/522247020/dst-management-platform-lzcapp>
- 本地开发目录：`~/Code/dst-management-platform-lzcapp`
- 子域名：`dmp.<微服域名>`，例如 `https://dmp.lc03test.heiyu.space`
- 镜像：`ghcr.io/miracleeverywhere/dst-management-platform-api:v*`（上游只提供 linux/amd64）

## 目录

```text
package.yml                 LPK 元数据（包名、版本、多语言、权限）
lzc-manifest.yml            运行结构：HTTP 路由、UDP ingress、挂载
lzc-build.yml               构建配置
lzc-icon.png                应用图标（由上游 dmp.svg 转换）
.github/lazycat-action.yml      ca-x/lazycat-github-action 的版本跟踪配置
.github/workflows/lazycat.yml   GitHub Actions 流水线（reusable workflow）
docs/gitea-actions/build-lpk.yml  Gitea 版 composite 流水线（参考，已不在默认分支运行）
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

## 自动化（GitHub Actions）

`.github/workflows/lazycat.yml` 调用 `ca-x/lazycat-github-action` 的 reusable workflow：

| 项 | 值 |
| --- | --- |
| 触发 | 每天 03:17（cron）、手动 dispatch |
| 版本源 | GHCR 上的 `v*` 正式 tag（`^v\d+\.\d+\.\d+$`） |
| 策略 | `publish`：改 `package.yml` + `lzc-manifest.yml` → 提交 → 打 tag → 建 Release |
| 产物 | `cloud.lazycat.app.dmp-v<version>.lpk`（`versioned-release-asset: true`） |
| 交付 | `direct`，清单直接引用 `ghcr.io`，不需要镜像仓库凭据 |
| 保留 | Release 最多保留 1000 条（`release.cleanup`） |

手动指定版本：

```bash
gh workflow run lazycat.yml -f version=3.2.2
```

配置细节都在 `.github/lazycat-action.yml`。当前 `stores` 全部关闭，所以不需要任何 Secret。

### 装到微服

流水线只发 Release。装机可以：

```bash
# 方式一：开发机有授权
gh release download v3.2.2 -R 522247020/dst-management-platform-lzcapp -p '*.lpk'
lzc-cli lpk install cloud.lazycat.app.dmp-v3.2.2.lpk --apk n

# 方式二：盒子上直接装（不需要开发者工具授权）
scp cloud.lazycat.app.dmp-v3.2.2.lpk <box>:/tmp/dmp.lpk
ssh <box> 'lpk-manager install /tmp/dmp.lpk'
```

要让流水线自动装盒，把 `docs/gitea-actions/build-lpk.yml` 里的“安装到微服”步骤搬到 GitHub workflow，并配置 `LAZYCAT_BOX_HOST` / `LAZYCAT_BOX_SSH_KEY`。

### 发布到商店

`direct` 交付只适合自用/内测。要提交官方商店或喵喵商店：

1. 把镜像复制到懒猫官方源（`lzc-cli appstore copy-image`），或把 `delivery.mode` 改成 `lazycat` 并为 Runner 配 `LZC_API_TOKEN`；
2. 官方商店要求 `lzc-cli lpk lint` 无 warning，且 `services.*.image` 以 `registry.lazycat.cloud` 开头；
3. 私有商店要求 LPK 下载地址是 GitHub Release Asset —— 现在就满足（`versioned-release-asset: true`），配 `APPSTORE_URL` / `APPSTORE_TOKEN` 即可。

### Gitea 版本

`docs/gitea-actions/build-lpk.yml` 是 Gitea Actions 的 composite 版（check → 提交 → build → `akkuman/gitea-release-action`，可选 `lzc-cli` 装盒），放在 `docs/` 下不会被误触发。经验值：

- Gitea 上只能用 composite Action，reusable workflow 是 GitHub 专属；
- Action 启动要从 GitHub Release 下 6 MB 二进制，慢网会 60s 超时，可用仓库变量 `LAZYCAT_ACTION_RELEASE_BASE_URL` 指到内网镜像；
- Gitea 发给 Action 的 `GITHUB_TOKEN` 是 Gitea token，查上游 release notes 会 401 并回退模板；
- Gitea 的 Release 地址不是 GitHub 地址，私有商店会拒绝。

## 许可

打包脚本遵循 MIT；应用本身来自上游 DMP，版权归 miracleEverywhere。

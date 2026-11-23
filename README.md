# 饥荒管理平台（DMP）懒猫微服应用

把 [miracleEverywhere/dst-management-platform-api](https://github.com/miracleEverywhere/dst-management-platform-api) 打包成懒猫微服 LPK，并用 GitHub Actions 自动跟随上游镜像版本发布。

- 包名：`cloud.lazycat.app.dmp3`（`cloud.lazycat.app.dmp` 已被别人占用）
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
content/lazycat-injects/    网盘文件选择器注入脚本（打包进 /lzcapp/pkg/content）
.github/screenshots/        商店截图
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

## 微服集成（inject，无 OIDC）

### 免密登录

三阶段 inject，账号密码只在业务成功时落盘，登录页自动填充并提交：

| 阶段 | inject | 行为 |
| --- | --- | --- |
| request | `dmp-capture-credentials` | 抓 `POST /v3/user/register`、`POST /v3/user/login`、`PUT /v3/user/myself` 里的 `username` / `password` |
| response | `dmp-commit-credentials` | 仅当 HTTP 2xx 且响应体 `code == 200` 时写 `ctx.persist`（按微服用户隔离） |
| browser | `dmp-login-autofill` | `#/login*` 命中后填充 `input.v-field__input[type=text]` 与 `input[autocomplete=password]`，`autoSubmit` 自动登录 |

首次注册管理员、之后改密都会自动更新保存的凭据；没接 OIDC，也不改上游代码。

### 文件拦截（网盘打开/保存）

| 阶段 | inject | 行为 |
| --- | --- | --- |
| request | `lazycat-file-bridge` | `/__lazycat_file_bridge/*` 同源桥接到 `https://file.<微服域名>/files/home`，保留 Cookie 与 `Owner`，只放行 GET/HEAD/PUT |
| browser | `lazycat-file-chooser` | 接管 `<input type=file>`、`showOpenFilePicker/showSaveFilePicker`、`blob:` 下载链接，弹出「网盘文件 / 本地文件」选择器 |

所以「存档导入」里选文件时可以直接从懒猫网盘挑 ZIP，备份下载也能直接存到网盘。脚本来自官方 `lzc-file-chooser-inject.js`，放在 `content/lazycat-injects/` 打包进镜像内容目录。

> 没有配 `application.file_handler`：DMP 没有「按文件路径打开」的路由，声明 MIME 会抢走系统里所有 `.zip` 的打开方式却没有对应页面。

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

### 发布到商店（官方商店，自动）

`.github/lazycat-action.yml` 已开启官方商店：

- `images[].delivery.mode: lazycat`：check 时把上游镜像 copy 到 `registry.lazycat.cloud`，清单同步改写（官方商店只接受懒猫源镜像）；
- `stores.official.enabled: true` + `create_if_missing: true`：首次运行自动建应用、传信息、传 LPK 并提审；
- 应用信息与 5 张截图都在配置里（`.github/screenshots/`）；
- `skip_if_version_exists: true`：已在线版本不重复提交。

需要仓库 Secret `LZC_API_TOKEN`（懒猫开发者平台 → 个人访问令牌）。没配这个 Secret 时，check 阶段的镜像 copy 会失败，整条流水线会停在检查步骤。

已知冲突：官方商店里 `cloud.lazycat.app.dmp` 已被其他开发者占用（v2.1.1，2025-08），所以本包改用 `cloud.lazycat.app.dmp3`。

### Gitea 版本

`docs/gitea-actions/build-lpk.yml` 是 Gitea Actions 的 composite 版（check → 提交 → build → `akkuman/gitea-release-action`，可选 `lzc-cli` 装盒），放在 `docs/` 下不会被误触发。经验值：

- Gitea 上只能用 composite Action，reusable workflow 是 GitHub 专属；
- Action 启动要从 GitHub Release 下 6 MB 二进制，慢网会 60s 超时，可用仓库变量 `LAZYCAT_ACTION_RELEASE_BASE_URL` 指到内网镜像；
- Gitea 发给 Action 的 `GITHUB_TOKEN` 是 Gitea token，查上游 release notes 会 401 并回退模板；
- Gitea 的 Release 地址不是 GitHub 地址，私有商店会拒绝。

## 许可

打包脚本遵循 MIT；应用本身来自上游 DMP，版权归 miracleEverywhere。

# magpie 只读安全审计

审计方式：只读静态阅读本工作区源码，并核对公开元数据（GitHub 仓库与 Release、OSV、npm registry、Go module proxy、`https://usemagpie.ai/install.sh` 文本）。未运行本仓库的程序、测试、`install.sh`、二进制或镜像，未执行 `go mod download` / npm / pnpm / docker build。未向模型供应商或真实攻击目标发请求。不包含利用步骤。敏感串已打码。

## 被审提交

结论：本 fork 的默认分支与上游 tag `v0.1.3931` 是同一提交，与上游默认分支 `main` 不是同一提交。

- 被审 commit：`7c37424fcf3922f4daed4bb2af625065bf506e9e`
- 本 fork：`Silver-cybersec/magpie`，默认分支 `main`，HEAD 即上述 SHA。提交说明：`gui: the Linux app draws without backdrop blur…`（yetone，2026-11-12）。
- 上游 `yetone/magpie` tag `v0.1.3931` 指向同一 SHA。该 Release（发布在 `yetone/magpie-releases`）含 17 个资产，其中包括 `SHA256SUMS` 与 `SHA256SUMS.sig`。
- 审计当时上游默认分支 `main` 为 `d6b5f24db5d077ab0a611404f6ed88775647e6e4`（`routing: Codex seats of one name are not folded into one account card`）。GitHub compare：`7c37424…main` 为 `ahead`（ahead 1、behind 0）。本 fork 没有相对上游的独有提交，只是比 `main` 少这 1 个提交。该提交不在本次阅读范围内。
- 许可证文件：根目录 `LICENSE` 为 MIT，Copyright (c) 2026 yetone。上游仓库元数据 SPDX 为 MIT。

## 1. 结论摘要

结论：开源完整性为**部分**；个人在自己电脑上使用的总体风险为**中**。未发现隐藏后门或把提示词默认上传到 magpie 运营方的逻辑，但凭据明文落盘、本机网关默认可被本机进程调用、运行时安装的插件与供应商安装脚本，以及「curl 管道执行安装脚本」的信任模型，足以把风险定在中等。

- 核心程序（网关、CLI、TUI、Web UI、桌面壳、Docker 镜像定义、安装脚本、官网 Worker、自更新）在本 MIT 仓库，能从源码构建。官方桌面/CLI 签名发布物由另一个公开仓库 `yetone/magpie-releases` 的 CI 构建；签名私钥与苹果公证材料不在任一仓库。
- 一批订阅供应商（Qoder、Cursor、Kiro 等）的实际请求代码已不在本仓库，运行时以 npm 插件安装。插件进程能读文件系统、访问网络。安装钩子被 `--ignore-scripts` 关掉，插件自己的代码仍会执行。
- 发布版默认每天向 PostHog 报一次使用计数（版本、系统、可选的内置 id）。源码构建默认不报。提示词不在这批事件里。可关。
- API Key 与 OAuth 令牌以明文写在用户目录，文件模式 `0600`，静态存储不加密。备份包可选 AES-256-GCM。
- 自更新校验内置 ed25519 公钥；`install.sh` 只核对官网 JSON 里的 SHA-256，不核对这把签名。官网 `install.sh` 与本 commit 的 `site/public/install.sh` SHA-256 相同（`b272fe81d6405c60059a903a01ba7fe8f0b10d2563ee73e4884e9a69df932180`）。

## 2. 开源完整性

结论：本仓库公开了产品主体源码和构建配方，官方「下载即用」的签名二进制不能只靠本仓库的 CI 复现；若干运行时组件在仓库外。

### 2.1 各交付物能否由本仓库源码构建

- 桌面 App：`make app`（macOS，`.app`，需本机 cgo / WebKit）、`make release-windows`（Windows，系统 WebView2，可交叉编译）、`make release-linux`（Linux，GTK 3 + WebKitGTK 4.1，必须在对应架构上原生编译）。入口 `main.go`，界面资源在 `internal/gui/assets`，用 Wails v3。
- TUI：`internal/tui`。`make cli` 使用 `-tags nogui`，不链接桌面 WebView。
- Web UI：同一套 `internal/gui/assets`，由进程内 `Handler` 提供。`magpie web` 与「浏览器访问」把它提供给浏览器（`internal/gui/web.go`、`apppage.go`）。没有独立的前端打包工程。
- CLI：`internal/cli`。`make release-cli` 交叉编译 `darwin/arm64`、`darwin/amd64`、`linux/amd64`、`linux/arm64`、`windows/amd64`、`windows/arm64`、`android/arm64`。
- Docker：根目录 `Dockerfile`。构建阶段 `CGO_ENABLED=0 -tags nogui`，产物是终端/Web/CLI 网关，不是 GTK 桌面程序。基础镜像 `golang:1.26-alpine` 与 `debian:12-slim`，未锁 digest。
- Termux：`Makefile` 在 `GOOS=android` 时使用 `nogui`。安装脚本拉取 `magpie-cli-android-$arch`。源码可以自己 `go build -tags nogui`。
- 安装脚本：`site/public/install.sh`，由官网 Worker 原样提供。
- 自更新：`internal/update`，编进同一个二进制。

### 2.2 构建与发布流水线

- 本仓库 `.github/workflows` 公开：`test.yml`、`docker.yml`、`docker-daily.yml`、`docker-stable.yml`、`release.yml`、`privacy-filter.yml`、`ui-preview.yml`、`notes-mirror.yml`。
- `release.yml` **不在本仓库编译**桌面或 CLI。它用 `RELEASES_REPO_TOKEN` 向 `yetone/magpie-releases` 发 `repository_dispatch`，等对方 Release 凑满 17 个资产后，把同一批文件复制回本仓库 Releases。注释写明：自更新、`install.sh` 和官网只读 `magpie-releases`。
- `yetone/magpie-releases` 公开（描述 “Signed builds of magpie”）。其 `release.yml` 用只读 deploy key 检出 `yetone/magpie` 的对应 ref，执行 `make app` / `make release-cli` / `make release-windows` / `make release-linux`。
- 签名与公证只在 `magpie-releases`：macOS Developer ID（`MAC_CERTIFICATE_P12`，没有则 ad-hoc 签名、不公证）、`UPDATE_SIGNING_KEY` 对 `magpie release <version>\n` + `SHA256SUMS` 做 ed25519 签名，产出 `SHA256SUMS.sig`。工作流会核对公钥与二进制内嵌的那把一致。Windows 与 Linux 应用**没有**代码签名。
- 同一工作流用 `actions/attest-build-provenance` 做 SLSA 证明。证明绑定的是 `magpie-releases` 这次 run 处理过这些字节，**不是**源码仓库的 commit。更新器仍以 ed25519 为主，GitHub 证明是附加检查。
- 发布说明的中文由 `magpie-releases` 调用 DeepSeek 生成，不是从本仓库编译出来的。
- 同一工作流会按 `SHA256SUMS` 重写 Homebrew formula / cask（`scripts/update-tap.sh`，文件在 `magpie-releases`）。
- Docker 镜像由**本仓库** `docker.yml` 构建并推到 `ghcr.io/<owner>/magpie`（上游即 `ghcr.io/yetone/magpie`），清单有 build provenance。`:daily`、`:stable` 只移动已有 digest，不重新编译。`:stable` 读 `https://usemagpie.ai/api/latest?channel=stable`。
- 因此：源码与配方公开，第三方可以编出功能相同的未签名二进制；与官方下载字节级一致还依赖 `magpie-releases` 的 runner、Go 工具链、cgo 依赖和只存在于 secret 里的签名密钥。不能称为可独立复现的签名发布。

### 2.3 闭源或外部组件

- 「mover ≥ 0.2.54」不是闭源二进制。`mover` 是 `internal/provider/migrate.go` 里的 Go 结构，把已下线的内置订阅账号交到社区插件。Qoder 的包名是 `@magpie-community/opencode-qoder-auth`（`internal/provider/migrate_qoder.go`）。`0.2.54` 是历史注释里「模型配置字段」的最低版本；**本 commit 的 `min` 已是 `0.2.67`**。npm `latest` 元数据同为 `0.2.67`，许可证 MIT，`repository` 指向 `github.com/magpie-community/plugins` 的 `packages/qoder`，`scripts` 为空。该社区仓库公开且为 MIT。OSV 对这个包名加 `0.2.67` 返回 0 条。
- 已退休、改由插件提供的订阅见根目录 `AGENTS.md` 的表（Cursor、Kiro、Qoder、Factory、Grok、Devin、Zed 等）。Antigravity 的内置代码仍在，直到账号迁完。Gemini CLI 仍内置。
- 插件运行时下载 [Bun 1.4.2](https://github.com/oven-sh/bun/releases/download)，默认构建的 SHA-256 写在 `internal/plugin/bun.go` 的 `bunSums`。
- 社区插件优先走 npm；npm 没有的版本走 `magpie-community/plugins` 的 `dist` 分支，tarball 必须对上 `index.json` 的 sha512（`internal/plugin/dist.go`）。`bun add/update/remove` 带 `--ignore-scripts`。
- 本地脱敏：`pf-serve` 从 `https://github.com/yetone/magpie/releases/download/pf-v4/` 下载，SHA-256 在 `internal/pii/pii.go`。源码与补丁在 `native/privacy-filter`，上游 pin 为 `localai-org/privacy-filter.cpp` 的 `735a6c28…`。模型 `privacy-filter-q8.gguf` 来自 Hugging Face 固定 revision，同样 pin 了 SHA-256。AstrLink Guard 与记忆用的 MiniLM 也是用户打开后才下载，并核对哈希。
- 模型目录：`https://models.dev/api.json` 与 `models.json`（`internal/catalog`）。失败时用二进制里带的 gzip 副本。
- 供应商 CLI：用户开始某些登录时，会执行对方的安装脚本，或在没有 shell 时由 magpie 自己下载。见第 3 节。
- 官网 `usemagpie.ai` 的 Worker 源码在 `site/`（`worker.js`、`wrangler.jsonc`）。部署账号、R2 桶、PostHog 项目和 DeepSeek 密钥不在仓库。Worker 自己写明：它只搬运 `SHA256SUMS` 和签名，校验在 App 里做。
- 无 `.gitmodules`，无指向私有 Git 服务器的 `replace`。`go.mod` 唯一 `replace` 是本地 `./third_party/jsonschema`（上游 v6.0.3，Apache-2.0，少了 `init` 时编译 metaschema；说明在 `third_party/jsonschema/MAGPIE.md`）。
- `github.com/MoeForge/moejs` v0.1.0 与 `moego` v0.1.0-alpha.1 编进二进制，Apache-2.0。proxy.golang.org 的 `.info` 指向公开 GitHub `github.com/moeforge/moejs`、`github.com/moeforge/moego`。网关中间件在进程内用 moejs 跑（`internal/middleware`）。OpenCode 形态的供应商插件用 Bun（`internal/plugin/host.js`）。
- `internal/qoder/endpoints.go` 保留迁账号所需的凭据形状，注明移植自 MIT 的 CLIProxyAPI，目录内有对应 `LICENSE`。请求路径已不在这里。

### 2.4 许可证一致性

结论：仓库主体是 MIT，编进二进制的第三方以 Apache-2.0、Unicode-3.0 等许可随 `THIRD_PARTY_NOTICES` 分发。未看到 GPL/AGPL 直接依赖或闭源 SDK 被静态链接进 magpie 本身。

- HEIC 解码 `github.com/gen2brain/h265` 注释写明是纯 Go，不是 cgo 链接的 x265。
- `modernc.org/sqlite` 是纯 Go 的 SQLite。
- 社区插件是独立 MIT 包，不改变本仓库许可证，但运行的是另一份代码。
- 供应商 CLI、Bun、WebView2、GTK/WebKit 是外部程序或系统库，各有自己的许可，不由本 MIT 文件覆盖。
- Apache-2.0 代码可以和 MIT 程序一起分发，Apache 部分仍受 Apache-2.0 约束；`THIRD_PARTY_NOTICES` 抄录了 moejs 的通知。这不是「整仓单一 MIT 覆盖到每一个下载」。

## 3. 代码安全

结论：未发现隐蔽后门。安全模型是「本机代理 + 明文凭据 + 可关闭的计数上报 + 签名自更新」。主要风险是本机上任何进程都能使用网关、凭据文件落在磁盘上，以及用户触发的插件和供应商安装脚本。

### 3.1 凭据处理

- 供应商 API Key 在 `~/.config/magpie/providers.json`（`$XDG_CONFIG_HOME/magpie`；便携模式在可执行文件旁的 `data/`）。注释写明不从环境变量读 Key。`writePrivate` 以 `0600` 原子替换（`internal/provider/logins.go`）。`store` 会先 `MkdirAll(..., 0755)`，已存在的目录不会被改成 `0700`，因此目录可能可被同机其他用户列出文件名，文件内容仍是 `0600`。
- 同目录还有订阅登录、`plugin-auth.json`（插件 OAuth/API 登录）、`stash.json`（改写 agent 配置前的原值，注释写明里面有 key）、网关 Key（`internal/access`）、`web-page.json`（浏览器访问口令）。这些写入路径使用 `0600`。`install-id` 与 `stats-sent` 是 `0644`，内容是随机安装 id 和日期，不是 API Key。
- 静态存储不加密。同步/备份包在用户提供口令时用 PBKDF2-SHA256（60 万次）和 AES-256-GCM（`internal/backup/backup.go`）。订阅登录默认不进备份，除非用户选择带上。
- 界面 API 对 Key 做掩码（`provider.Mask`，`internal/gui/providers.go`）。日志经 `internal/applog` 的 `scrub`：本意不记录 Key 值，并遮挡看起来像 bearer、query token、`sk-`、`AIza`、`gh_`、JWT 的片段。拒绝网关 Key 时，只在足够长时记录末 4 个字符（`internal/gateway/lan.go` 的 `refusedKey`）。
- Codex / Claude Code 的系统钥匙串项会被读取，以便发现「当前登录的是哪个账号」并在迁移时搬走令牌。这是在读这些 agent 自己的凭据，不是读 SSH 或浏览器密码库。`NoAccountDiscovery` 或 `magpie accounts discovery` 可停止这种发现。
- Windows 上的 `0600` 并不等于 ACL。本次未在 Windows 上验证实际 DACL。

### 3.2 本地网关

- 默认监听 `127.0.0.1:3425`（`settings.DefaultPort`、`SavedAddr`）。只有设置里打开「在局域网共享」且没有用 `MAGPIE_ADDR` 钉住地址时，才听 `0.0.0.0`（`internal/gateway/lan.go` 的 `listenAddr`）。
- Docker 镜像设置 `MAGPIE_ADDR=0.0.0.0:3425`，并 `EXPOSE 3425 3430`。默认命令是 `serve`（网关），不是网页。非回环来源必须带启用的网关 Key。容器用 host 网络时，宿主机访问 `127.0.0.1:3425` 仍算本机，不需要 Key。
- 本机非浏览器客户端不需要 Key，这是给本机 agent 用的。远程、以及带 `Forwarded` / `X-Forwarded-For` / `Cf-Connecting-IP` 等头的回环连接（隧道），要启用的网关 Key。`MAGPIE_TRUST_PROXY=1` 会把这些转发头再当成可信本机，只应在你自己控制的反代上设置。
- 浏览器 CSRF / DNS rebinding：`corsGuard` 拒绝未列入设置的网页、`Origin: null`、以及 `*.localhost`（WebKit 会做 DNS）。`rebound` 拒绝「浏览器特征 + Host 不是本机」的回环请求。带有效网关 Key 的调用可以通过。历史问题 #2242、#2256 在代码和 `LESSONS.md` 里有对应修复。
- 桌面窗口的 `/api` 在进程内，没有 cookie（`docs/subsystems/gui-shell.md`）。`magpie web` 和「浏览器访问」用随机 Key 换 `SameSite=Lax`、`HttpOnly` cookie，`/api` 还限制为页面自己的源。浏览器访问默认关闭；`lan` 模式听 `0.0.0.0`，没有 TLS 时口令在网上是明文，除非使用 `--tls-cert`。
- 转发 Key：`provider.KeepsCredentials` 只跟随同一 origin 的重定向（或同主机的 http→https）。跨主机重定向不跟随，因此 `x-api-key` 一类头不会被 Go 默认客户端带到另一个域名。Key 发往用户为该供应商配置的 base URL。`InsecureTLS` 仅在用户打开时、且只对该供应商主机跳过证书校验。
- 插件 `fetch` 使用插件自己的逻辑和账号令牌。恶意或被篡改的插件可以把令牌发到别的域名。这是插件信任边界，不是网关重定向逻辑。

### 3.3 外联与遥测

结论：发布版有每日计数，默认不含提示词或对话正文。对话正文只在用户打开的 OTLP「包含正文」时发到用户填的收集器。

magpie 自己的服务与下载（会主动连接）：

| 主机 | 用途 |
| --- | --- |
| `usemagpie.ai` | 更新 feed `/api/latest`、安装脚本、发行说明、插件安装计数 `/api/plugins/report`、插件统计、赞助跳转 |
| `us.i.posthog.com` | 发布版每日 `magpie active` / `magpie uses` / `magpie partner`（`internal/stats`）。项目 Key 以 `phc_nyDy…BerTC` 写在源码里，按 PostHog 的模型只能写入 |
| `github.com` / `api.github.com` | 发布文件、证明、源码、技能与插件清单 |
| `gh-proxy.com` | GitHub 失败时的回退镜像（`internal/source`）；国内镜像开关不会把任意 GitHub URL 交给它 |
| `registry.npmjs.org`、`api.npmjs.org` | 插件。国内镜像开关或用户 registry 可改为 `registry.npmmirror.com` |
| `cdn.jsdelivr.net`、`raw.githubusercontent.com` | 社区插件清单 / dist |
| `github.com/oven-sh/bun` 与 npmmirror 的 bun 二进制路径 | Bun |
| `models.dev` | 模型目录 |
| `huggingface.co`、`hf-mirror.com` | 用户打开脱敏或语义记忆后的模型 |
| `open.er-api.com` | 把标价折成人民币时的汇率，无 Key |
| `status.claude.com`、`status.openai.com`、`status.moonshot.cn`、`status.deepseek.com` | 状态页摘要 |
| `antigravity-hub-auto-updater-….run.app` | 只读取官方清单里的 `version:`，放进 Antigravity 请求。不执行清单 |
| `api.day.app` | 用户配置 Bark 通知时 |

- 关闭计数：设置里的 Privacy、`NoStats`、`DO_NOT_TRACK=1`、`MAGPIE_NO_STATS=1`。`NoUsageStats` 只留「有人在用」，不报 agent/供应商/模型 id。源码构建（版本不是 release）不发送，除非设置了 `MAGPIE_STATS_HOST`。
- 计数内容：随机 `install-id`、版本、OS、架构、已连接 agent 的内置 id、供应商预设 id（自定义供应商记为 `custom`）、模型 id（自定义或未知记为 `custom`/`other`）。注释写明不发送名称、URL、账号、Key、提示词或用量数值。插件上报只含 Discover 公开列表里的包名。
- 官网页面另有 PostHog，尊重 `doNotTrack` / Global Privacy Control（`site/worker.js`）。那是网站，不是桌面程序。
- OTLP（`plugins/otel`、`internal/otlp`、`internal/settings/otel.go`）默认关闭。打开后发到用户写的 endpoint。`Bodies` 才附带请求和回复，密钥按脱敏结果，默认截断 256 KiB；`BodiesWhole` 才发全文。
- `AgentTelemetryOff` 会去关各个 agent 自己的遥测开关，这是在改那些程序的配置，不是 magpie 额外上报。
- 供应商 API 域名集中在 `internal/provider/presets.go` 与各 `*_usage.go`。只有配置了对应供应商才会连接。本次抽取到约 340 个字面主机，其中大量是预设控制台、文档和赞助链接，不逐条列出。

### 3.4 自更新

- 来源：默认 `https://usemagpie.ai/api/latest`，失败再读 GitHub；设置可固定为只走官网或只走 GitHub（`internal/update/source.go`）。两者都要过 `Release.verify`。
- 从 `0.1.3862` 起，feed 必须带 ed25519 签名，公钥编进 `internal/update/signed.go`（公开半钥，此处记为 `3XuYQE…SbYQ=`）。签名覆盖版本行和 `SHA256SUMS`。feed 里的哈希与签名清单不一致则拒绝。
- 从 `0.1.3891` 起额外向 `api.github.com` 要该 SHA-256 的 GitHub build attestation，身份必须是 `yetone/magpie-releases` 的 `release.yml`。证明缺失或验签失败则不安装。API 不可达、限流、5xx，或用户把更新源设成「只走官网」，则**仍安装**，日志写明未核对照明，ed25519 已经通过。拦下 GitHub 不能把校验降到「只信哈希」。
- `magpie update` 只在 `Newer(feed, 当前版本)` 时安装（`internal/cli/update_cli.go`），因此正常路径不会降级。`MAGPIE_UPDATE_FEED` 指向自建 feed 时**取消**签名强制（注释写明测试用）。本机若被设置这个变量，更新信任就落到那个 feed。
- `--mirror` / 设置里的镜像只允许加在 `github.com` 下载 URL 前面。feed 与哈希不走镜像。镜像改一个字节会因哈希失败。设置里的镜像还必须是 `https`。`--proxy` 是用户指定的 HTTP(S)/SOCKS 代理，代理能看见目标主机；HTTPS 正文仍由 TLS 保护，除非该代理被加入系统信任并做解密。
- macOS 替换 `.app` 时还要求与当前已安装应用同一签名团队（`internal/update/update.go`）。Windows/Linux 二进制没有这项。
- `NoAutoUpdate` 关闭自动检查与下载。发布版默认会自己检查（约 6 小时，或 `UpdateEvery`）。

### 3.5 install.sh 与卸载

- 用法就是 `curl -fsSL https://usemagpie.ai/install.sh | sh`。脚本与本 commit 的副本字节相同，但管道执行意味着**当时**官网返回什么就执行什么。脚本本身不再二次 `curl | sh` 去拉另一份安装器。
- 下载始终向 `usemagpie.ai/api/latest` 要 URL 和 SHA-256，再用 `sha256sum`/`shasum` 比对。不校验 `SHA256SUMS.sig`。镜像只改 GitHub 文件的前缀。不锁定一个版本号，装的是 feed 认为的最新版。
- macOS：把 `magpie.app` 放到 `/Applications`（不可写则 `~/Applications`），并把命令链接到 `~/.local/bin`。不调用 `sudo`。
- Linux：二进制到 `~/.local/bin`。桌面构建还会写 `~/.local/share/applications/magpie.desktop`、图标，并注册 `magpie://`。若要桌面版且缺少 WebKitGTK，在有终端时用 `sudo` 跑 apt/dnf/pacman/zypper 安装系统包。失败则退回终端构建。
- Termux：装到 `$PREFIX/bin`。
- `--uninstall` 先跑 `magpie uninstall --yes --keep-data`（断开 agent、恢复 magpie 改过的配置，保留 `~/.config/magpie`），再删命令、应用、菜单项和开机启动。`--purge` 连 magpie 的设置、供应商和用量一起删。若本机已经没有可用的 magpie，会再下载一份最新构建来做断开，用完删除。
- `internal/uninstall`：先断开 agent，再去掉守护进程、登录启动、快捷方式和 URL scheme，然后删 magpie 自己的数据。可执行文件留下。Library 里仍被 agent 使用的技能/服务器会保留。

### 3.6 修改其他 agent 的配置，以及账号切换

- 连接 agent 时会改其配置，把 base URL 指到本机网关，并写入模型选择。涉及的目录包括 `~/.claude`、`~/.codex`，以及代码里支持的其他 agent（VS Code、Zed 等，`internal/agent`）。改之前的值进 `stash.json`。断开和卸载按 stash 写回；用户在 magpie 之后又改过的字段会留下。只删 `~/.config/magpie` 而不执行 uninstall，agent 配置会继续指向已经不在的 `127.0.0.1:3425`（`internal/uninstall/uninstall.go` 的说明）。
- 额度用完后的行为是产品功能，不是隐藏开关：网关按路由把请求交给仍有额度的 Key 或账号；Codex / Claude Code 还可以把该 agent 的登录切到下一个仍可用的账号（`docs/subsystems/providers-accounts.md` 的 Used-up switch）。多账号、自动换账号、以及用订阅令牌走本地网关而不是官方客户端，通常落在各供应商服务条款里对账号共享、自动化登录和非官方客户端的限制范围内。可能的后果是账号被限制。这里不描述令牌如何提取或如何绕过客户端检查。

### 3.7 恶意代码与后门排查

结论：在本次阅读的范围内，没有看到混淆后的第二套载荷、隐藏的指令控制地址、读取 `~/.ssh` 私钥、读取浏览器 Cookie 数据库，或在 `go:generate` 之外于构建时执行来路不明的下载。

- `internal/plugin/host.js` 没有 `eval` / `new Function`。它会改写子进程的 stdio，避免插件打印破坏协议。Bun 插件没有沙箱：能读文件系统、发网络请求、启动进程。moejs 中间件跑在网关进程里，单次钩子有时间上限。
- `exec` / `bash -c` 出现在明确的安装与系统集成路径：供应商 CLI、可选的 rtk 安装脚本（`https://raw.githubusercontent.com/rtk-ai/rtk/refs/heads/master/install.sh`）、PenguinHarness 的安装脚本、Claude/Codex 官方安装命令、Linux 桌面包依赖。这些是用户动作触发的，不是启动时静默执行。
- Grok 在没有 shell 时的自带下载（`internal/provider/grok_install.go`）只检查版本号形状和 `--version` 能跑，**没有**独立哈希。Claude Code 的下载对照的是同一发布源 `downloads.claude.ai` 的 `manifest.json` 里的 SHA-256 和大小，不是另一把离线公钥。
- `native/privacy-filter/build.sh` 在 CI 里拉取 pin 住的 privacy-filter.cpp、Vulkan 头文件，以及 LunarG SDK（有 SHA-256）。它不随普通 `go build` 运行。产物是用户打开脱敏模型时才下载的 `pf-serve`。
- 未发现开机之外的隐藏持久化。正常的持久化是用户打开的登录启动、`magpie daemon`、以及上述 agent 配置。

## 4. 供应链

结论：Go 模块由 `go.sum` 锁定，唯一 `replace` 指向本仓库；没有产品级 npm 安装钩子；Docker 与本仓库的 GitHub Actions 用浮动标签。OSV 在若干锁定版本上有漏洞记录，其中更新证明校验和 HTML/WebP 解码与 magpie 的代码路径相邻，其余多为间接依赖或本地数据。未发现被 OSV 标成恶意包的直接依赖。社区插件源码不在本仓库，不能凭这次阅读宣称插件无恶意。

### 4.1 清单与锁定

- `go.mod`：`go 1.26.8`，模块 `github.com/yetone/magpie`。`go.sum` 在库。`replace` 仅 `github.com/santhosh-tekuri/jsonschema/v6 => ./third_party/jsonschema`。无 `GOPRIVATE`、无私有代理。直接依赖的模块路径是公开上游或上述 MoeForge。proxy.golang.org 能返回 moejs / moego 的公开 origin。状态：**已核实**。
- 前端：产品 UI 无 `package.json`。`.github/ui-preview/package.json` 只依赖 `playwright` `1.61.1`，无 `scripts`，因此没有 postinstall。状态：**已核实**（该文件）。测试夹具里的 `package.json` 不参与发布。
- Dockerfile：`golang:1.26-alpine`、`debian:12-slim`，标签浮动，未锁 digest，构建参数 `VERSION` 来自 tag 或 SHA。状态：**已核实**。
- 子模块：无 `.gitmodules`。状态：**已核实**。
- 脚本拉取：`install.sh` 拉 feed 与发布文件（有 SHA-256，无签名）；`catalog` 的 `go:generate` 会 `curl https://models.dev/api.json`（开发时，不在普通构建）；`privacy-filter/build.sh` 见上。状态：**已核实**。
- GitHub Actions（本仓库）：`actions/checkout@v4`/`@v5`、`setup-go@v5`、`docker/*@v4`/`@v7`、`oven-sh/setup-bun@v2` 等均为可变 tag，不是 commit SHA。`magpie-releases` 的 release 工作流对 checkout、artifact、attest、gh-release **有** SHA pin。状态：**已核实**。
- npm 插件：`--ignore-scripts` 避免安装期脚本；运行期代码仍完全信任。自定义 `NPM_CONFIG_REGISTRY` / `.npmrc` 会把安装指到用户指定的 registry。状态：**已核实**（调用参数），插件内容 **无法确认**。

### 4.2 OSV / GHSA（对锁定版本查询，2026-11-12）

查询：`https://api.osv.dev/v1/querybatch`，版本为 `go.mod` / `go.sum` 中的版本。`yetone/magpie` 的 GitHub Security Advisories 列表长度为 0。下列「已核实」只表示 OSV 把该 id 标在这个版本上，不是已复现。

| 依赖与版本 | OSV / GHSA | 摘要（OSV 原文缩短） | 状态 |
| --- | --- | --- | --- |
| `github.com/klauspost/compress` v1.18.3 | GHSA-259r-337f-4rfw / CVE-2026-63209；另有 GO-2026-6865（两块帧 panic） | s2 越界；压缩库 panic | **已核实**命中。本仓库用 s2 压缩**本地**用量快照（`internal/usage/request_snapshot.go`），zstd 用于会话与同步。不是对外提供的解压服务。可达性未证明 |
| `github.com/sigstore/sigstore-go` v1.1.4 | GHSA-9vcr-p3rj-q5q6（CVSS 3.1 完整性高、攻击复杂度高）；GHSA-wqqc-jjcq-vfxm（完整性低） | 证明校验在日志或时间戳上的缺陷 | **已核实**命中。调用点是更新证明（`internal/update/attest.go`）。ed25519 清单签名不依赖这条路径 |
| `github.com/sigstore/sigstore` v1.10.0 | GHSA-fcv2-xgw5-pqxf | 旧 TUF 客户端路径穿越写文件 | **已核实**命中版本。本仓库 Go 文件未直接 import 该模块。是否链进二进制 **无法确认** |
| `golang.org/x/crypto` v0.53.0 | GO-2026-5932（openpgp）；GO-2026-6303 / 6354 / 6355（ssh） | 停更的 openpgp；SSH 通道问题 | **已核实**命中版本。业务代码未直接 import 这些子包。**无法确认**是否被间接链入 |
| `golang.org/x/net` v0.56.0 | GO-2026-6603、6610、6611、6612、6617、6860、6866 等 | 多条描述 HTTP/2；6860/6866 描述 `x/net/html` | **已核实**命中版本。`x/net/http2` 用于访问上游的客户端（`internal/netproxy/calm.go`）。`x/net/html` 用于搜索结果解析（`internal/gateway/search_fetch.go`）。网关监听用标准库 `net/http`。哪几条落到客户端或 HTML 解析 **无法确认** |
| `golang.org/x/text` v0.39.0 | GO-2026-6629 | `secure/precis` 解析 panic | **已核实**命中版本。未看到业务代码 import 该子包。**无法确认** |
| `golang.org/x/image` v0.43.0 | GO-2026-6222 | VP8L（无损 WebP）解码内存 | **已核实**命中版本。本仓库 import 了 `x/image/webp` 与 `draw`。是否解码不受信任的 VP8L **无法确认** |
| `google.golang.org/grpc` v1.76.0 | GHSA-p77j-4mvh-x3m3（机密性与完整性高，授权绕过）；另有数条 xDS / HTTP/2 DoS | gRPC 服务端问题 | **已核实**命中版本。本仓库 `*.go` 未直接 import。属间接依赖。magpie 不对外提供 gRPC 服务。可达性 **无法确认** |
| 其余直接依赖（bubbletea、wails v3.0.0-beta.24、sqlite、uuid、fsnotify、websocket、jsonschema v6.0.3、gjson 等） | 本次 query 无 vuln | — | **已核实**无返回 |
| `@magpie-community/opencode-qoder-auth` 0.2.67 | OSV query 无 vuln | npm 元数据 MIT、无 scripts | **已核实**（元数据与 OSV）。包内源码未审 |

未把上述条目标成「冲突」：OSV 的版本匹配与「本进程会执行到有缺陷的函数」不是同一件事。没有一条被标成恶意包投毒。

## 5. 使用建议

结论：在自己的电脑上，把网关留在回环、凭据只留在本机账户、关掉不需要的上报和自动更新，并只安装你核对过来源的插件，可以把第 1 节的中等风险压到日常可接受的范围。

- 从源码构建，或只使用你能核对 `SHA256SUMS.sig` 的 `magpie-releases` 资产。不要用未核对签名的 `curl | sh` 作为唯一安装方式；若使用，先下载脚本再读。
- 保持默认 `127.0.0.1:3425`。不要为了省事设置 `MAGPIE_ADDR=0.0.0.0` 或 `MAGPIE_TRUST_PROXY=1`。局域网共享时使用单独的网关 Key，并限制模型与账号。Docker 桥接发布必须使用 Key；host 网络下回环仍然无 Key。
- 浏览器访问优先「仅本机」加 `--tls-cert`。不要把带 `k=` 的链接发到不受信任的聊天里。
- Key 文件留在自己的账户下。整机共享账户时，其他人能读 `0600` 文件。需要拷到另一台机器时用带口令的备份，不要把 `providers.json` 放进同步盘明文。
- 设置中关闭使用计数（或导出 `DO_NOT_TRACK=1` / `MAGPIE_NO_STATS=1`）。不要打开 OTLP Bodies，除非收集器是你自己的。
- 若不需要自动换二进制，打开 `NoAutoUpdate`，更新时看版本号是否变新，并让证明检查能连上 GitHub。不要在日常环境设置 `MAGPIE_UPDATE_FEED`。
- 插件只装 `magpie-community` 上你看过的包。不要把 npm registry 指到不认识的镜像。不需要的订阅发现可以关掉。
- 理解多账号自动切换可能违反供应商条款并导致封号。这是使用方式的风险，不是这次审计里的后门结论。
- 卸载用 `magpie uninstall` 或安装脚本的 `--uninstall`，以便写回 agent 配置。

## 6. 未覆盖与无法确认

- 未运行、未单步、未对发布二进制做符号或哈希对比，因此不能证明 `v0.1.3931` 的资产就是这个 commit 的比特级产物。只能确认公开工作流声明它检出该 ref 并执行 Makefile。
- 上游 `main` 上多出的 `d6b5f24` 未读。
- `magpie-community/plugins` 各包源码、Bun 二进制本体、`pf-serve` 二进制本体未审。只审了本仓库如何下载、如何校验、以及 Qoder 包的 registry 元数据。
- 线上 Worker 与 R2 是否等于本 commit 的 `site/`：只核对了 `install.sh` 一致。Worker 部署可以在不改这个文件的情况下改变 `/api/latest`。
- `magpie-releases` 的 `scripts/notes.sh`、公证失败后的分支、Homebrew tap 的其余文件未逐行读完。
- OSV 命中项未做到「补丁是否覆盖 magpie 的调用点」。
- 各供应商当前服务条款原文未逐份核对。
- Windows ACL、macOS 钥匙串 ACL、以及发行版上 WebKit 的实际 DNS 行为未在本环境验证。
- 没有一套完整的「恶意包」名录被整份对照；结论限于 OSV 查询、模块路径和公开仓库元数据。

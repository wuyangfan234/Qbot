# Hermes 记忆位置与 NAS 访问方式

调查时间：2026-10-09，本机（macOS，计算机名「吴杨凡的Mac mini」，LocalHostName `wuyangfandeMac-mini`，默认路由经 `en1` 到 `192.168.8.1`）。

本报告只记录协议、主机、端口、路径、用户名，以及凭据「是否存在、在哪个文件的哪一项」。口令、token、密钥、私钥材料均未抄写。

## 结论

Hermes 已安装并在运行。长期记忆有两层：Hermes 自己的 Markdown 记忆在 `~/.hermes/memories/`，当前启用的记忆提供方是 Memmy，库文件在 `~/.memmy/memory-service/memory.sqlite`。NAS 操作手册写在 Hermes skill 里，不在独立的「NAS 配置文件」中。

家庭 NAS 是绿联 **OasisNAS**（Debian 12 + UGOS）。本次实测 `oasis.local` 解析到 `192.168.8.215`。访问方式是：

- 管理与文件操作主通道：**SSH 22**，用户 `wuyangfan`，本机私钥 `~/.ssh/id_ed25519` 免密（本次 `BatchMode` 登录成功）。
- 文件共享：**SMB**（445/139 在听，`smbd` 为 active）。AFP 548、NFS 2049 拒绝连接。
- 管理后台：**HTTPS 9443**。
- 本机 **当前没有挂载** 这块 NAS。`/Volumes/oasis-date` 是本机外接 APFS 盘，不是网络共享。

## 调查过的位置

| 位置 | 结果 |
|---|---|
| `which hermes` | `/Users/wuyangfan/.local/bin/hermes` |
| `~/.hermes` | Hermes home，记忆、skill、密钥目录都在这里 |
| `~/.memmy` | Memmy 本地记忆服务 |
| `~/Library/Application Support/hermes` 与 `Hermes` | 桌面端 Chromium 配置，不是 NAS 手册 |
| `/Applications/Hermes.app`、`/Applications/Memmy.app` | 已安装 |
| `~/Library/LaunchAgents` 中文件名含 hermes 的 plist | 见下文运行组件 |
| `brew services` | 无 hermes 服务 |
| Docker | 本机无 hermes 容器 |
| `ps` | gateway、dashboard、web-ui、agent bridge 在跑 |
| `mount`、`df`、`ls /Volumes`、`smbutil statshares -a` | 无 SMB/AFP/NFS/SSHFS 挂载 |
| `~/.ssh/config`、`~/.ssh/config_oasis`、`known_hosts` | 见「易混淆项」 |
| 登录钥匙串与系统钥匙串的条目属性（未使用会导出密文的参数） | 有 SMB 与若干 NAS 相关条目，值未读 |

## Hermes 本体

- 版本：Hermes Agent v0.21.6+138.g302c5d9.dirty（2026.9.24），安装目录 `~/.hermes/hermes-agent`，安装方式 git，Python 3.14.7。
- 配置：`~/.hermes/config.yaml`。`memory.memory_enabled: true`，`memory.provider: memmy-memory`。
- 同时在跑的相关进程：gateway、本机 dashboard `127.0.0.1:9119`、hermes-web-ui、agent bridge。LaunchAgents 里与之对应的有 `ai.hermes.gateway-manual.plist`、`ai.hermes.dashboard.plist`、`ai.hermes.web-ui.plist` 等。若干 `*.disabled-*` / `*.bak-*` 是已停用的旧项。
- `~/.hermes/memory/` 是空目录。`cn.wuyangfan.hermes-supermemory-sync.plist` 存在，但 NAS 上 `supermemory.service` 本次为 inactive，局域网 `6767` 拒绝连接。

## 记忆的实际存放位置

### 1. Hermes 内置文件记忆

| 文件 | 权限 | 大小 | 说明 |
|---|---|---|---|
| `~/.hermes/memories/MEMORY.md` | 600 | 10688 字节，92 行 | 2026-10-09 14:16 仍在更新。NAS 相关内容是操作备忘，不是连接配置表 |
| `~/.hermes/memories/USER.md` | 600 | 4610 字节，42 行 | 未出现 NAS 主机、协议或共享路径 |
| `~/.hermes/memories/*.lock` | 600 | 0 | 锁文件 |
| `~/.hermes/pending/memory/` | — | 有 json | 待写入记忆队列，内容与 MEMORY.md 的 NAS 备忘同类 |

`MEMORY.md` 里和 NAS 直接相关的非敏感事实：

- ZenTrade / QVeris 的正式 key 写在 NAS 上的 `secrets.json`（本机另有 `~/.hermes/secrets/*.env`）。
- Cloudflare 隧道 ID `4e78c08e-c95b-4a04-878c-8616cccfc4c6` 为 token 远程管理；记忆写明本机 DNS token 改不了隧道 ingress。
- ZenTrade 对外曾用 `zentrade.wuyangfan.cn` 指到 `http://172.17.0.1:3018`；2026-10-08 起安卓默认地址改为节点小宝 `https://deeprs.iepose.cn`。这些是应用入口，不是磁盘共享协议。

同一文件里还有**与 NAS 无关**的明文口令和管理密钥（约第 33 行 traces/OIC 登录密码，约第 63 行管理面板密钥）。本报告不转写这些值。

### 2. 当前启用的 Memmy 记忆

| 项 | 值 |
|---|---|
| 插件 | `~/.hermes/plugins/memmy-memory/`（plugin.yaml 1.1.4，kind exclusive） |
| 插件配置 | `~/.hermes/plugins/memmy-memory/config.json`：`endpoint` 为 `http://127.0.0.1:18960`，`token` 项存在但为空字符串，`userId` 为 `local-user` |
| 服务配置 | `~/.memmy/config.yaml`：`memmyMemory.storage.mode=local`，`backend=sqlite` |
| 库文件 | `~/.memmy/memory-service/memory.sqlite`（约 2.4GB，另有 shm/wal） |
| 进程 | Memmy.app，记忆服务命令行带 `--config /Users/wuyangfan/.memmy/config.yaml` |
| 规模 | 表 `memories` 6158 行；其中未删除且提到 `192.168.8.215` / `oasis.local` / `OasisNAS` 的 222 行（SkillMemory 111，LongTermMemory 110）。`user_memories.content` 提到 NAS 的未删除行 53 行 |
| 技能说明 | `~/.hermes/skills/memmy-memory/SKILL.md` |

这 222 行里有 169 行文本包含 password、token、密钥、密码一类词。本次只做计数，没有导出正文。`~/.memmy/config.yaml` 里还有 apiKey、token、JWT 字段（`app.cloudUuid` 等），值未读取进本报告。

`config.yaml` 的 MCP 段另有一项名为 `memory` 的服务，命令是 `npx -y @modelcontextprotocol/server-memory`。它不是上面这份 sqlite，本次也没有把它当成 NAS 手册来源。

### 3. 被 Hermes 当作操作记忆使用的 skill

这是 NAS 连接方式写得最完整的地方：

- `~/.hermes/skills/devops/oasis-nas/SKILL.md` 及 `references/`
- `~/.hermes/skills/devops/nas-homelab-operations/SKILL.md` 及 `references/oasis-nas-env.md`
- SMB 投递约定在 `~/.hermes/skills/devops/source-snapshot-delivery/SKILL.md`：放到 `/volume2/同步文件/<类别>/<项目>-<日期>/`，Finder 用 `smb://<NAS-IP>` 取

`oasis-nas` 的「环境坑」一节写了 SSH 密码兜底，并带一段部分掩码的口令提示。本报告不转写该提示。同节写明私钥登录才是常规方式。

## NAS 访问方式

以下主机与系统信息在 2026-10-09 用 ping、TCP 连接和一次 `BatchMode` SSH 复核过。SSH 只读取主机名、Samba 共享定义、`secrets.json` 的权限和键名、磁盘容量、监听端口和服务状态，没有打印任何键值。

### 身份与地址

| 字段 | 值 |
|---|---|
| 主机名 | `OasisNAS` |
| mDNS | `oasis.local` → `192.168.8.215`（本次 ping，约 2.2–2.6 ms） |
| 网段 | `192.168.8.0/24`（skill 与本机默认网关 `192.168.8.1` 一致） |
| 系统 | Debian GNU/Linux 12 (bookworm)。skill 记录 UGOS 1.16 / 1.16.0，本次未再读出版本文件 |
| SSH | 端口 **22**，用户 **`wuyangfan`**，公钥认证。本机私钥 `~/.ssh/id_ed25519`（600），公钥 `~/.ssh/id_ed25519.pub`（644）。`known_hosts` 有 `192.168.8.215` 与 `oasis.local` |
| sudo | skill 记录已配置 `NOPASSWD`（`/etc/sudoers.d/wuyangfan-nopasswd`）。本次未执行需要 sudo 的命令 |
| 管理后台 | `https://oasis.local:9443`（TCP 可连）。Web 登录口令未读取 |
| SMB 服务名 | `OASISNAS._smb._tcp.local` |

skill 写过历史 IP 曾是网段内 `.125`，后来漂移到 `.215`。本次 `.215` 与 mDNS 一致，但 skill 仍提醒 DHCP 会变，丢失时应重新解析 `oasis.local`。

### 协议与端口（本次从 Mac 探测）

| 端口 | 结果 | 含义 |
|---|---|---|
| 22 | 开放 | SSH |
| 445、139 | 开放，NAS 上 `smbd` active | SMB |
| 9443 | 开放 | UGOS HTTPS 管理 |
| 548 | 拒绝 | AFP 未听 |
| 2049 | 拒绝 | NFS 未听 |
| 873 | 拒绝 | rsync 守护未听 |
| 23 | 开放 | Telnet 仍在听。skill 已把它标为明文风险待关闭。本次没有登录 |
| 6767 | 拒绝 | 自托管 Supermemory。`supermemory.service` 为 inactive |
| 8899 | 开放 | skill 中的 vibe-trading，绑定在 NAS 局域网地址 |
| 47331 | 开放 | skill 中的 OpenAlice |
| 10086 | 开放 | skill 中的 1Panel |

没有发现 AFP、NFS、SSHFS 或 WebDAV 作为这块盘的挂载协议。文件共享就是 SMB；远程 shell 是 SSH。

### SMB 共享

定义文件：NAS 上 `/etc/samba/smb.conf` 再 `include` `/etc/samba/smbshare.conf`。下面是该文件里的非敏感字段。`writeable = no` 与 `write list` 同时存在，写权限以 write list 为准。

| 共享名 | 路径 | 浏览 | valid users | write list |
|---|---|---|---|---|
| `personal_folder` | `%H`（登录用户的 home） | yes | `@ughomeusers` | （该节未单列 write list；`writeable = yes`） |
| `Time Machine1` | `/volume1/Time Machine1` | yes | `@admin wuyangfan cloud` | 同左 |
| `迅雷下载` | `/volume1/迅雷下载` | yes | `thunder wuyangfan cloud @admin @users` | 同左 |
| `docker` | `/volume2/docker` | yes | `@admin wuyangfan cloud` | 同左 |
| `同步文件` | `/volume2/同步文件` | yes | `@admin wuyangfan cloud` | 同左 |
| `cp32 视频` | `/volume1/cp32 视频` | yes | `@admin` | `@admin` |

Finder 最近使用的地址（`~/Library/Application Support/com.apple.sharedfilelist/com.apple.LSSharedFileList.RecentServers.sfl4`，URL 中无口令）包括：

- `smb://192.168.8.215`
- `smb://cloud@192.168.8.215/同步文件`
- `smb://cloud@192.168.8.215/迅雷下载`
- `smb://cloud@192.168.8.215/Time Machine1`
- `smb://cloud@192.168.8.215/docker`
- `smb://cloud@192.168.8.215/personal_folder`
- `smb://wuyangfan@OASISNAS._smb._tcp.local/cp32 视频`
- `smb://wuyangfan@OASISNAS._smb._tcp.local/personal_folder`

收藏服务器里有 `smb://192.168.8.215`。这些都是历史记录，不代表当前已挂载。

### 存储卷（本次 `df`）

| 挂载点 | 容量 | 已用 | 可用 | 用途（来自 skill，与路径一致） |
|---|---|---|---|---|
| `/volume1` | 7.2T | 2.9T | 4.4T | 主存储（媒体等） |
| `/volume2` | 901G | 179G | 723G | 次存储（docker、同步文件） |

skill 里「volume1 剩 4.8T」是旧快照，以本次 4.4T 可用为准。

### 卷上与访问相关的路径

- 用户主目录：`/home/wuyangfan`（SSH 的 `personal_folder` 即 `%H`）
- ZenTrade 数据与凭据文件：`/volume2/docker/zentrade/data/user_data/secrets.json`
- 该文件本次 `stat`：模式 **670**，属主 **root:root**，560 字节。skill/旧会话里写过 600，与这次不一致。
- 键名（只有名字）：`ai_api_key`、`ai_base_url`、`ai_context_window`、`ai_max_output_tokens`、`ai_model`、`ai_provider`、`ai_user_agent`、`qveris_api_key`、`qveris_daily_limit`、`tickflow_api_key`
- 容器内对应路径（旧会话记录，本次未进容器复核）：`/app/data/user_data/secrets.json`

### 公网与隧道（来自记忆和 skill，本次未改任何隧道）

- NAS 上 cloudflared 以 token 模式跑，隧道 ID `4e78c08e-c95b-4a04-878c-8616cccfc4c6`。token 本身在容器启动参数里，本次没有 `docker inspect`，也没有把 token 写入本报告。
- 本机 `~/.hermes/.env`（600）存在键名 `CLOUDFLARE_API_TOKEN`、`CF_ACCOUNT_API_TOKEN`。记忆写明它们只有 DNS 权限，改不了隧道 ingress。同目录还有文件 `cloudflare-api.token`、`hermes-cloudflared.token`、`wiki-cloudflared.token`（均为 600）。值未读。
- 节点小宝是另一条远程入口。应用 URL 见 MEMORY.md 的安卓默认地址，不是 SMB/SSH。

## 凭据存在性（值一律未写入）

| 凭据 | 是否存在 | 位置 |
|---|---|---|
| SSH 私钥 | 存在 | `~/.ssh/id_ed25519`（600）。公钥在同名 `.pub`。NAS `authorized_keys` 由 skill 记录为已写入，本次未打开该文件 |
| SSH 口令兜底 | skill 声称存在 | `~/.hermes/skills/devops/oasis-nas/SKILL.md`「环境坑」节，含部分掩码提示。本次钥匙串筛选没有单独列出「192.168.8.215 的 SSH 口令」条目。实际登录走的是公钥 |
| SMB 网络密码 | 存在（未解密） | 登录钥匙串 `login.keychain-db`，class `inet`，协议 `smb`：账户 `cloud`，服务器 `192.168.8.215`，描述为「网络密码」；账户 `wuyangfan`，服务器 `OASISNAS._smb._tcp.local`；另有一条账户「无用户账户」、服务器 `OASISNAS._smb._tcp.local.` |
| Time Machine 的 SMB 密码 | 存在（未解密） | 系统钥匙串，class `inet`，协议 `smb`，账户 `wuyangfan`，服务器 `OASISNAS._smb._tcp.local.`，描述「时间机器网络密码」 |
| Vibe-Trading API 口令 | 存在（未解密） | 登录钥匙串 generic password，账户 `wuyangfan`，服务名 `Vibe-Trading OasisNAS API_AUTH_KEY` |
| WeKnora 本地账户 | 存在（未解密） | 登录钥匙串，账户 `weknora-local@oasis.local`，服务名 `com.oasis.weknora.local` |
| NAS 上的应用密钥 | 存在 | `/volume2/docker/zentrade/data/user_data/secrets.json` 的键名见上 |
| 本机对应 env | 存在 | `~/.hermes/secrets/` 下 `qveris.env`、`tickflow.env` 等，模式 600。本次没有打开内容 |
| Wi-Fi | 存在（未解密） | 系统钥匙串 AirPort 项，账户名 `Oasis`、`Oasis_5G`、`oasisw_5G`。这是无线网名，不是 NAS 文件协议 |
| Memmy / Hermes 其它 API key | 存在 | `~/.memmy/config.yaml`、`~/.hermes/.env`、`~/.hermes/config.yaml` 的 provider 段。与 NAS 磁盘访问无关，值未抄 |

## 本机当前挂载

`mount`、`df -h`、`smbutil statshares -a` 都没有 SMB、AFP、NFS、CIFS、SSHFS 或 WebDAV。

| 挂载点 | 来源 | 是不是 NAS |
|---|---|---|
| `/` 及 `/System/Volumes/*` | 内置 APFS `disk3`（物理盘 `disk0`，251GB） | 否 |
| `/Volumes/oasis-date` | 外接物理盘 `disk6` 上的 APFS `disk7s1`，约 931GB，已用 134GB | 否。名字像 NAS，介质是本地外接盘 |
| `/Volumes/Todos 0.1.12-arm64` | 磁盘映像 `disk4`，HFS | 否 |
| `/Volumes/Macintosh HD` | 指向 `/` 的符号链接 | 否 |

最近服务器列表里还有一条 `nfs://disk5s1.local/mnt/disk5s1`。当前没有对应挂载，也没有把它算作 OasisNAS。

## 易混淆项

`~/.ssh/config_oasis` 里的 Host `oasis` **不是** 这块 NAS：

- HostName `154.23.172.155`
- Port 22
- User `root`
- IdentityFile `~/.ssh/id_ed25519`

`MEMORY.md` 把 `154.23.172.155` 记为 QuantDinger。主 `~/.ssh/config` 只 Include 了 Colima 和 `github.com`，没有 Include `config_oasis`，也没有 Host 指向 `192.168.8.215`。连 NAS 时用的是 skill 里的 `wuyangfan@192.168.8.215` 或 `wuyangfan@oasis.local`，不是 `ssh oasis`。

`nas-homelab-operations` 还提到香港 `103.153.5.166` 上的 frp 可作为备用入口。本次没有连接该地址。

## 本次没有做的事

没有挂载或卸载任何卷，没有登录 SMB、Telnet 或 9443 后台，没有 `docker inspect` 隧道 token，没有把钥匙串密文解出来，也没有修改 NAS 或 Hermes 配置。

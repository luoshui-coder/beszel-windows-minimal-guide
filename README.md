# Beszel：Windows Server 2019 / 2022 极简内网监控

目标：在一台服务器的浏览器里，集中查看各 Windows 服务器的 CPU、内存和磁盘空间。

**只运行两个官方程序：一个 Hub 查看端，每台主机一个 Agent 采集端。** 本教程使用手动启动方式，不安装 Windows 服务，不配置邮件告警，不需要 Linux、Docker、WSL、Caddy 或独立数据库。

> 适合先做小规模试点。关闭程序窗口、注销账号或重启后需重新启动。不要把手动启动版当作无人值守的长期监控服务。

文档更新：2026-10-26。使用固定版本 **v0.21.1**；已复核下载包哈希，并根据现场反馈补充账号、磁盘日志和浏览器排查说明。已有 Agent 启动、SSH 连接和 C/D 盘识别的日志证据，但不代表 Windows Server 2019 / 2022 的完整实机验收。

## 你需要准备什么

| 程序 | 放在哪里 | 作用 |
|---|---|---|
| `beszel.exe` | 选一台 Windows 作为中心服务器 | 提供统一查看页面 |
| `beszel-agent.exe` | 每台被监控服务器 | 采集本机资源指标 |

中心服务器若也要监控自身，同样运行 Agent。

浏览器使用当前稳定版 **Microsoft Edge 或 Chrome**。服务器系统受支持，不代表系统内的旧浏览器能正常显示页面；不要继续使用 Edge 86 等旧版本，离线升级方法见后文。

查看方式：通过远程桌面进入中心服务器，在**中心服务器自己的浏览器**打开页面。网页仅监听本机，无需对外开放 8090。

```text
中心服务器的浏览器 → 本机 Hub：127.0.0.1:8090
                          │
                          ├── SSH 加密连接 → Windows A Agent：45876
                          ├── SSH 加密连接 → Windows B Agent：45876
                          └── SSH 加密连接 → Windows C Agent：45876
```

这里的 SSH 是 Agent 内置的监控通信，不需要安装 Windows OpenSSH Server，也不用配置远程登录账号。

## Step 1：下载、解压

在能联网的电脑下载，复制到内网后解压到固定文件夹：

- [查看端 ZIP：beszel_windows_amd64.zip](https://github.com/henrygd/beszel/releases/download/v0.21.1/beszel_windows_amd64.zip)
- [采集端 ZIP：beszel-agent_windows_amd64.zip](https://github.com/henrygd/beszel/releases/download/v0.21.1/beszel-agent_windows_amd64.zip)

例如中心程序放在 `C:\Beszel\Hub`，各机采集程序放在 `C:\Beszel\Agent`。若没有该目录的写权限，选择自己有权限的专用目录；不要覆盖旧部署或放入业务程序目录。

ZIP 解压后除了 EXE 还可能有许可证和说明文件，保留即可。本仓库只提供教程，软件从上面的官方链接下载。

生产服务器的软件传入按单位审核流程进行。本次已核对官方 ZIP 内容及 SHA-256，但这不代表已在现场通过安全或兼容性验收。校验信息见文末。

## Step 2：中心服务器启动 Hub

在查看端解压文件夹内，按住 **Shift → 鼠标右键 → 在此处打开 PowerShell 窗口**。

执行：

```powershell
.\beszel.exe serve --http=127.0.0.1:8090
```

保持窗口打开。在**同一台服务器的 Edge/Chrome**里打开：

```text
http://127.0.0.1:8090
```

第一次按页面提示创建管理员账号。账号采用邮箱格式，纯内网可填写 `admin@example.com` 作为登录标识，不要求真实邮箱或联网收信；默认首次创建无需邮件验证。不配置 SMTP 也能使用基础监控，但无法通过邮件找回密码，请保存好账号密码。

预期：出现管理页面，可以登录。

- 这里的 127.0.0.1 指中心服务器自己，不能在另一台电脑输入这个地址访问中心。
- 不开放 8090 入站端口，不把监听地址改成 0.0.0.0。
- Hub 会保存本地数据，保持程序文件夹及生成的数据目录，不要每次换文件夹启动。
- 如果程序报端口占用，先确认是否已经开了一个 Hub，不要停止不明业务进程。

## Step 3：页面添加一台服务器

点击 **Add System / 添加系统**：

| 字段 | 填写内容 |
|---|---|
| 名称 | 自己能认出来的名称，例如“应用服务器01” |
| Host / IP | 被监控服务器的实际内网 IP |
| Port | 保持 45876 |
| Public Key | 完整复制页面显示的公钥 |

保存系统。第一次连接前显示离线是正常的。

**监控 Hub 自己**：在 Hub 所在服务器也运行一个 Agent，添加系统时 Host 填 `127.0.0.1`、Port 填 `45876`，将页面公钥用于本机 Agent 启动命令。Hub 和 Agent 都要保持运行；这条本机回环连接无需开放外部入站端口，可跳过 Step 4。其它服务器仍填写各自的实际内网 IP。

本教程使用 **Hub 主动拉取的 SSH 模式**：只需要 Hub 公钥，不必在 Agent 上设置 Token、HUB_URL 或证书。界面若出现在线安装脚本，不需要执行。

> 若之前已经按其它教程设置过 DISABLE_SSH=true、HUB_URL 等环境变量，不应直接混用。本教程按未配置过的全新程序环境编写。

## Step 4：被监控服务器放行一个端口

用 Windows 防火墙图形界面操作；这是需要管理员权限的网络配置：

1. 打开“高级安全 Windows Defender 防火墙”。
2. 选择“入站规则”→“新建规则”→“自定义”。
3. 程序选本机实际的 `beszel-agent.exe`。
4. 协议选 **TCP**，本地端口填 **45876**。
5. 在“作用域”将**远程 IP 限定为中心服务器的 IP**。
6. 选择“允许连接”，选择适用的网络配置文件，命名为“Beszel 采集”并完成。

不要关闭整机防火墙，不要把远程来源设为任意地址。若已有宽泛放行规则，新增窄规则不会抵消它，需要检查。

跨网段时，让网管放行“中心服务器 → 被监控服务器 TCP 45876”。

## Step 5：启动 Agent

在被监控服务器的 Agent 解压文件夹打开 PowerShell，执行：

```powershell
.\beszel-agent.exe --key "这里替换为页面复制的完整公钥"
```

保留英文双引号，将占位文字换成完整公钥，窗口保持打开。

**监控 C、D 两个盘时**，若 C 盘已作为主卷自动识别，只需额外指定 D 盘：

```powershell
$env:EXTRA_FILESYSTEMS = 'D:'
.\beszel-agent.exe --key "这里替换为页面复制的完整公钥"
```

**还有 E 盘时**，改为执行：

```powershell
$env:EXTRA_FILESYSTEMS = 'D:,E:'
.\beszel-agent.exe --key "这里替换为页面复制的完整公钥"
```

只填写实际存在的额外盘符。系统主卷也要到页面核验，不能假设一定识别正确。无盘符挂载卷、集群共享卷、SAN/RAID 映射需另行试验。

如果 Agent 已在运行，先在原窗口按 `Ctrl+C` 停止，再在同一个 PowerShell 窗口设置环境变量并重新启动。这种 `$env:` 设置只作用于当前会话及其子进程；新开 PowerShell 或重启后需要重新设置。

返回中心页面，数个采集周期后应显示在线和资源数据。其它服务器重复 Step 3～5。

采集连接由公钥认证并经 SSH 加密；Agent 不需要数据库账号或域管理员密码。

### 如何看启动日志

```text
Detected disk name=D: device=D: mount=D: io=D: root=false
Detected disk name=C: device=C: mount=C: io=C: root=true
Starting SSH server addr=:45876 network=tcp
SSH connection established
```

- D 盘这一行表示已识别 D 盘；`root=false` 表示它不是主监控卷，不是采集失败。
- `Starting SSH server` 只表示 Agent 已监听端口，不能据此判定 Hub 已连通。
- `SSH connection established` 表示 SSH 连接已建立，仍需到 Hub 页面核验指标持续更新。
- `smartctl failed ... no serial number` 表示未取得磁盘硬件健康信息，虚拟磁盘或 RAID 环境可能出现；单凭这条日志不能判定磁盘故障，通常不影响基础 CPU、内存和磁盘容量采集。本文不以 SMART 信息作为验收条件。

## 怎么确认已经成功

只检查这四件事：

- 主机在线，CPU 和内存数据持续更新。
- 每个需要监控的系统盘、数据盘都出现。
- 容量与本机磁盘属性大致一致；GB/GiB 和不同采样时刻可能产生差别。
- 试点运行覆盖一个业务高峰，任务管理器中 Agent 占用可接受。

磁盘**空间占用率**和磁盘**活动时间/繁忙程度**是不同指标。读写速率图不代表所有 Windows PerfMon 指标都有；本文主要验收 CPU、内存与磁盘容量。

## 部署到数据库服务器时怎么选

优先在数据库服务器仅运行 Agent，将 Hub 放到独立管理服务器或其它负载较低的主机。

| 程序 | 主要资源开销 | 需要确认什么 |
|---|---|---|
| Agent | 定期读取系统指标，通常较轻量；不连接业务数据库，也不扫描数据库表或遍历磁盘文件来统计容量 | 在业务高峰观察进程 CPU、内存及数据库响应、磁盘延迟 |
| Hub | 保存监控历史、查询数据和展示页面；随主机数量、保留时间和访问量增加 | 预留数据存储空间，避免与数据库繁忙的数据盘、事务日志盘争用 I/O |

这里没有给出固定 CPU/内存保证值，实际开销需在目标主机测量。如果必须同机运行 Hub 和 Agent，先进行小规模试点，不能把“通常轻量”当成对数据库零影响的保证。

## 页面没有样式：先检查浏览器

若出现巨大 Beszel 标志、默认字体和按钮挤在一起，说明样式没有正常生效。现场反馈使用过 **Edge 86.0.622.38**，该版本过旧，浏览器兼容性应优先排查；尚不能仅凭截图认定唯一原因。

1. 在地址栏打开 `edge://settings/help` 查看版本，优先升级到当前稳定版。
2. 按 `Ctrl+F5` 强制刷新，排除缓存影响。
3. 若仍异常，按 F12 → Network（网络）→ 筛选 CSS → 刷新：请求失败或 404 时检查资源请求；返回 200 时还需确认响应是 CSS 而非 HTML，再检查浏览器兼容性和控制台错误。

如果已有获准通过内网访问 Hub 的管理电脑，可用其新版浏览器访问同一地址对比。本教程默认 Hub 只监听 `127.0.0.1`，此时只能在 Hub 本机访问；不要为这个对比测试直接开放端口。

### Windows Server 2022 离线更新 Edge

微软的 [Edge 支持系统列表](https://learn.microsoft.com/en-us/deployedge/microsoft-edge-supported-operating-systems) 包含 Windows Server 2022。无需停留在 Edge 86，也无需寻找专用的“Server 2022 浏览器版本”。

1. 在联网电脑打开 [Edge 企业版官方下载页](https://www.microsoft.com/zh-cn/edge/business/download)。
2. 选择 **Stable（稳定版）→ 最新版本 → Windows 64-bit / x64**。
3. 下载完整 MSI 离线包，通常名为 `MicrosoftEdgeEnterpriseX64.msi`，不要选 ARM64 或需要联网下载的小型安装器。
4. 传入内网服务器，关闭 Edge 后以管理员权限安装；无需为了更新浏览器停止 Hub、Agent 或数据库服务。
5. 重新打开 `edge://settings/help` 确认版本，再访问 Hub 并强制刷新。纯内网环境中，在线检查更新失败不等于离线安装失败，以已安装版本为准。

### COOP 警告是不是导致 CSS 失效

通过 `http://内网IP:8090` 访问时，可能出现 `Cross-Origin-Opener-Policy header has been ignored ... origin was untrustworthy`。它表示浏览器忽略了要求可信来源的安全响应头，通常不会阻止 CSS 加载，不能据此认定样式故障。

在 Hub 本机按本教程使用 `http://127.0.0.1:8090`，或使用可连通的 `http://localhost:8090`，通常可以避免这条警告；但仍需单独处理旧浏览器或 CSS 请求问题。跨主机长期访问如需 HTTPS，应另行规划，不属于本手动本机查看方案。

## 停止、再次启动和退出试点

- 在 Agent 窗口按 **Ctrl+C**：停止本机采集，不停止业务服务。
- 在 Hub 窗口按 **Ctrl+C**：停止集中查看。
- 再次使用：从原文件夹重新执行对应启动命令；额外盘符的环境变量也需要重新设置。
- 关闭窗口、注销或重启后需要重新启动。远程桌面“断开连接”不等于“注销”，但也不应依赖会话长期存活。
- 不再使用时，停止相关进程，移除本次创建的防火墙规则；程序和数据目录按需保留或删除。

这一版不安装服务、不修改业务配置、不安装驱动。正式服务器仍建议先用一台非关键主机测试；先尝试普通权限，遇到权限错误先定位，不直接换成域管理员。

## 最常见的几个问题

| 问题 | 先检查 |
|---|---|
| 本机页面打不开 | Hub 窗口是否运行、是否在中心服务器本机访问、8090 是否冲突 |
| 添加后一直离线 | Agent 是否运行、IP/公钥是否正确、TCP 45876 是否放行 |
| 只有系统盘 | EXTRA_FILESYSTEMS 是否设置、盘符是否存在、Agent 是否从该 PowerShell 启动 |
| D 盘日志显示 root=false | 表示额外卷已识别，不是失败；等待 Hub 连接并在详情页确认容量 |
| smartctl 提示 no serial number | 是硬件健康信息采集问题；先确认基础指标是否正常，不据此判定磁盘损坏 |
| 页面无样式、标志巨大 | 优先检查浏览器版本，再强制刷新并查看 CSS 请求 |
| 窗口一闪而过 | 不双击 EXE；在文件夹 PowerShell 中执行命令，读取报错 |
| 杀毒/EDR 拦截 | 核对官方来源和文件哈希，按单位流程处理，不关闭整机防护 |
| 重启后没有数据 | 这是手动版，需重新启动；长期自启动可后续单独配置 |

## 官方资料与核验边界

- [官方 v0.21.1 发布](https://github.com/henrygd/beszel/releases/tag/v0.21.1)
- [Hub 安装](https://beszel.dev/guide/hub-installation)
- [Agent 安装](https://beszel.dev/guide/agent-installation)
- [通信与安全](https://beszel.dev/guide/security)
- [额外磁盘](https://beszel.dev/guide/additional-disks)
- [官方校验清单](https://github.com/henrygd/beszel/releases/download/v0.21.1/beszel_0.21.1_checksums.txt)

已核验的固定版本 ZIP SHA-256：

```text
beszel_windows_amd64.zip
ea8ae9bf52a2161be7fefd29d25d108980ae9db97d5fb7a439a1bcffcdafdd10

beszel-agent_windows_amd64.zip
21bf63f3777244bb2532aa69d673c0c423dc2bb43402fc7ec93f2036abb99698
```

核验范围包括发布资产、文件内容/哈希及源码参数；现场反馈日志证明 Agent 曾建立 SSH 连接，并在配置后识别 C、D 两个卷。尚未完成目标 Windows Server 2019 / 2022 的全面验收，也未获得浏览器升级后页面恢复的反馈。现场兼容性、权限、真实占用和指标准确性仍需试点确认，本教程不是部署完成报告。

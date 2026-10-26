# Beszel：Windows Server 2019 极简内网监控

目标：在一台服务器的浏览器里，集中查看各 Windows 服务器的 CPU、内存和磁盘空间。

**只运行两个官方程序：一个 Hub 查看端，每台主机一个 Agent 采集端。** 本教程使用手动启动方式，不安装 Windows 服务，不配置邮件告警，不需要 Linux、Docker、WSL、Caddy 或独立数据库。

> 适合先做小规模试点。关闭程序窗口、注销账号或重启后需重新启动。不要把手动启动版当作无人值守的长期监控服务。

文档整理：2026-10-26。使用固定版本 **v0.21.1**，官方发布资产与配置核对日期为 2026-10-25；未在目标 Windows Server 2019 实机验收。

## 你需要准备什么

| 程序 | 放在哪里 | 作用 |
|---|---|---|
| `beszel.exe` | 选一台 Windows 作为中心服务器 | 提供统一查看页面 |
| `beszel-agent.exe` | 每台被监控服务器 | 采集本机资源指标 |

中心服务器若也要监控自身，同样运行 Agent。

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

第一次按页面提示创建账号。

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

**还有 D、E 数据盘时**，改为执行：

```powershell
$env:EXTRA_FILESYSTEMS = 'D:,E:'
.\beszel-agent.exe --key "这里替换为页面复制的完整公钥"
```

只填写实际存在的额外盘符。系统主卷也要到页面核验，不能假设一定识别正确。无盘符挂载卷、集群共享卷、SAN/RAID 映射需另行试验。

返回中心页面，数个采集周期后应显示在线和资源数据。其它服务器重复 Step 3～5。

采集连接由公钥认证并经 SSH 加密；Agent 不需要数据库账号或域管理员密码。

## 怎么确认已经成功

只检查这四件事：

- 主机在线，CPU 和内存数据持续更新。
- 每个需要监控的系统盘、数据盘都出现。
- 容量与本机磁盘属性大致一致；GB/GiB 和不同采样时刻可能产生差别。
- 试点运行覆盖一个业务高峰，任务管理器中 Agent 占用可接受。

磁盘**空间占用率**和磁盘**活动时间/繁忙程度**是不同指标。读写速率图不代表所有 Windows PerfMon 指标都有；本文主要验收 CPU、内存与磁盘容量。

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

核验范围为发布资产、文件内容/哈希及源码参数；没有在目标 Windows Server 2019 执行上述命令，现场兼容性、权限、真实占用和指标准确性仍需试点确认。本教程不是部署完成报告。

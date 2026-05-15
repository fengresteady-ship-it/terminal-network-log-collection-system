# 网络慧眼 — 端网日志采集系统2.0

## 系统简介

网络慧眼是一款面向 Windows 端点的网络日志采集与分析系统，旨在为安全测试、威胁检测和应急响应提供完整的主机行为记录和网络流量取证能力。

系统通过集成 **Sysmon**（系统内部事件监控）和 **tshark**（网络流量抓包）两大核心能力，实现对端点主机进程创建、网络连接、文件操作、注册表修改等行为的全面记录，同时捕获对应的网络数据包，形成"主机行为 + 网络流量"的双维度日志采集。

本系统提供直观的 Web 管理界面和完整的 REST API，支持本机操作和局域网远程控制，可单独使用，也可与入侵与攻击模拟（BAS）平台联动实现自动化测试闭环。

### 核心能力

- **双维度采集** — 同时采集 Sysmon 主机事件日志和网络流量数据包，时间对齐，交叉印证
- **独立控制** — 日志采集与流量抓包可独立启停，按需组合
- **实时展示** — 数据包实时流式显示，事件统计自动更新
- **局域网暴露** — 一键获取局域网地址，支持远程 API 调用
- **自动化就绪** — 提供任务（Mission）API，可与自动化脚本集成，实现批量场景测试

### 技术架构

```
┌─────────────────────────────────────────────┐
│              Web 管理界面 (:9880)             │
├──────────────┬──────────────┬───────────────┤
│  数据包捕获   │  Sysmon 日志  │   系统监控     │
│  (tshark)    │  (PowerShell) │  (WMI)        │
├──────────────┴──────────────┴───────────────┤
│               REST API 层                    │
├─────────────────────────────────────────────┤
│            Go HTTP Server (单文件部署)         │
└─────────────────────────────────────────────┘
```

- 后端采用 Go 语言开发，前端嵌入 HTML/JS，编译为单个 exe，无需安装
- 数据包通过 SSE（Server-Sent Events）实时推送到浏览器
- Sysmon XML 日志自动后台解析为 JSON 格式，方便后续分析

---

## 核心组件介绍

### Sysmon（System Monitor）

Sysmon 是微软 Sysinternals 出品的 Windows 系统高级监视工具，作为内核级驱动运行，能够记录详细的系统行为事件到 Windows 事件日志。它是端点检测与响应（EDR）领域广泛使用的基础工具。

**主要监控能力：**

| EventID | 事件类型 | 说明 |
|---------|---------|------|
| 1 | 进程创建 | 记录命令行、父进程、哈希值等 |
| 3 | 网络连接 | 记录源/目标 IP、端口、协议 |
| 7 | 模块加载 | DLL 加载事件 |
| 11 | 文件创建 | 文件写入和创建事件 |
| 12/13 | 注册表操作 | 注册表键的创建、删除和修改 |
| 23 | 文件删除 | 文件删除事件 |

在本系统中，Sysmon 负责记录主机层面的所有行为轨迹，为安全分析提供完整的进程链、网络连接和文件操作证据。

### Wireshark / tshark

Wireshark 是全球最广泛使用的网络协议分析工具，其命令行版本 **tshark** 提供强大的流量抓取和解析能力。配合 Npcap 驱动，tshark 能够捕获网卡上的所有网络数据包。

**在本系统中的作用：**

- 按指定网卡和BPF过滤器捕获网络流量
- 保存为标准 pcap 格式，可被 Wireshark、tcpdump 等工具打开分析
- 实时解析数据包摘要并通过 SSE 推送到 Web 界面
- 与 Sysmon 日志配合，提供网络维度的取证数据

---
## 快速开始

以管理员身份运行 `sysmon-manager.exe`，自动打开浏览器访问 `http://localhost:9880`。

- 本机访问：`http://localhost:9880`
- 局域网访问：点击网页左上角「局域网」按钮复制地址

## 前置依赖

- **Sysmon** — 内核级事件日志驱动，可通过网页一键安装
- **tshark (Wireshark)** — 网络抓包工具（含 Npcap 驱动），可通过网页一键安装

## Web 界面

### 数据包捕获

- ▶ **开始抓包** — 启动 tshark，实时显示数据包流
- ■ **停止抓包** — 停止 tshark，保存 pcap 文件
- 支持 BPF 过滤器（如 `tcp`、`port 443`）、网卡选择、快速过滤预设（TCP/UDP/HTTP/DNS/ICMP）

### Sysmon 日志采集

- ▶ **开始采集日志** — 记录起始时间戳
- ■ **结束采集日志** — 导出时间范围内的 Sysmon XML 日志
- 与抓包完全独立，互不影响
- 按钮状态互斥：开始后只能点结束，结束后才能点开始

### 事件统计

自动解析 Sysmon XML 日志，按 EventID 分类展示进程创建、网络连接、文件操作等事件统计。

### 系统信息

实时显示 CPU、内存、磁盘使用率，以及 Sysmon 和 tshark 的安装状态。

## API 接口

### 抓包

```
POST /api/capture/start    {"iface": "8", "filter": "tcp"}
POST /api/capture/stop
GET  /api/capture/interfaces    # 网卡列表
GET  /api/capture/stream        # SSE 实时数据包流
```

### 日志采集

```
POST /api/log/start        # 开始采集（仅记录时间戳）
POST /api/log/stop         # 结束采集（导出 Sysmon XML）
```

### 任务（抓包+日志同时）

```
POST /api/mission/start    {"iface": "8", "filter": "tcp"}
POST /api/mission/stop     # 同时停止抓包 + 导出 Sysmon XML
```

### 系统管理

```
GET  /api/status           # Sysmon 安装状态
GET  /api/sysinfo          # 系统信息（CPU/内存/磁盘）
GET  /api/dashboard        # 事件统计数据
GET  /api/lanip            # 局域网 IP 地址
POST /api/install          # 一键安装 Sysmon
POST /api/capture/install  # 一键安装 Wireshark
POST /api/shutdown         # 关闭服务
```

## 数据存储

| 目录 | 内容 |
|------|------|
| `sysmon/pcap/` | tshark 抓包文件（`.pcap`，按时间戳命名） |
| `sysmon/xml/` | Sysmon 导出的 XML 日志 |
| `sysmon/xml_json/` | 自动解析的 JSON 格式事件数据（每 30 秒扫描） |

## 常见用法

**手动操作：** 选择网卡 → 开始抓包 → 开始采集日志 → 执行测试 → 结束采集日志 → 停止抓包

**API 远程控制：**
```bash
curl -X POST http://10.0.0.1:9880/api/mission/start \
  -H "Content-Type: application/json" \
  -d '{"iface":"8"}'
# 执行测试...
curl -X POST http://10.0.0.1:9880/api/mission/stop
```

**独立控制：** `/api/capture/*` 只管抓包，`/api/log/*` 只管 Sysmon 日志，互不干扰。

## 交流
fengresteady@gmail.com
中关村实验室 APT团队

## License
[BSD 3](https://github.com/fengresteady-ship-it/terminal-network-log-collection-system/blob/main/LICENSE)

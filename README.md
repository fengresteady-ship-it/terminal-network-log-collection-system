# 网络慧眼 — 端网日志采集系统2.0

基于 Sysmon + tshark 的 Windows 端点网络日志采集平台，提供 Web 管理界面和 REST API，支持数据包抓取、Sysmon 事件日志采集与实时监控。

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
[BSD 3]<#https://github.com/fengresteady-ship-it/terminal-network-log-collection-system/blob/main/LICENSE>

---
layout: post
title: "Scenario Map: 集群存储 CSV 与 S2D 排查导航 — Cluster Storage Troubleshooting"
date: 2026-03-17
categories: [Scenario, Storage]
tags: [storage, csv, s2d, cluster, troubleshooting, disk-arbitration, scsi3-pr, redirected-io, storage-spaces-direct, cache, pool-degraded]
type: "scenario-map"
---

# Scenario Map: 集群存储 CSV 与 S2D 排查导航

**Topic:** Cluster Storage, CSV & S2D Troubleshooting  
**Category:** Storage / Failover Clustering  
**Last Updated:** 2026-03-17

---

## 总览思维导图

```mermaid
mindmap
  root((集群存储排查))
    磁盘仲裁
      SCSI-3 PR 丢失
      脑裂
      故障转移时磁盘离线
    CSV 暂停
      Paused Direct I/O
      Paused Redirected I/O
      Paused All I/O
    Redirected I/O 性能
      VM 卡顿
      网络饱和
      无法恢复 Direct I/O
    CSV 文件系统损坏
      Spot-fixing
      ChkDsk 离线修复
    S2D 池降级
      物理磁盘故障
      虚拟磁盘不健康
      修复作业卡住
    S2D 缓存驱动器故障
      写入丢失
      容量盘不健康
      自动恢复
    S2D 网络问题
      RDMA 丢失
      SMB Direct 失败
      集群网络分区
    Storage Spaces
      虚拟磁盘 Detached
      池元数据损坏
```

---

## 场景 A: 磁盘仲裁失败

```mermaid
flowchart TD
    A[集群磁盘离线] --> B{Event Log 检查}
    B -->|Event 1034/1035 ClusDisk| C[SCSI-3 PR 预留失败]
    C --> D{所有节点都能看到磁盘?}
    D -->|否| D1["检查 SAN 路径:\n- FC Zoning\n- iSCSI Target 配置\n- MPIO 路径状态"]
    D -->|是| E{存储是否支持 SCSI-3 PR?}
    E -->|否| E1["存储不支持 PR!\n联系存储厂商\n或改用 iSCSI Target Server"]
    E -->|是| F["检查:\n1. 固件版本\n2. 多路径 DSM 兼容性\n3. SAN 交换机配置\n4. 运行 Cluster Validation"]
```

### 关键事件 ID

| Event ID | 来源 | 含义 |
|----------|------|------|
| **1034** | ClusDisk | 磁盘仲裁：无法获取 SCSI-3 预留 |
| **1035** | ClusDisk | 磁盘仲裁：预留被其他节点持有 |
| **1069** | FailoverClustering | 集群资源上线失败 |
| **1205** | ClusDisk | 集群磁盘资源离线 |

---

## 场景 B: CSV 暂停状态

```mermaid
flowchart TD
    A["CSV 状态异常\nGet-ClusterSharedVolume"] --> B{CSV 状态?}
    B -->|"Paused (Direct I/O)"| C["节点可直接访问磁盘\n但元数据通道异常\n检查到 Coordinator 的 SMB 连接"]
    B -->|"Paused (Redirected I/O)"| D["节点无法直接访问磁盘\nI/O 通过网络重定向\n检查 HBA/SAN 路径"]
    B -->|"Paused (All I/O)"| E["严重! 所有 I/O 暂停\n检查磁盘健康\n检查 Storage Spaces 状态\n可能需要手动恢复"]
    C --> C1["Resume-ClusterSharedVolume\n或移动 Coordinator 所有权"]
    D --> D1["修复底层存储路径\n路径恢复后自动回到 Direct I/O"]
    E --> E1["Get-VirtualDisk 检查健康\nGet-PhysicalDisk 检查磁盘\n可能需要修复 Storage Spaces"]
```

### CSV 状态参考表

| 状态 | 含义 | 影响 | 常见原因 |
|------|------|------|---------|
| **Online** | 正常 | 无 | — |
| **Paused (Direct I/O)** | 直接 I/O 正常，元数据通道异常 | 低 | SMB 连接问题 |
| **Paused (Redirected I/O)** | 所有 I/O 通过网络重定向 | **高（性能下降）** | HBA/SAN 路径丢失 |
| **Paused (All I/O)** | 所有 I/O 暂停 | **严重** | 磁盘/Storage Spaces 故障 |

### 诊断命令

```powershell
Get-ClusterSharedVolume                    # CSV 总览
Get-ClusterSharedVolumeState               # 每个节点的 I/O 状态
Get-ClusterSharedVolume | fl *             # 详细信息
```

---

## 场景 C: Redirected I/O 导致性能问题

> **类比**：Redirected I/O 就像交通改道——本来走高速直达（Direct I/O），现在被迫走狭窄的城市道路（网络），自然堵车。

```mermaid
flowchart TD
    A["VM 运行缓慢\n网络带宽饱和"] --> B["检查 CSV I/O 状态\nGet-ClusterSharedVolumeState"]
    B --> C{是否在 Redirected I/O?}
    C -->|否| C1[非 CSV 重定向问题\n排查其他性能瓶颈]
    C -->|是| D{为什么进入 Redirected?}
    D -->|节点无法看到磁盘| E["检查 HBA/SAN 路径\nmpclaim -s -d\nGet-PhysicalDisk"]
    D -->|Direct I/O 曾失败| F["检查 CSV Event Log\n5120/5121 事件\n可能是临时性 I/O 错误"]
    D -->|镜像 Storage Space on CSV| G["设计如此!\n镜像 SS 在 CSV 上\n始终使用 Redirected I/O"]
    E --> E1["修复路径后:\nResume-ClusterSharedVolume\n或移动 CSV 所有权"]
    F --> F1["解决底层 I/O 错误\n自动恢复到 Direct I/O"]
```

---

## 场景 D: CSV 文件系统损坏

```mermaid
flowchart TD
    A[CSV 文件系统损坏] --> B["NTFS IsAlive 检测到异常\nEvent 5120"]
    B --> C{损坏类型?}
    C -->|临时性| D["Spot-fixing 自动修复\n15秒一轮\n2-3分钟间隔重复\n卷保持在线!"]
    C -->|持久性| E["记录修复操作\n通知管理员\n卷仍然在线"]
    E --> F["维护窗口执行:\n卷短暂离线(秒级)\nCSV I/O 透明暂停\n修复完成后自动恢复"]
    D -->|修复成功| D1["Event 98: NTFS 健康"]
    D -->|修复失败| E
```

---

## 场景 E: S2D 存储池降级

```mermaid
flowchart TD
    A["S2D 池状态异常"] --> B["Get-StoragePool\nGet-VirtualDisk\nGet-PhysicalDisk"]
    B --> C{PhysicalDisk 状态?}
    C -->|Lost Communication| D["磁盘连接丢失\n检查物理连接\n检查 SBL 状态"]
    C -->|Unhealthy| E["磁盘故障\nS2D 自动启动修复\nGet-StorageJob 查看进度"]
    C -->|Removing| F["磁盘正在被移除\n等待数据迁移完成"]
    C -->|Missing| G["磁盘消失\n检查服务器/磁盘是否下线"]
    E --> H{VirtualDisk HealthStatus?}
    H -->|Warning| I["降级但可用\n修复进行中\n监控 Get-StorageJob"]
    H -->|Unhealthy| J["数据可能有风险!\n检查剩余磁盘数是否满足弹性\nMirror 需要至少 N-1 磁盘正常"]
```

### S2D 健康状态矩阵

| 组件 | 命令 | 健康 | 降级 | 故障 |
|------|------|:----:|:----:|:----:|
| 物理磁盘 | `Get-PhysicalDisk` | OK | Degraded | Lost Communication / Unhealthy |
| 虚拟磁盘 | `Get-VirtualDisk` | Healthy | Warning | Unhealthy |
| 存储池 | `Get-StoragePool` | OK | Degraded | Unknown |
| 子系统 | `Get-StorageSubSystem` | OK | - | - |
| 修复作业 | `Get-StorageJob` | — | Running | Suspended |

---

## 场景 F: S2D 缓存驱动器故障

```mermaid
flowchart TD
    A[缓存驱动器故障] --> B["未 de-stage 的写入丢失(本地)\n但存在于其他服务器的副本中"]
    B --> C["绑定的容量盘\n短暂显示 Unhealthy"]
    C --> D["自动缓存重绑定\n(Cache Rebinding)"]
    D --> E["自动数据修复\n从其他服务器的副本恢复"]
    E --> F["容量盘恢复 Healthy"]
    F --> G["更换故障的缓存驱动器"]
    
    H["⚠️ 所有缓存盘故障"] --> I["整个服务器性能严重下降\n但数据安全(其他服务器有副本)"]
    I --> J["尽快更换缓存驱动器\n最低要求: 每服务器 2 块缓存盘"]
```

---

## 场景 G: S2D 网络问题

```mermaid
flowchart TD
    A[S2D 性能骤降] --> B{RDMA 状态?}
    B -->|"Get-NetAdapterRdma\n显示 Disabled/Error"| C["RDMA 丢失!\n回退到 TCP = 性能断崖\n检查 NIC 驱动/固件\n检查交换机 DCB/PFC 配置"]
    B -->|RDMA 正常| D{SMB 连接状态?}
    D -->|"Get-SmbConnection\n连接异常"| E["SMB Direct 失败\n检查 SMB 多通道配置\n检查防火墙规则"]
    D -->|连接正常| F["非网络问题\n检查磁盘/CPU/内存"]
    C --> G["修复 RDMA 后\nSBL 自动重建连接\n性能恢复"]
```

---

## 场景 H: Storage Spaces 虚拟磁盘 Detached

```mermaid
flowchart TD
    A["Get-VirtualDisk\nOperationalStatus: Detached"] --> B{原因?}
    B -->|健康磁盘不足| C["弹性要求未满足\n检查物理磁盘数量\n修复/更换故障磁盘"]
    B -->|池元数据损坏| D["尝试:\nRepair-VirtualDisk\nSet-VirtualDisk -IsManualAttach $false"]
    B -->|节点重启后未自动挂载| E["Set-VirtualDisk -IsManualAttach $false\n或手动: Connect-VirtualDisk"]
    C --> F["磁盘修复后:\nGet-StorageJob 监控修复\n修复完成后 VD 自动 Attach"]
    D -->|失败| G["最后手段:\nDebug-StorageSubSystem\n联系 Microsoft 支持"]
```

---

## CSV I/O 路径决策树

```mermaid
flowchart TD
    A["I/O 请求到达 CSV"] --> B{是元数据操作?}
    B -->|"是 (Create/Delete/Rename)"| C["通过 SMB 转发到\nCoordinator 节点的 NTFS"]
    B -->|"否 (Read/Write)"| D{当前节点是 Coordinator?}
    D -->|是| E["Direct I/O\n直接访问磁盘"]
    D -->|否| F{节点能直接看到磁盘?}
    F -->|是| G["Direct I/O\n直接访问磁盘"]
    F -->|否| H["Block Redirected I/O\n通过网络到能访问磁盘的节点"]
    G -->|"I/O 失败"| H
    
    style E fill:#2d6,stroke:#333
    style G fill:#2d6,stroke:#333
    style H fill:#f96,stroke:#333
    style C fill:#69f,stroke:#333
```

---

## 诊断工具总表

| 工具 | 用途 |
|------|------|
| `Get-ClusterSharedVolume` | CSV 状态总览 |
| `Get-ClusterSharedVolumeState` | 每节点 CSV I/O 状态 |
| `Get-PhysicalDisk` | S2D 物理磁盘健康 |
| `Get-VirtualDisk` | S2D 虚拟磁盘健康 |
| `Get-StoragePool` | 池健康与容量 |
| `Get-StorageJob` | 活跃修复作业 |
| `Get-StorageSubSystem` | S2D 整体健康 |
| `Debug-StorageSubSystem` | S2D 深度诊断 |
| `Get-ClusterLog` | 集群诊断日志 |
| `Get-SmbConnection` | SMB 连接状态 |
| `Get-NetAdapterRdma` | RDMA NIC 状态 |

---

---

# English Version

---

# Scenario Map: Cluster Storage, CSV & S2D Troubleshooting

**Last Updated:** 2026-03-17

---

## Scenario A: Disk Arbitration Failures

| Event ID | Source | Meaning |
|----------|--------|---------|
| **1034** | ClusDisk | Cannot acquire SCSI-3 reservation |
| **1035** | ClusDisk | Reservation held by another node |
| **1069** | FailoverClustering | Cluster resource failed to come online |
| **1205** | ClusDisk | Cluster disk resource went offline |

**Troubleshooting**: Check SAN paths → Verify SCSI-3 PR support → Check firmware/DSM → Run Cluster Validation

---

## Scenario B: CSV Paused States

| State | Impact | Common Cause | Action |
|-------|--------|-------------|--------|
| **Online** | None | — | — |
| **Paused (Direct I/O)** | Low | SMB connection issue | Resume or move coordinator |
| **Paused (Redirected I/O)** | **High** | HBA/SAN path lost | Fix storage paths |
| **Paused (All I/O)** | **Critical** | Disk/Storage Spaces failure | Check `Get-VirtualDisk` health |

## Scenario C: Redirected I/O Performance

> **Analogy**: Redirected I/O is like a traffic detour — instead of the highway (Direct I/O), you're forced through narrow city streets (network).

Check: `Get-ClusterSharedVolumeState` → If FileSystemRedirectedIOReason ≠ None → Fix underlying storage path

## Scenario D: S2D Pool Degraded

```powershell
Get-PhysicalDisk | Where OperationalStatus -ne OK    # Find problem disks
Get-VirtualDisk | Where HealthStatus -ne Healthy      # Find affected VDs  
Get-StorageJob                                        # Monitor repair progress
```

## Scenario E: S2D Cache Drive Failure
- Lost writes exist on other servers (safe if mirror/parity)
- Capacity drives temporarily unhealthy → auto cache rebinding → auto data repair → healthy
- **Minimum 2 cache drives per server** required

## Scenario F: CSV I/O Decision Tree

```
I/O Request → Metadata? → Yes → Coordinator via SMB
                       → No → Is coordinator? → Yes → Direct I/O
                                              → No → Can see disk? → Yes → Direct I/O
                                                                   → No → Block Redirected I/O
```

## Diagnostic Tools

| Tool | Purpose |
|------|---------|
| `Get-ClusterSharedVolume` | CSV status |
| `Get-ClusterSharedVolumeState` | CSV I/O state per node |
| `Get-PhysicalDisk` | S2D physical disk health |
| `Get-VirtualDisk` | S2D virtual disk health |
| `Get-StoragePool` | Pool health and capacity |
| `Get-StorageJob` | Active repair jobs |
| `Debug-StorageSubSystem` | S2D deep diagnostics |
| `Get-ClusterLog` | Cluster diagnostic log |
| `Get-NetAdapterRdma` | RDMA NIC status |

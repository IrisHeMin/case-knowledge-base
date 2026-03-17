---
layout: post
title: "Scenario Map: Windows 存储栈排查导航 — Storage Stack Troubleshooting"
date: 2026-03-17
categories: [Scenario, Storage]
tags: [storage, troubleshooting, storport, disk, partition, ntfs, refs, mpio, iscsi, performance, corruption]
type: "scenario-map"
---

# Scenario Map: Windows 存储栈排查导航

**Topic:** Windows Storage Stack Troubleshooting  
**Category:** Storage  
**Last Updated:** 2026-03-17

---

## 总览思维导图 / Overview Mindmap

```mermaid
mindmap
  root((存储栈排查))
    磁盘不可见
      设备管理器检查
      驱动加载问题
      iSCSI 连接
      SAN/FC 连通性
      磁盘策略(离线/只读)
    分区/卷问题
      MBR 损坏
      GPT 保护性 MBR
      动态磁盘外来
      卷无法挂载
    文件系统损坏
      NTFS chkdsk
      ReFS 自修复
      CSVFS spot-fixing
    iSCSI 连通性
      目标未发现
      会话断开
      持久连接重连
    MPIO 路径故障
      所有路径失败
      单路径失败
      DSM 策略问题
    存储性能
      PerfMon 计数器
      StorPort ETL
      SSD SMART
    BitLocker
      恢复密钥丢失
      TPM 变更
    重删问题
      作业失败
      Chunk Store 损坏
```

---

## 场景 A: 磁盘在 Disk Management 中不可见

```mermaid
flowchart TD
    A[磁盘不可见] --> B{设备管理器有未知设备?}
    B -->|是| C[安装/更新驱动]
    B -->|否| D{连接类型?}
    D -->|iSCSI| E{iSCSI Initiator 已连接?}
    E -->|否| E1[检查 Target IP/IQN/认证]
    E -->|是| E2[检查 Target 端 LUN 分配]
    D -->|FC/SAN| F{HBA 可见?}
    F -->|否| F1[检查 HBA 驱动/固件]
    F -->|是| F2[检查 Zoning/LUN Masking]
    D -->|本地| G{Disk Management 中显示?}
    G -->|离线| G1["右键 → Online\n检查 SAN Policy"]
    G -->|只读| G2["diskpart → attributes disk clear readonly"]
    G -->|外来| G3["Import Foreign Disks"]
    G -->|不显示| G4[检查物理连接/BIOS]
```

### 排查要点

| 检查项 | 命令/工具 | 说明 |
|--------|----------|------|
| 磁盘列表 | `Get-Disk` / `diskpart → list disk` | 查看所有磁盘状态 |
| SAN 策略 | `diskpart → san` | Offline Shared / Online All |
| 驱动状态 | 设备管理器 | 黄色感叹号 = 驱动问题 |
| iSCSI 会话 | `iscsicli SessionList` | 查看活跃 iSCSI 会话 |
| MPIO 路径 | `mpclaim -s -d` | 查看多路径磁盘和路径状态 |

---

## 场景 B: 分区/卷问题

```mermaid
flowchart TD
    A[分区/卷问题] --> B{具体症状?}
    B -->|MBR 损坏无法启动| C["bootrec /fixmbr\nbootrec /fixboot\nbootrec /rebuildbcd"]
    B -->|GPT 磁盘显示为 Protective MBR| D["不要在 MBR-only 系统上操作 GPT 磁盘\n确认 UEFI 启动模式"]
    B -->|动态磁盘显示为 Foreign| E["Disk Management → Import Foreign\n或 diskpart → import"]
    B -->|卷无法挂载| F{有驱动器号?}
    F -->|否| F1["diskpart → assign letter=X"]
    F -->|有但无法访问| F2["检查文件系统: chkdsk /f\n检查权限"]
    B -->|分区表损坏| G["TestDisk 等第三方工具恢复\n或从备份还原"]
```

---

## 场景 C: 文件系统损坏

```mermaid
flowchart TD
    A[文件系统损坏] --> B{文件系统类型?}
    B -->|NTFS| C{Event ID 55?}
    C -->|是| D["chkdsk /f (修复元数据)\nchkdsk /r (修复 + 坏扇区扫描)\nchkdsk /b (重评估坏簇, 仅 NTFS)"]
    C -->|Event 98| D2[NTFS 已自修复, 仅信息性]
    B -->|ReFS| E[ReFS 自动修复: integrity streams\n检查 Get-FileIntegrity]
    E --> E1{修复失败?}
    E1 -->|是| E2["配合 Storage Spaces Mirror\n从冗余副本恢复"]
    B -->|CSVFS| F["CSV spot-fixing 流程:\n1. IsAlive 检测异常\n2. 15秒 spot-fix 尝试\n3. 2-3分钟间隔重复\n4. 失败则维护窗口离线修复"]
```

### 关键事件 ID

| Event ID | 来源 | 含义 |
|----------|------|------|
| **55** | Ntfs | NTFS 检测到文件系统损坏 |
| **98** | Ntfs | NTFS 健康检查完成（通常是自修复后） |
| **137** | Ntfs | NTFS 遇到 I/O 错误 |
| **5120** | CsvFs | CSV 文件系统错误 |
| **5121** | CsvFs | CSV 卷进入 redirected mode |
| **5142** | CsvFs | CSV 卷 I/O 暂停 |

---

## 场景 D: iSCSI 连通性

```mermaid
flowchart TD
    A[iSCSI 问题] --> B{具体症状?}
    B -->|Target 未发现| C[检查 Discovery Portal IP:Port\n检查防火墙 TCP 3260\n检查 iSNS 服务]
    B -->|会话频繁断开| D[检查网络稳定性\n增大 Timeout 值\n检查 NIC 电源管理]
    B -->|持久连接不重连| E["检查 Persistent Target 配置\niscsicli PersistentLoginTarget\n检查 iSCSI 服务启动类型=自动"]
    B -->|性能差| F[检查是否使用专用网络\n启用 Jumbo Frames\n检查 QoS 优先级]
```

---

## 场景 E: MPIO 路径故障

```mermaid
flowchart TD
    A[MPIO 路径故障] --> B{故障范围?}
    B -->|所有路径失败| C["磁盘离线!\n检查所有 HBA 状态\n检查 SAN 交换机\n检查 LUN 分配"]
    B -->|单路径失败| D["磁盘降级但可用\n检查失败路径的 HBA/光纤\nmpclaim -s -d 查看路径状态"]
    B -->|负载不均衡| E["检查 DSM 策略\nGet-MPIOSetting\n确认 Round Robin 或正确策略"]
    C --> C1[Event ID 1/2/3/21 MPIO]
    D --> D1[修复物理路径后自动恢复]
```

---

## 场景 F: 存储性能问题

```mermaid
flowchart TD
    A[存储性能问题] --> B{延迟在哪一层?}
    B -->|应用层感知慢| C["PerfMon: Avg. Disk sec/Read\nAvg. Disk sec/Write\n> 20ms = 有问题"]
    C --> D{队列深度?}
    D -->|Queue Length 高| E[磁盘忙: IOPS 不足\n检查 RAID 配置\n考虑 SSD/NVMe 升级]
    D -->|Queue Length 正常| F[非磁盘瓶颈\n检查 CPU/内存/网络]
    B -->|驱动层延迟| G["StorPort ETL Tracing\n精确定位 port driver 层延迟"]
    B -->|SSD 寿命问题| H["检查 SMART 数据\nGet-PhysicalDisk \| Get-StorageReliabilityCounter\n关注: Wear, Temperature, MediaErrors"]
```

### 关键 PerfMon 计数器

| 计数器 | 正常值 | 告警阈值 | 说明 |
|--------|--------|---------|------|
| Avg. Disk sec/Read | < 10ms | > 20ms | 平均读延迟 |
| Avg. Disk sec/Write | < 10ms | > 20ms | 平均写延迟 |
| Current Disk Queue Length | < 2 | > 5 | 当前等待队列 |
| Disk Transfers/sec | - | - | IOPS |
| Disk Bytes/sec | - | - | 吞吐量 |

---

## 场景 G/H: BitLocker 与重删问题

### BitLocker

| 问题 | 解决方案 |
|------|---------|
| 恢复密钥丢失 | 检查 Microsoft 账户、Azure AD、AD、USB、打印件 |
| TPM 芯片更换 | 使用恢复密钥解锁 → 重新启用 BitLocker |
| 硬件变更后无法启动 | 进入 WinRE → 输入恢复密钥 |
| 集群 CSV 上的 BitLocker | 使用 SID-based protector (CNO 对象) |

### Data Deduplication

| 问题 | 排查 |
|------|------|
| 优化作业失败 | `Get-DedupStatus`、检查 Event Log Deduplication |
| Chunk Store 损坏 | `Start-DedupJob -Type Scrubbing` |
| 空间未回收 | 检查文件是否满足条件（>64KiB, 非加密, 非系统卷）|
| 高 CPU 使用 | 调整 Schedule: `Set-DedupSchedule` |

---

## 诊断工具总表 / Diagnostic Tools

| 工具 | 用途 | 栈层级 |
|------|------|--------|
| `diskpart` | 磁盘/分区管理 | Partition Manager |
| `chkdsk` | 文件系统修复 | File System |
| `fsutil` | 文件系统高级查询 | File System |
| StorPort ETL | I/O 延迟精确诊断 | Port Driver |
| `mpclaim -s -d` | MPIO 路径管理 | MPIO |
| `iscsicli` | iSCSI 管理 | iSCSI |
| `Get-PhysicalDisk` | 物理磁盘健康 | Storage Spaces |
| PerfMon | 性能计数器 | 全栈 |
| Event Viewer | 事件日志 | 全栈 |

---

---

# English Version

---

# Scenario Map: Windows Storage Stack Troubleshooting

**Topic:** Windows Storage Stack Troubleshooting  
**Last Updated:** 2026-03-17

---

## Scenario A: Disk Not Visible

```mermaid
flowchart TD
    A[Disk Not Visible] --> B{Unknown device in Device Manager?}
    B -->|Yes| C[Install/update driver]
    B -->|No| D{Connection type?}
    D -->|iSCSI| E{Initiator connected?}
    E -->|No| E1[Check Target IP/IQN/auth]
    E -->|Yes| E2[Check LUN assignment on Target]
    D -->|FC/SAN| F{HBA visible?}
    F -->|No| F1[Check HBA driver/firmware]
    F -->|Yes| F2[Check Zoning/LUN Masking]
    D -->|Local| G{Status in Disk Management?}
    G -->|Offline| G1["Right-click → Online\nCheck SAN Policy"]
    G -->|Read-only| G2["diskpart → attributes disk clear readonly"]
    G -->|Foreign| G3[Import Foreign Disks]
    G -->|Not shown| G4[Check physical connection/BIOS]
```

## Scenario B: File System Corruption

| File System | Repair Method | Key Event IDs |
|-------------|---------------|--------------|
| **NTFS** | `chkdsk /f` (metadata), `/r` (+ bad sectors), `/b` (re-evaluate bad clusters) | 55 (corruption), 98 (healthy) |
| **ReFS** | Automatic via integrity streams + Storage Spaces mirror | - |
| **CSVFS** | Spot-fixing: 15s rounds → 2-3 min intervals → maintenance offline | 5120, 5121, 5142 |

## Scenario C: iSCSI Issues

| Issue | Resolution |
|-------|-----------|
| Target not discovered | Check Discovery Portal IP:3260, firewall, iSNS |
| Session drops | Check network stability, increase timeout, check NIC power management |
| Persistent not reconnecting | Verify persistent login config, iSCSI service startup = Automatic |

## Scenario D: MPIO Path Failures

| Scope | Impact | Action |
|-------|--------|--------|
| All paths failed | Disk offline | Check all HBAs, SAN switches, LUN assignment |
| Single path failed | Degraded, functional | Repair physical path, auto-recovery |
| Load imbalance | Performance impact | Check DSM policy (`Get-MPIOSetting`) |

## Scenario E: Storage Performance

**Decision tree**: Latency high? → PerfMon (`Avg Disk sec/Read > 20ms`?) → Queue Length high? → Disk bottleneck → Check RAID/upgrade to SSD. Queue normal? → Not disk issue.

**Key counters**: Avg Disk sec/Read, Avg Disk sec/Write (normal < 10ms, alert > 20ms), Current Disk Queue Length (alert > 5)

## Diagnostic Tools

| Tool | Purpose | Layer |
|------|---------|-------|
| `diskpart` | Disk/partition management | Partition |
| `chkdsk` | File system repair | File System |
| StorPort ETL | I/O latency at driver level | Port Driver |
| `mpclaim -s -d` | MPIO path management | MPIO |
| `iscsicli` | iSCSI management | iSCSI |
| `Get-PhysicalDisk` | Storage Spaces disk health | Storage Spaces |
| PerfMon | Performance counters | All layers |

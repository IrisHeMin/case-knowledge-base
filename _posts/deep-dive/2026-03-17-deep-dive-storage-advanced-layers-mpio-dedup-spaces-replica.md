---
layout: post
title: "Deep Dive: 存储高级层 — MPIO、重删、存储空间与副本"
date: 2026-03-17
categories: [Knowledge, Storage]
tags: [storage, mpio, bitlocker, deduplication, storage-spaces, storage-replica, storage-migration, spaceport, fvevol]
type: "deep-dive"
---

# Deep Dive: 存储高级层 — MPIO、重删、存储空间与副本

**Topic:** Windows Storage Advanced Layers  
**Category:** Storage  
**Level:** 中高级 / Senior-Expert  
**Last Updated:** 2026-03-17

---

## 1. 概述

在[简化存储栈](../deep-dive-windows-storage-stack-architecture/)的基础上，Windows 还提供了多个**可选层**，用于多路径冗余、加密、数据优化、存储虚拟化和复制。这些层像"插件"一样插入存储栈的不同位置。

> **类比**：如果简化存储栈是一栋普通大楼，那么这些高级层就是可选的安保系统（BitLocker）、去重压缩机（Dedup）、多条电梯通道（MPIO）、楼层合并器（Storage Spaces）和异地备份管道（Storage Replica）。

---

## 2. MPIO — 多路径 I/O

### 2.1 为什么需要 MPIO

当服务器通过多条物理路径（多个 HBA、多条光纤、多个交换机）连接到同一个 SAN LUN 时，Windows 会把每条路径识别为一个独立的磁盘。这不仅浪费资源，还会造成混乱。

> **类比**：MPIO 就像 GPS 导航知道有多条路到同一个目的地。如果一条路堵了（路径故障），自动切换到另一条路。负载均衡 = 把车流分散到所有可用道路上。

### 2.2 架构

```
                    ┌─────────────┐
                    │   应用层     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  文件系统    │
                    ├─────────────┤
                    │  卷管理器    │
                    ├─────────────┤
                    │  分区管理器   │
                    ├─────────────┤
                    │  MPDev.sys   │  ← 替代 disk.sys
                    │  或 ClassPnP │
                    ├─────────────┤
                    │  MPIO.sys    │  ← 聚合所有路径
                    ├─────────────┤
                    │   DSM        │  ← 设备特定模块
                    ├─────┬───────┤
                    │     │       │
               ┌────▼─┐ ┌▼────┐ ┌▼────┐
               │Path 1│ │Path2│ │Path3│  ← 多条物理路径
               │Port  │ │Port │ │Port │
               └──────┘ └─────┘ └─────┘
```

**关键组件**：
- **MPIO.sys**：聚合所有指向同一 LUN 的路径，呈现一个**伪 LUN (Pseudo-LUN)** 给上层
- **DSM (Device Specific Module)**：设备特定模块，决定负载均衡策略。可以有多个 DSM 共存
- **MPDev.sys / ClassPnP.sys**：替代标准的 disk.sys 作为类驱动

### 2.3 Microsoft DSM 负载均衡策略

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| **Failover** | 使用主路径，故障时切换到备用路径 | 简单的主备冗余 |
| **Failback** | 故障恢复后自动切回主路径 | 需要优先使用特定路径 |
| **Round Robin** | 轮询所有可用路径 | 均匀分布 I/O 负载 |
| **Round Robin with Subset** | 仅在活动路径子集中轮询 | 保留部分路径作为备用 |
| **Weighted Path** | 按权重分配 I/O | 不同路径带宽不同时 |

### 2.4 诊断命令

```powershell
mpclaim -s -d           # 列出所有多路径磁盘及其路径
Get-MPIOSetting         # 查看 MPIO 全局设置
Get-MPathIOConfiguration # 查看路径配置
```

---

## 3. BitLocker — 磁盘加密

### 3.1 工作原理

> **类比**：BitLocker 就像给硬盘装了一个保险箱。即使有人偷走了物理磁盘，没有钥匙（TPM 芯片或恢复密钥）只能看到乱码。

**核心驱动**：`FVEVol.sys`，位于文件系统和 VSS 之间

```
文件系统 (NTFS/ReFS)
        │
    FVEVol.sys    ← BitLocker 加密/解密层
        │
  Volume Snapshot (VSS)
```

### 3.2 关键要点

- 最佳搭配 **TPM v1.2+**（无需 USB 启动密钥）
- 加密整个卷；离线时数据完全加密保护
- 启动时 FVEVol.sys 用 TPM 中的密钥解锁卷
- **32 位恢复密钥**是最后的救命稻草
- 支持 NTFS 和 ReFS
- ⚠️ **安全 vs 可修复性的取舍**：BitLocker 卷的数据恢复几乎不可能（除非有恢复密钥）

### 3.3 恢复密钥保存位置

- Microsoft 账户在线
- Azure AD / Entra ID
- USB 闪存驱动器
- 打印输出
- Active Directory（组策略推送）

---

## 4. 数据重删 (Data Deduplication)

### 4.1 概述

> **类比**：重删就像共享图书馆。不是每个人都买一本相同的书，而是图书馆保留一本，所有人去引用它。节省大量书架空间。

自 Windows Server 2012 引入，通过消除重复数据块来减少存储占用。

### 4.2 三种优化方式

1. **压缩 (Compression)**：减小数据体积
2. **文件级重删**：识别完全相同的文件
3. **块级重删 (Chunk-level)**：将文件切成小块（chunk），仅存储唯一块，重复块用引用替代

### 4.3 NTFS 文件内部结构（理解重删的前提）

```
文件记录 (MFT Record)
├── $STANDARD_INFORMATION  ← 创建时间、修改时间等
├── $FILE_NAME             ← 文件名
├── $ATTRIBUTE_LIST        ← 属性列表（有非驻留属性时才出现）
└── $DATA                  ← 数据流（主数据流 = unnamed data stream）
```

- **主数据流 (Primary Data Stream)**：`$DATA:""`，文件的实际内容
- **备选数据流 (ADS)**：附加的命名数据流
- 重删**仅处理主数据流**

### 4.4 空间节省效果

| 场景 | 内容类型 | 空间节省 |
|------|---------|---------|
| 用户文件共享 | Office 文档、照片、视频 | 30-50% |
| 部署共享 | 软件包、CAB、符号文件 | 70-80% |
| 虚拟化库 | ISO、VHD/VHDX 文件 | **80-95%** |
| 通用文件服务器 | 混合类型 | 50-60% |

### 4.5 限制与注意事项

- ❌ 仅 Server 版本
- ❌ 不支持：启动/系统卷、远程驱动器、CSV、加密文件、< 64KiB 的文件
- ❌ ReFS 直到 WS2019 才支持
- ⚠️ **风险放大**：一小块数据被大量文件引用，一旦损坏影响范围大
- ✅ **缓解措施**：校验和验证、元数据冗余副本、定期 Scrubbing 作业

### 4.6 栈集成

重删层位于文件系统之上。但并非所有 I/O 都经过重删层。

关键文件：`DDPSVC.dll`（服务）、`ddpchunk.dll`、`ddpstore.dll`、`ddpeval.exe`（评估工具）

---

## 5. Storage Spaces — 存储空间

### 5.1 概述

> **类比**：Storage Spaces 就像一个水库系统。不是每家每户都挖一口井（独立磁盘），而是把所有水源汇聚到一个水库（Pool），然后按需分配水（Virtual Disk）给各家。

自 WS2012 引入，是 Windows 的**软件定义存储虚拟化技术**。

### 5.2 核心概念

```
物理磁盘 (Physical Disks)
    ↓ 汇聚
存储池 (Storage Pool)      ← 由 SpacePort.sys 管理
    ↓ 划分
虚拟磁盘 (Virtual Disk / Storage Space)  ← 回到栈中的 disk.sys 层
    ↓ 分区 + 格式化
卷 (Volume)                ← 可以像普通磁盘一样使用
```

### 5.3 关键驱动与组件

| 组件 | 说明 |
|------|------|
| **SpacePort.sys** | 主端口驱动，PnP 通知，管理池和空间 |
| **smspace.dll** | 硬件提供程序 |
| **Spaces Protective Partition** | 每个池成员磁盘上的元数据分区 |
| **SpaceDump.sys** | WS2016+ 用于蓝屏时的 dump 操作 |

### 5.4 弹性类型

| 弹性类型 | 描述 | 容量效率 | 容错 |
|---------|------|---------|------|
| **Simple** | 数据条带化，无冗余 | 100% | ❌ 无 |
| **Mirror** | 数据复制 2-3 份 | 33-50% | ✅ 可容 1-2 盘故障 |
| **Parity** | 数据 + 奇偶校验条带化（类似 RAID5） | 67-87% | ✅ 可容 1-2 盘故障 |

### 5.5 栈集成位置

Storage Spaces 层位于**分区管理器之上**。虚拟磁盘创建后，回到栈中的**类驱动 (disk.sys)** 层，再经过分区、卷、文件系统。

### 5.6 注意：Virtual Disk ≠ Virtual Hard Disk

| 概念 | 含义 |
|------|------|
| **Virtual Disk (虚拟磁盘)** | = Storage Space，由 SpacePort.sys 创建 |
| **Virtual Hard Disk (虚拟硬盘)** | = .VHD/.VHDX 文件，由 Hyper-V 使用 |

> 两者完全不同！名字容易混淆。

---

## 6. Storage Replica — 存储副本

### 6.1 概述

> **类比**：同步复制 = 寄挂号信（你不继续操作，直到确认对方收到了）。异步复制 = 寄普通信（你继续工作，信件稍后会到）。

自 WS2016 引入，用于卷级别复制，实现灾难恢复。

### 6.2 关键驱动

**WVRF.sys**：类驱动 (disk.sys) 的上层过滤驱动

### 6.3 同步 vs 异步

**同步 (Synchronous)**：
```
1. 应用发送写入 → 2. 写入本地日志 + 复制到目标日志
→ 3. 目标确认收到 → 4. 源端提交给应用
→ 5. 双方从日志写入实际数据
```
- ✅ 零数据丢失
- ⚠️ 延迟受网络距离影响

**异步 (Asynchronous)**：
```
1. 应用发送写入 → 2. 写入本地日志 → 3. 提交给应用
→ 4. 后台复制到目标日志 → 5. 目标确认
```
- ✅ 低延迟（应用不等待远端确认）
- ⚠️ 可能丢失少量最新数据

### 6.4 支持场景

| 场景 | 描述 |
|------|------|
| **Server to Server** | 两台独立服务器间复制 |
| **Stretch Cluster** | 单集群跨站点节点间复制 |
| **Cluster to Cluster** | 两个不同集群间复制 |

### 6.5 目标中断时的行为

| 情况 | 处理方式 |
|------|---------|
| **短暂中断** | 重连后从日志中补发差量 |
| **日志环绕 (Log Wraps)** | 日志已覆盖旧数据，需全量同步 |
| **位图损坏** | 需要完全重新同步 |

### 6.6 版本限制

| 功能 | Datacenter 版 | Standard 版 |
|------|:------------:|:-----------:|
| 可复制卷数 | 无限 | **1 个** |
| 合作伙伴数 | 无限 | **1 个** |
| 卷大小上限 | 无限 | **2 TiB** |

---

## 7. Storage Migration Service — 存储迁移服务

自 WS2019 引入，简化文件服务器迁移。

### 三阶段流程

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ 1. 盘点   │ ──> │ 2. 传输   │ ──> │ 3. 割接   │
│ Inventory │     │ Transfer │     │ Cut Over │
└──────────┘     └──────────┘     └──────────┘
```

1. **盘点**：扫描源服务器的文件、共享和配置
2. **传输**：通过 SMB 将数据复制到目标服务器
3. **割接**（可选）：目标接管源服务器身份（名称、IP），对用户透明

**关键角色**：Orchestrator 服务器（运行 `Microsoft.StorageMigration.Service.exe`）、WAC 管理界面

---

## 8. 完整存储栈（含所有可选层）

```
┌─────────────────────────────────┐
│         应用程序                  │
├─────────────────────────────────┤
│         I/O 子系统 (IRP)         │
├─────────────────────────────────┤
│         文件系统                  │
│    NTFS.sys / ReFS.sys          │
├─────────────────────────────────┤
│    ★ Data Deduplication         │  ← 可选：数据重删
├─────────────────────────────────┤
│    ★ BitLocker (FVEVol.sys)     │  ← 可选：磁盘加密
├─────────────────────────────────┤
│         Volume Snapshot (VSS)    │
├─────────────────────────────────┤
│         Volume Manager           │
├─────────────────────────────────┤
│    ★ Storage Replica (WVRF.sys) │  ← 可选：存储复制
├─────────────────────────────────┤
│         Partition Manager        │
├─────────────────────────────────┤
│    ★ Storage Spaces (SpacePort) │  ← 可选：存储虚拟化
├─────────────────────────────────┤
│         Class Driver (disk.sys)  │
│    或 ★ MPIO (MPDev.sys)        │  ← 可选：多路径
├─────────────────────────────────┤
│         Port / Mini Port         │
├─────────────────────────────────┤
│         磁盘子系统（硬件）        │
└─────────────────────────────────┘
```

---

---

# English Version

---

# Deep Dive: Storage Advanced Layers — MPIO, Dedup, Storage Spaces & Replica

**Topic:** Windows Storage Advanced Layers  
**Category:** Storage  
**Level:** Senior-Expert  
**Last Updated:** 2026-03-17

---

## 1. Overview

Beyond the [simplified storage stack](../deep-dive-windows-storage-stack-architecture/), Windows provides several **optional layers** for multipathing, encryption, data optimization, storage virtualization, and replication. These layers plug into the stack at various positions like add-on modules.

> **Analogy**: If the simplified stack is a standard building, these advanced layers are optional add-ons: a security system (BitLocker), a deduplication compressor (Dedup), multiple elevator shafts (MPIO), a floor-merging system (Storage Spaces), and an off-site backup pipeline (Storage Replica).

---

## 2. MPIO — Multipath I/O

### 2.1 Why MPIO

When a server connects to the same SAN LUN through multiple physical paths (multiple HBAs, fiber cables, switches), Windows sees each path as a separate disk. MPIO aggregates them into one.

> **Analogy**: MPIO is like a GPS that knows multiple roads to the same destination. If one road is blocked (path failure), traffic automatically reroutes. Load balancing = spreading traffic across all available roads.

### 2.2 Architecture

- **MPIO.sys**: Aggregates all paths to one LUN, presents a single **Pseudo-LUN** to upper layers
- **DSM (Device Specific Module)**: Determines load balancing policy. Multiple DSMs can coexist
- **MPDev.sys / ClassPnP.sys**: Replaces standard disk.sys as class driver

### 2.3 Microsoft DSM Load Balancing Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **Failover** | Use primary path, switch on failure | Simple active/passive |
| **Failback** | Auto-return to primary after recovery | Prefer specific path |
| **Round Robin** | Rotate across all available paths | Even I/O distribution |
| **Round Robin with Subset** | Rotate within active path subset | Reserve standby paths |
| **Weighted Path** | Distribute I/O by weight | Different path bandwidths |

### 2.4 Diagnostics

```powershell
mpclaim -s -d           # List all multipath disks and paths
Get-MPIOSetting         # View MPIO global settings
```

---

## 3. BitLocker — Disk Encryption

> **Analogy**: BitLocker is like a safe around your hard drive. Even if someone steals the physical disk, the data is gibberish without the key (TPM chip or recovery key).

- **Driver**: `FVEVol.sys`, sits between file system and VSS
- Best with **TPM v1.2+** (no USB startup key needed)
- **32-digit recovery key** is the last resort
- Supports NTFS and ReFS
- ⚠️ **Security vs Fixability trade-off**: Data recovery nearly impossible without recovery key

---

## 4. Data Deduplication

> **Analogy**: Dedup is like a shared library. Instead of everyone buying their own copy of the same book, you keep one copy in the library and everyone references it. Saves massive shelf space.

### How It Works
- Files are chunked into small blocks
- Unique chunks stored once; duplicates replaced with references
- Only processes the primary (unnamed) data stream

### Space Savings

| Scenario | Content | Savings |
|----------|---------|---------|
| User file shares | Office docs, photos, videos | 30-50% |
| Deployment shares | Software binaries, CABs | 70-80% |
| Virtualization libraries | ISOs, VHD/VHDX files | **80-95%** |
| General file share | Mixed | 50-60% |

### Limitations
- Server only; not on boot/system volumes, CSV, encrypted files, or files < 64KiB
- ReFS support since WS2019 only
- ⚠️ Risk amplification: small chunk corruption affects many files → mitigated by checksums + redundant copies + scrubbing

---

## 5. Storage Spaces

> **Analogy**: Storage Spaces is like a water reservoir system. Instead of each house having its own well (individual disks), you pool all water sources together, then distribute water (virtual disks) as needed.

### Core Concept
Physical Disks → **Storage Pool** (SpacePort.sys) → **Virtual Disk (Storage Space)** → Partition → Volume

### Resiliency Types

| Type | Description | Capacity | Fault Tolerance |
|------|-------------|----------|----------------|
| **Simple** | Striped, no redundancy | 100% | ❌ None |
| **Mirror** | 2-3 copies | 33-50% | ✅ 1-2 disk failures |
| **Parity** | Data + parity striping (RAID5-like) | 67-87% | ✅ 1-2 disk failures |

### Key Drivers
- **SpacePort.sys**: Primary port driver, manages pools and spaces
- **smspace.dll**: Hardware provider
- **SpaceDump.sys**: BSOD dump support (WS2016+)

> ⚠️ **Virtual Disk** (Storage Space from SpacePort.sys) ≠ **Virtual Hard Disk** (.VHD/.VHDX for Hyper-V). Completely different concepts!

---

## 6. Storage Replica

> **Analogy**: Sync replication = sending a registered letter (you don't proceed until the recipient confirms). Async = sending regular mail (you continue working; it'll get there eventually).

### Key Points
- **Driver**: WVRF.sys (upper filter for disk.sys)
- **Synchronous**: Zero data loss; both sides commit together. Higher latency.
- **Asynchronous**: Source commits first, replicates later. Lower latency, small data loss risk.
- **Scenarios**: Server-to-Server, Stretch Cluster, Cluster-to-Cluster
- **Standard edition limits**: 1 volume, 1 partnership, 2 TiB max size

---

## 7. Storage Migration Service

Since WS2019. Three-phase file server migration:
1. **Inventory**: Scan source server files, shares, configuration
2. **Transfer**: Copy data via SMB to destination
3. **Cut Over** (optional): Destination assumes source identity (name, IP) — transparent to users

---

## 8. Complete Storage Stack (All Optional Layers)

```
┌─────────────────────────────────┐
│         Application              │
├─────────────────────────────────┤
│         I/O Subsystem (IRP)      │
├─────────────────────────────────┤
│         File System              │
│    NTFS.sys / ReFS.sys           │
├─────────────────────────────────┤
│    ★ Data Deduplication          │  ← Optional
├─────────────────────────────────┤
│    ★ BitLocker (FVEVol.sys)      │  ← Optional
├─────────────────────────────────┤
│         Volume Snapshot (VSS)    │
├─────────────────────────────────┤
│         Volume Manager           │
├─────────────────────────────────┤
│    ★ Storage Replica (WVRF.sys)  │  ← Optional
├─────────────────────────────────┤
│         Partition Manager        │
├─────────────────────────────────┤
│    ★ Storage Spaces (SpacePort)  │  ← Optional
├─────────────────────────────────┤
│         Class Driver (disk.sys)  │
│    or ★ MPIO (MPDev.sys)         │  ← Optional
├─────────────────────────────────┤
│         Port / Mini Port         │
├─────────────────────────────────┤
│         Disk Subsystem (HW)      │
└─────────────────────────────────┘
```

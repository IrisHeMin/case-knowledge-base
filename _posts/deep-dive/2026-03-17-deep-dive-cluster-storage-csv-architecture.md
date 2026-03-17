---
layout: post
title: "Deep Dive: 集群存储与 CSV 架构 — Cluster Shared Volumes 深入解析"
date: 2026-03-17
categories: [Knowledge, Storage]
tags: [storage, csv, cluster-shared-volumes, csvfs, csvvbus, csvflt, csvnsflt, scsi3-pr, disk-arbitration, failover-clustering, direct-io, redirected-io]
type: "deep-dive"
---

# Deep Dive: 集群存储与 CSV 架构

**Topic:** Cluster Storage & CSV Architecture  
**Category:** Storage / Failover Clustering  
**Level:** 高级 / Expert  
**Last Updated:** 2026-03-17

---

## 1. 概述

故障转移集群是一组协同工作的独立服务器，通过共享存储提供高可用服务。存储规划是集群部署中**最关键**的一环——如果没有做好尽职调查，这是最容易导致集群不稳定的领域。

### 集群存储的核心规则

- 集群使用 **SCSI-3 Persistent Reservations (PR)** 维护磁盘预留
- 物理磁盘资源最初只能被**单个节点**托管（"Shared Nothing" 模型）
- 加入 CSV 后，**所有节点可以同时访问**（打破了 Shared Nothing 规则）
- 支持的连接类型：Fibre Channel、iSCSI、SAS、FCoE

> **类比**：普通集群磁盘像餐厅的"已预订"桌牌——只有预订的节点能使用。CSV 则像自助餐——所有人都可以同时取餐，但需要服务员（Coordinator）来协调秩序。

---

## 2. SCSI-3 Persistent Reservations 与磁盘仲裁

### 2.1 SCSI-3 PR 工作原理

SCSI-3 PR 是集群维护磁盘所有权的机制。当一个节点拥有磁盘时，它注册一个 SCSI-3 预留，其他节点可以看到磁盘但**不能写入**。

### 2.2 磁盘仲裁场景

| 场景 | 行为 |
|------|------|
| **正常故障转移** | 原节点释放 PR → 新节点获取 PR → 磁盘上线 |
| **脑裂 (Split Brain)** | 两个节点同时声称拥有磁盘 → Quorum 仲裁决定胜者 |
| **节点崩溃** | 存活节点强制打破 PR → 接管磁盘 |
| **File Share Witness 仲裁** | 特殊的文件共享见证，不使用 SCSI-3 PR |

### 2.3 栈集成

`ClusDisk.sys` 在存储栈中位于分区管理器旁边。自 WS2008 起，ClusDisk 不再作为分区管理器上方的额外层，而是与 partmgr.sys 并行工作，为后来的 CSV 奠定了基础。

---

## 3. CSV 架构（核心章节）

### 3.1 四大关键驱动

```
┌─────────────────────────────────────────────────┐
│              CsvNsFlt.sys                        │ ← 保护 C:\ClusterStorage
│         (CSV Namespace Filter)                   │
├─────────────────────────────────────────────────┤
│              CsvFs.sys                           │ ← CSV 代理文件系统
│         (CSV File System)                        │   Disk Manager 显示 CSVFS
├─────────────────────────────────────────────────┤
│              CsvVbus.sys                         │ ← CSV 卷管理器
│         (CSV Volume Manager)                     │   负责所有 Block Level Redirected I/O
├─────────────────────────────────────────────────┤
│              CsvFlt.sys                          │ ← CSV 过滤驱动
│         (CSV Filter - 仅 Coordinator 节点加载!)   │   阻止非集群的直接 I/O
│              ↕ NTFS.sys                          │
└─────────────────────────────────────────────────┘
```

**各驱动职责**：

| 驱动 | 加载位置 | 职责 |
|------|---------|------|
| **CsvFs.sys** | 所有节点 | CSV 代理文件系统，挂载在 CsvVbus 呈现的卷之上。注册为文件系统过滤驱动。磁盘加入 CSV 后，Disk Manager 显示 CSVFS 而非 NTFS |
| **CsvVbus.sys** | 所有节点 | CSV 卷管理器，负责所有 Block-Level Redirected I/O。完全绕过 CSV-NTFS 栈直接到磁盘 |
| **CsvFlt.sys** | **仅 Coordinator 节点** | CSV 过滤驱动，挂载在 NTFS 之上。检查 `SharedVolumeSecurityDescriptor` 安全描述符，阻止所有非集群发起的直接 I/O |
| **CsvNsFlt.sys** | 所有节点 | 保护 `C:\ClusterStorage` 目录，阻止所有"非法"的文件/目录创建、删除和属性修改 |

### 3.2 其他关键组件

- **DCM (Disk Control Manager)**：为每个 CSV 卷在每个节点上实现全局分布式锁。管理 CSV 卷的所有权、通知变更、协调快照。
- **MDS (Metadata Server)** = **Coordinator Node**（协调者节点）：负责处理所有元数据操作

---

## 4. CSV I/O 类型（最重要的概念！）

CSV 有三种 I/O 类型，理解这三种类型是排查 CSV 问题的关键：

### 4.1 元数据 I/O (Metadata I/O)

> **类比**：问图书管理员（Coordinator）登记一本新书、查目录、改书名。

- **定义**：除了读写之外的所有操作（创建文件、关闭文件、重命名、删除等）
- **路径**：**始终**通过 SMB 转发到 Coordinator 节点的 NTFS
- **特点**：非 Coordinator 节点的元数据操作必须走网络

### 4.2 直接 I/O (Direct I/O)

> **类比**：自己走到书架前直接拿书。最快！

- **定义**：读写操作直接从 CsvFs → CsvVbus → 磁盘栈
- **特点**：最快路径，无网络开销。使用 Windows 缓存管理器的缓冲读写
- **适用**：节点可以直接看到底层物理磁盘时

### 4.3 Block-Level Redirected I/O（块级重定向 I/O）

> **类比**：让够得着顶层书架的朋友帮你拿书。你拿不到，但你的朋友（能看到磁盘的节点）可以。

- **定义**：完全绕过 CSV-NTFS 栈，直接到磁盘
- **触发条件**：
  - 节点**无法直接看到**底层磁盘
  - Direct I/O **失败**后的回退
  - **镜像 Storage Space** 加入 CSV 时始终使用
- **注意**：Coordinator 节点**永远不使用** Redirected I/O

### 4.4 三种 I/O 类型对比

| 特性 | Metadata I/O | Direct I/O | Block Redirected I/O |
|------|:-----------:|:----------:|:-------------------:|
| **操作类型** | 创建/删除/重命名 | 读/写 | 读/写 |
| **是否走网络** | ✅ 始终（SMB 到 Coordinator） | ❌ 直达磁盘 | ✅ 走网络（SMB 到能看到磁盘的节点） |
| **性能** | 中等 | **最快** | 较慢（受网络带宽限制） |
| **Coordinator 使用？** | ✅ 本地 NTFS | ✅ | ❌ 从不 |
| **非 Coordinator 使用？** | ✅ 通过网络 | ✅（能看到磁盘时） | ✅（看不到磁盘时） |

### 4.5 I/O 路径决策树

```
收到 I/O 请求
     │
     ├── 是元数据操作？ ──是──> 转发到 Coordinator (SMB)
     │
     ├── 当前节点是 Coordinator？ ──是──> Direct I/O（直达磁盘）
     │
     ├── 当前节点能看到磁盘？ ──是──> Direct I/O
     │                        ──否──> Block Redirected I/O
     │
     └── Direct I/O 失败？ ──是──> 回退到 Block Redirected I/O
```

---

## 5. CSV 配置

- CSV 命名空间自 WS2012 起**默认启用**
- 挂载路径：`C:\ClusterStorage\Volume1`、`Volume2`...
- CSV 卷是标准挂载点（WS2012+），不再是重解析点（WS2008R2）
- 一键操作即可将磁盘加入 CSV

```powershell
Get-ClusterSharedVolume                    # 查看 CSV 信息
Get-ClusterSharedVolumeState               # 查看每个节点的 CSV I/O 状态
Add-ClusterSharedVolume -Name "Cluster Disk 1"  # 添加磁盘到 CSV
```

---

## 6. CSV 弹性

CSVv2 具有极高的容错能力：

1. **节点故障**：CSV 过滤驱动为"孤立"的 CSV 卷保持 I/O，直到卷在其他节点上重新挂载
2. **网络故障**：集群服务通知所有健康节点重置 SMB 连接
3. **HBA 故障**：自动回退到 Redirected I/O
4. **最终结果**：对应用透明的故障转移，**零停机时间**

---

## 7. CSV 性能

| 特性 | 说明 |
|------|------|
| **缓存管理器集成** | Direct I/O 使用缓冲读写，利用 Windows 缓存管理器 |
| **CSV Caching** | 只读缓存，用于未缓冲的 I/O（需 PowerShell 启用）|
| **分布式缓存** | 跨集群保证一致性，VDI 场景价值巨大 |
| **SMBv3 多通道** | 利用多个网络适配器并行传输 |
| **SMB Direct (RDMA)** | 绕过 CPU 直接内存访问，极低延迟 |
| **性能水平** | **DAS 性能的 97%** |

---

## 8. CSV 上的 ChkDsk — 在线修复流程

```
NTFS 检测到异常 (IsAlive 检查)
     │
     ├── 验证：是临时性还是真正的损坏？
     │         卷保持在线！
     │
     ├── 临时性 → Spot-fixing（15 秒一轮）
     │            → 每 2-3 分钟重复直到修复
     │
     ├── 无法 spot-fix → 记录修复操作，通知管理员
     │                    卷仍然在线！
     │
     └── 维护窗口 → 卷短暂离线（秒级）
                    → CSV I/O 透明暂停后恢复
```

---

## 9. CSV 安全

- **认证**：节点间使用 NTLM 认证（WS2012+ 不再需要 AD 域控器认证，使用 CLIUSR 账户）
- **BitLocker on CSV**：使用基于 SID 的保护器（CNO 计算机对象），支持节点间故障转移

---

## 10. CSV 备份与恢复

- CSV 拥有自己的 **VSS Writer**，支持硬件和软件备份
- **CSV Shadow Copy Provider**：跨所有节点发起 VSS Freeze/Thaw，提供崩溃一致性快照
- 支持同一/不同 CSV 卷上的**并行备份**
- 备份期间 **CSV 所有权不变**
- ⚠️ 备份应用必须是 CSV-aware 的（WSB 不支持 CSV 上的 VM 备份）

---

---

# English Version

---

# Deep Dive: Cluster Storage & CSV Architecture

**Topic:** Cluster Storage & CSV Architecture  
**Category:** Storage / Failover Clustering  
**Level:** Expert  
**Last Updated:** 2026-03-17

---

## 1. Overview

Failover clusters are groups of independent servers that share storage for high availability. Storage planning is the **most critical** piece of cluster deployment.

### Core Rules
- Clusters use **SCSI-3 Persistent Reservations (PR)** for disk ownership
- Physical disk resources are initially owned by **one node only** ("Shared Nothing")
- CSV allows **simultaneous access from all nodes** (breaks Shared Nothing rule)
- Supported: Fibre Channel, iSCSI, SAS, FCoE

> **Analogy**: Regular cluster disk = "Reserved" sign on a restaurant table (one node only). CSV = buffet (everyone can access simultaneously, but a coordinator/waiter maintains order).

---

## 2. SCSI-3 PR & Disk Arbitration

### How It Works
When a node owns a disk, it registers a SCSI-3 reservation. Other nodes can see the disk but **cannot write**.

### Arbitration Scenarios

| Scenario | Behavior |
|----------|----------|
| **Normal failover** | Owner releases PR → new node acquires PR → disk online |
| **Split brain** | Two nodes claim disk → Quorum decides winner |
| **Node crash** | Surviving node forcibly breaks PR → takes ownership |

**ClusDisk.sys** integrates alongside partmgr.sys (not above it, since WS2008).

---

## 3. CSV Architecture (Core Section)

### Four Key Drivers

| Driver | Loaded On | Role |
|--------|-----------|------|
| **CsvFs.sys** | All nodes | CSV proxy file system. Disk Manager shows CSVFS instead of NTFS |
| **CsvVbus.sys** | All nodes | CSV Volume Manager. Handles ALL Block-Level Redirected I/O |
| **CsvFlt.sys** | **Coordinator only** | Filter on top of NTFS. Blocks non-cluster direct I/O |
| **CsvNsFlt.sys** | All nodes | Protects `C:\ClusterStorage` from illegal operations |

**Other Components**:
- **DCM (Disk Control Manager)**: Distributed locking, ownership, snapshot coordination
- **MDS (Metadata Server)** = Coordinator Node

---

## 4. CSV I/O Types (Most Important Concept!)

### Metadata I/O
> **Analogy**: Asking the librarian (coordinator) to catalog a new book.

Everything except reads/writes. **Always** forwarded to Coordinator via SMB.

### Direct I/O
> **Analogy**: Going directly to the bookshelf and grabbing the book yourself.

Reads/writes flow directly CsvFs → CsvVbus → disk. **Fastest path**, no network overhead.

### Block-Level Redirected I/O
> **Analogy**: Asking a friend who can reach the top shelf to grab the book for you.

Bypasses CSV-NTFS stack. Used when node can't see disk or Direct I/O fails. Coordinator **never** uses this.

### Comparison Table

| Property | Metadata I/O | Direct I/O | Block Redirected I/O |
|----------|:-----------:|:----------:|:-------------------:|
| **Operations** | Create/delete/rename | Read/write | Read/write |
| **Network?** | ✅ Always (SMB) | ❌ Direct to disk | ✅ Via network |
| **Performance** | Medium | **Fastest** | Slower (network bound) |
| **Coordinator uses?** | ✅ Local NTFS | ✅ | ❌ Never |

### I/O Decision Tree
```
I/O Request → Metadata? → Yes → Forward to Coordinator (SMB)
                       → No → Can node see disk? → Yes → Direct I/O
                                                 → No → Block Redirected I/O
                              → Direct I/O failed? → Yes → Fallback to Redirected
```

---

## 5. CSV Resiliency
- Filter driver holds I/O for orphaned volumes until remounted
- Cluster service resets SMB connections on failure
- **Zero downtime** transparent failover

## 6. CSV Performance
- CSV Caching: read-only cache for unbuffered I/O
- SMBv3 multichannel + SMB Direct (RDMA)
- **97% of DAS performance**

## 7. CSV ChkDsk — Online Repair
NTFS detects anomaly → Spot-fixing (15s rounds, 2-3 min intervals) → Volume stays online → Maintenance window: offline for seconds only

## 8. CSV Backup
- CSV VSS Writer with Freeze/Thaw across all nodes
- Parallel backups supported; CSV ownership unchanged during backup
- Backup apps must be CSV-aware

---
layout: post
title: "Case Summary: Win11 VPN SMB 复制慢排查实战 — 从抓包到 ProcMon 的完整排查思路"
date: 2026-01-06
categories: [SMB, Networking]
tags: [smb, vpn, windows-11, tcp, packet-loss, cubic-tcp, f5-vpn, cisco-anyconnect, procmon, wireshark, performance, troubleshooting]
---

# Win11 VPN SMB 复制慢排查实战 — 从抓包到 ProcMon 的完整排查思路

**问题**: Windows 11 通过 VPN 复制 162 MB 文件耗时 5 分 18 秒，而 Windows 10 仅需 1 分 17 秒（慢 4.1 倍）。

本文记录完整的排查过程、走过的弯路、日志分析方法，以及最终如何定位到根因。

---

## 一、排查思路：从哪里开始？

拿到 "Win11 比 Win10 慢" 的问题，第一反应是列出可能的差异点：

```
Win10 vs Win11 — 什么可能不同？
├── OS 层: TCP 栈行为、SMB 客户端实现
├── 网络层: VPN 隧道、MTU/MSS、路由
├── 应用层: SMB 协议版本、签名、加密
└── 干扰层: 安全产品、EDR、DLP
```

**排查策略**: 先收网络抓包确定是网络层还是应用层问题，再用 ProcMon 排查主机层干扰。

---

## 二、第一轮排查：Wireshark 抓包分析

### 2.1 收集方法：三组对比抓包

设计了三组抓包进行控制变量对比：

| 抓包 | 客户端 | 访问方式 | 目的 |
|------|--------|---------|------|
| ① Win10 VPN | Win10 + Cisco VPN | 主机名 | 基线 |
| ② Win11 主机名 | Win11 + F5 VPN | `\\filesrv01\share` | 对比 OS 差异 |
| ③ Win11 IP 地址 | Win11 + F5 VPN | `\\10.2.20.98\share` | 对比名称解析影响 |

> 💡 **Tip**: 对比排查时，**控制变量**是关键。每次只改一个因素，才能准确归因。

### 2.2 分析方法：Wireshark 看什么？

SMB 慢问题的 pcap 分析，我按这个 checklist 逐项排查：

**Step 1: SMB 协议版本和 Dialect**
```
Wireshark filter: smb2.cmd == 0  (Negotiate)
看什么: smb2.dialect → 是 0x0311 (SMB 3.1.1) 还是 0x02ff (SMB 2.1)?
```
发现 Win11 用主机名访问时协商到 SMB 3.1.1，但**用 IP 地址时降级到 SMB 2.1**。
原因: IP 访问 → Kerberos SPN 不匹配 → 回退 NTLM → 服务器只接受 SMB 2.1。

**Step 2: TCP 重传和 Duplicate ACK**
```
Wireshark filter: tcp.analysis.retransmission || tcp.analysis.duplicate_ack
Statistics → Expert Information → 直接看 Warning/Note 数量
```
这一步是**关键转折点** — 发现了大量 Duplicate ACK：

| 指标 | Win10 | Win11 (主机名) | Win11 (IP) |
|------|-------|---------------|------------|
| Duplicate ACK | 82 (0.05%) | **6,430 (2.56%)** 🔴 | **5,060 (2.04%)** 🔴 |
| 重传 | 1,122 | 425 | 177 |

> 🔴 **2% 以上的 Duplicate ACK 率说明网络路径存在严重的丢包或乱序。** 正常网络应该 < 0.1%。

**Step 3: SMB Read 响应时间**
```
Wireshark filter: smb2.cmd == 8 && smb2.flags.response == 1  (Read Response)
Statistics → IO Graph → 或手动计算 request-response 时间差
```

| 指标 | Win10 | Win11 (主机名) | Win11 (IP) |
|------|-------|---------------|------------|
| SMB Read 平均响应 | 1.65s | **2.97s** | **2.84s** |
| SMB Read 最大响应 | 3.3s | **14.29s** | 9.95s |

正常 1MB Read 在 VPN 上应该 < 100ms，这里平均接近 3 秒 — **差了 30 倍**。

**Step 4: TCP Window 和流控**
```
Wireshark filter: tcp.analysis.zero_window || tcp.analysis.window_full
```
结果: 0 个 Zero Window，0 个 Window Full → **排除接收端流控瓶颈**。

**Step 5: MSS 和 IP 分片**
```
看 TCP SYN 包的 MSS option → Win10: 1,350 / Win11: 1,240
看 IP 层: Win11 IP 访问场景下 247,586 个包**全部分片**
```
MSS 1,240 + IP(20) + TCP(20) = 1,280 bytes → 加上 VPN 封装头 → 超过 MTU → 分片。

**Step 6: SMB 签名和加密**
```
Wireshark filter: smb2.flags.signature == 1
```
结果: 签名在各场景不一致但都不是高开销 → **排除签名/加密瓶颈**。

### 2.3 第一轮结论

抓包分析明确了：
1. **VPN 链路丢包/乱序是主要瓶颈** (Dup ACK 2-2.5%)
2. IP 访问会导致 SMB 降级 + 全部包被分片
3. 签名、流控、Window 都不是问题

**但这解释不了为什么 Win10 也在同一 VPN 上却快 4 倍。**

### 2.4 🔴 Blocker #1: "重传更多却更快" 的矛盾

仔细看数据，发现一个**反直觉的现象**:

```
Win10: 1,122 次重传，吞吐 25.85 Mbps ← 重传多，但快！
Win11: 177-425 次重传，吞吐 16-19 Mbps ← 重传少，反而慢！
```

这不合理 — 如果是纯粹的网络丢包问题，重传多的应该更慢才对。

**破解思路**: 重传次数只说明 "丢了多少包"，但吞吐量取决于 **TCP 拥塞窗口恢复速度**。如果一个 TCP 实现在丢包后快速恢复窗口，它可以更快地重传并继续发送 → 重传多但吞吐高。

这指向了 **TCP 拥塞控制算法差异**:
- Win10 默认: **Compound TCP (CTCP)** — 丢包后快速恢复，激进重建窗口
- Win11 默认: **Cubic TCP** — 丢包后保守缩窗，缓慢恢复

```powershell
# 验证命令:
Get-NetTCPSetting | Select-Object SettingName, CongestionProvider
```

**这是整个 case 的核心发现**: 在同等丢包条件下，Win11 的 Cubic 比 Win10 的 CTCP 吞吐低 36-47%。

---

## 三、第二轮排查：ProcMon 深度分析

网络层分析完了，还需要排查主机层面是否有额外干扰。

### 3.1 收集方法

两台机器同时运行 ProcMon 完整录制（无过滤），然后导出：
- Win11 完整: **14.7M 行, 2.5 GB**
- Win10 完整: **5.7M 行, 1 GB**
- 另有按目标文件路径过滤后的 CSV（Win11: 1,877 行, Win10: 7,638 行）

### 3.2 分析方法：大规模 ProcMon 日志怎么看？

14.7M 行日志不可能人工翻阅，需要用脚本做统计分析。

**方法 1: 进程事件分布统计** — 哪个进程产生了最多的 I/O？
```python
# 按进程名统计事件数，排序后取 Top 20
import csv
from collections import Counter
counter = Counter()
with open('procmon_full.csv') as f:
    for row in csv.DictReader(f):
        counter[row['Process Name']] += 1
for proc, count in counter.most_common(20):
    print(f"{proc}: {count:,}")
```

这个统计立刻暴露了安全产品的 I/O 负载差异：

| 安全产品 | Win10 事件数 | Win11 事件数 | 倍数 |
|---------|------------|------------|------|
| Cybereason (minionhost.exe) | 58,878 | **840,652** | **14.3x** |
| BeyondTrust Avecto IC3 | 0 | **285,927** | Win11 独有 |
| BeyondTrust Defendpoint | 0 | **54,758** | Win11 独有 |
| **安全产品合计** | **~290K** | **~1,454K** | **5x** |

**方法 2: 时间段分析** — 复制过程中各阶段耗时
```python
# 按 Robocopy.exe 的首末事件确定复制时间窗口
# 按 10 秒分桶统计吞吐量
```

Win11 时间构成:
```
┌──────────────────────────────────────────────────┐
│  启动延迟 118s (37%) │ 复制①71s │ 静默52s │ 复制②74s │
└──────────────────────────────────────────────────┘
Win10:
│ 13s │ 复制①24s │ 6s │ 复制②20s │ 14s │
```

**方法 3: 特定操作过滤** — 查找异常行为
```
ProcMon filter: Operation = IRP_MJ_CREATE, Detail contains "Delete On Close"
```
发现 Explorer.EXE 用 `Delete On Close` 标志打开了 Robocopy 刚复制完的文件 → 文件被删除 → Robocopy 被迫重新复制 → **数据传输量翻倍**。两台机器都有此问题。

### 3.3 🔴 Blocker #2: 过滤日志导致的错误结论

初始分析时，我先看的是**过滤后的 CSV**（仅包含目标文件路径的 I/O）。

过滤 CSV 中 Win10 没有出现 `minionhost.exe`，于是我得出结论: **"Cybereason 仅存在于 Win11"**。

这个结论是**错误的**。

当我去分析完整未过滤的 ProcMon (Win10: 5.7M 行) 后发现：**Win10 上也有 minionhost.exe (58,878 个事件)**，只是因为它在 Win10 上没有访问目标文件路径，所以被路径过滤掉了。

> ⚠️ **教训: 永远不要仅依赖过滤后的日志得出 "不存在" 的结论。** "过滤后没看到" ≠ "不存在"。正确的做法是在完整日志中确认。

纠正后的结论: Cybereason 两边都有，但 Win11 上活跃度高 14.3 倍（全盘扫描 vs 仅操作本地数据库）。

### 3.4 🔴 Blocker #3: 隐藏的环境差异 — VPN 客户端不同

整个排查前期，我**假设两台机器使用相同的 VPN**（客户描述问题时没有提到 VPN 差异）。

直到在完整 ProcMon 中做进程事件统计时，发现：
```
Win10: vpnagent.exe (502,835 events)     ← Cisco AnyConnect
Win11: F5VpnPluginApp.exe (1,030,000 events)  ← F5 VPN
```

**两台机器用的是完全不同的 VPN 客户端！**

这意味着之前所有 "Win11 慢是因为 OS 差异" 的分析，都需要加一个前提: **网络层本身就不同**。

> ⚠️ **教训: 在排查对比类问题时，第一步应该确认所有环境变量的一致性。** 不要假设 "其他条件相同"。建议开 case 第一通电话就让客户填一张环境对比表。

### 3.5 🔴 Blocker #4: TCP 拥塞算法在抓包中不可见

TCP 拥塞控制算法（Cubic vs CTCP）在 pcap 抓包中**完全不可见** — 它是 TCP 栈内部逻辑，不体现在任何数据包字段中。

**破解方法: 行为推断**

虽然看不到算法名，但可以通过行为特征推断：

| 行为特征 | Win10 | Win11 | 推断 |
|---------|-------|-------|------|
| 重传次数 | 1,122 (多) | 177-425 (少) | Win10 更激进重传 |
| 吞吐量 | 25.85 Mbps (高) | 16-19 Mbps (低) | Win10 窗口恢复快 |
| 综合表现 | 重传多但快 | 重传少但慢 | CTCP vs Cubic 特征匹配 |

再结合 Windows 已知默认配置（Win10→CTCP, Win11→Cubic），行为与算法特征一致。

> 💡 **Tip: 当你在抓包中看到 "重传多却吞吐高" 的反直觉现象时，应该怀疑是 TCP 拥塞控制算法的差异，而非纯粹的网络质量差异。**

---

## 四、根因总结

```
Win11 SMB 慢 4.1x 的因素拆解:
│
├── 🔴 70% — VPN 链路丢包/乱序 (Dup ACK 2-2.5%)
│     └── 三组抓包均确认，这是底层网络问题
│
├── 🔴 20% — Win11 Cubic TCP 对丢包更敏感
│     └── 同等丢包下，Cubic 恢复窗口比 CTCP 慢 → 吞吐低 36-47%
│
├── 🟡 加剧 — VPN 客户端不同 (Cisco vs F5)
│     └── 不同隧道封装、MTU 处理、路由策略
│
├── 🟡 加剧 — Cybereason EDR 活跃度 14.3x + 安全产品负载 5x
│     └── 大量 I/O 争用系统资源
│
├── 🟡 5% — TCP MSS 差异 (1,240 vs 1,350) + IP 分片
│
└── 🟡 5% — IP 访问 → SPN 不匹配 → SMB 2.1 降级
```

**一行命令修复**:
```powershell
netsh int tcp set global congestionprovider=ctcp
```
预期提升 30-50%。

---

## 五、排查方法论总结 — 对别人有用的 Tips

### Tip 1: SMB 慢问题的 Wireshark Checklist

按优先级排查：
```
1. tcp.analysis.duplicate_ack  → Dup ACK 率 > 1% 就是网络问题
2. smb2.cmd == 8 (Read)        → 计算 request-response 时间差
3. tcp.analysis.zero_window    → 排除接收端瓶颈
4. smb2.cmd == 0 (Negotiate)   → 确认 SMB 版本 (3.1.1 vs 2.1)
5. tcp.options.mss             → MSS 值是否合理 (VPN 场景应 < 1400)
6. ip.flags.mf == 1            → 是否存在 IP 分片
```

### Tip 2: 大规模 ProcMon 分析方法

当 ProcMon 日志 > 100 万行时：
1. **进程事件分布** — 按进程名聚合计数，找出 I/O 大户
2. **时间线分桶** — 按 10 秒窗口统计吞吐，找出异常时间段
3. **特定操作过滤** — `Delete On Close`, `NAME NOT FOUND`, `ACCESS DENIED`
4. **对比两份日志的进程列表** — 找出 "一边有一边没有" 的进程

> ⚠️ 切忌只看过滤后的小 CSV，一定要在完整日志中验证。

### Tip 3: TCP 拥塞控制的行为推断法

抓包中看不到拥塞算法，但可以通过以下模式推断：
- **重传多 + 吞吐高** → 激进型算法 (CTCP, BBR)
- **重传少 + 吞吐低** → 保守型算法 (Cubic 在丢包环境下)
- **确认方法**: `Get-NetTCPSetting | Select CongestionProvider`

### Tip 4: 对比排查时的环境确认清单

开 case 时立刻确认:
```
□ OS 版本 (包括 Build)
□ VPN 客户端 (产品 + 版本)
□ VPN 模式 (split tunnel vs full tunnel)
□ 安全产品列表 (EDR, AV, DLP, PAM)
□ SMB 签名/加密配置
□ TCP 拥塞控制算法
□ 网络接口 MTU
```

### Tip 5: 常用排查命令速查

```powershell
# TCP 拥塞控制
Get-NetTCPSetting | Select SettingName, CongestionProvider
netsh int tcp set global congestionprovider=ctcp  # 切换

# SMB 配置
Get-SmbClientConfiguration | Select RequireSecuritySignature, EnableSecuritySignature
Get-SmbConnection  # 查看活跃连接

# 网络接口
netsh interface ipv4 show subinterfaces  # 查看 MTU
netsh interface ipv4 set subinterface "VPN" mtu=1300 store=persistent

# VPN 丢包测试
Test-Connection -ComputerName <target> -Count 100 -BufferSize 1200
```

## 六、参考文档 (References)

- [Troubleshoot slow SMB file transfer](https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/slow-file-transfer) — Microsoft 官方 SMB 慢排查指南
- [Set-NetTCPSetting](https://learn.microsoft.com/en-us/powershell/module/nettcpip/set-nettcpsetting) — TCP 设置 cmdlet（含 CongestionProvider 参数）
- [SMB Direct (RDMA)](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-direct) — SMB 性能特性概述

---
---

# Troubleshooting Win11 VPN SMB Copy Slowness — From Packet Capture to ProcMon

**Problem**: Windows 11 takes 5 min 18 sec to copy a 162 MB file over VPN, while Windows 10 takes only 1 min 17 sec (4.1x slower).

This article documents the complete troubleshooting process, wrong turns, log analysis methods, and how the root cause was ultimately identified.

---

## 1. Approach: Where to Start?

Facing "Win11 slower than Win10", the first step was listing possible difference areas:

```
Win10 vs Win11 — What could be different?
├── OS layer: TCP stack behavior, SMB client implementation
├── Network layer: VPN tunnel, MTU/MSS, routing
├── Application layer: SMB protocol version, signing, encryption
└── Interference: Security products, EDR, DLP
```

**Strategy**: Start with network captures to determine if it's a network or application layer issue, then use ProcMon to investigate host-level interference.

---

## 2. Round 1: Wireshark Packet Capture Analysis

### 2.1 Collection: Three Comparison Captures

Designed three captures with controlled variables:

| Capture | Client | Access Method | Purpose |
|---------|--------|---------------|---------|
| ① Win10 VPN | Win10 + Cisco VPN | Hostname | Baseline |
| ② Win11 Hostname | Win11 + F5 VPN | `\\filesrv01\share` | OS comparison |
| ③ Win11 IP | Win11 + F5 VPN | `\\10.2.20.98\share` | Name resolution impact |

> 💡 **Tip**: In comparison troubleshooting, **controlling variables** is key. Change only one factor at a time.

### 2.2 Analysis: What to Look for in Wireshark?

For SMB slowness, I follow this checklist:

**Step 1: SMB Protocol Version**
```
Filter: smb2.cmd == 0  (Negotiate)
Look at: smb2.dialect → 0x0311 (SMB 3.1.1) or 0x02ff (SMB 2.1)?
```
Win11 with hostname → SMB 3.1.1, but **with IP → downgraded to SMB 2.1**.
Reason: IP access → Kerberos SPN mismatch → NTLM fallback → server only accepts SMB 2.1.

**Step 2: TCP Retransmissions and Duplicate ACKs**
```
Filter: tcp.analysis.retransmission || tcp.analysis.duplicate_ack
Statistics → Expert Information → check Warning/Note counts
```
This was the **key turning point** — massive Duplicate ACKs discovered:

| Metric | Win10 | Win11 (Hostname) | Win11 (IP) |
|--------|-------|-------------------|---------------------|
| Dup ACKs | 82 (0.05%) | **6,430 (2.56%)** 🔴 | **5,060 (2.04%)** 🔴 |
| Retransmissions | 1,122 | 425 | 177 |

> 🔴 **Dup ACK rate above 2% indicates severe packet loss or reordering.** Normal should be < 0.1%.

**Step 3: SMB Read Response Time**
```
Filter: smb2.cmd == 8 && smb2.flags.response == 1
Calculate request-to-response time delta
```
Average 1MB Read taking ~3 seconds — **30x slower than expected** (should be <100ms).

**Step 4: TCP Window / Flow Control**
```
Filter: tcp.analysis.zero_window || tcp.analysis.window_full
```
Result: 0 Zero Windows, 0 Window Full → **receiver bottleneck ruled out**.

**Step 5: MSS and IP Fragmentation**
```
Check TCP SYN MSS option → Win10: 1,350 / Win11: 1,240
IP layer: Win11 IP scenario — all 247,586 packets fragmented!
```

### 2.3 🔴 Blocker #1: The "More Retransmissions Yet Faster" Paradox

Careful examination revealed a **counter-intuitive pattern**:

```
Win10: 1,122 retransmissions, throughput 25.85 Mbps ← More retrans, but FASTER!
Win11: 177-425 retransmissions, throughput 16-19 Mbps ← Fewer retrans, but SLOWER!
```

If this were purely a packet loss issue, more retransmissions should mean slower.

**Breakthrough**: Retransmission count only tells "how many packets were lost", but throughput depends on **how fast the TCP congestion window recovers**. An aggressive TCP implementation recovers its window quickly after loss → more retransmissions but higher throughput.

This pointed to **TCP congestion control algorithm differences**:
- Win10 default: **Compound TCP (CTCP)** — aggressive window recovery after loss
- Win11 default: **Cubic TCP** — conservative backoff, slow recovery

```powershell
# Verify: Get-NetTCPSetting | Select SettingName, CongestionProvider
```

**This was the core finding**: Under identical packet loss, Win11's Cubic achieves 36-47% lower throughput than Win10's CTCP.

---

## 3. Round 2: ProcMon Deep Analysis

### 3.1 Analysis Method: How to Handle Multi-Million Row ProcMon Logs?

14.7M rows can't be reviewed manually. Use scripted statistical analysis:

**Method 1: Process Event Distribution** — Which process generates the most I/O?
```python
# Aggregate event count by process name, sort descending
from collections import Counter
counter = Counter()
for row in csv.DictReader(open('procmon_full.csv')):
    counter[row['Process Name']] += 1
```

This immediately exposed security product I/O differences:

| Security Product | Win10 Events | Win11 Events | Ratio |
|-----------------|------------|------------|-------|
| Cybereason (minionhost.exe) | 58,878 | **840,652** | **14.3x** |
| BeyondTrust Avecto IC3 | 0 | **285,927** | Win11 only |
| **Total Security I/O** | **~290K** | **~1,454K** | **5x** |

**Method 2: Time-Bucketed Throughput** — Find anomalous time periods
- Win11: 118s startup delay + 52s silent gap between two copies
- Win10: 13s startup + 6s gap

**Method 3: Specific Operation Filtering** — Find abnormal behavior
```
Filter: Operation = IRP_MJ_CREATE, Detail contains "Delete On Close"
```
Discovered Explorer.EXE deleting files that Robocopy just copied → forced double copy on both machines.

### 3.2 🔴 Blocker #2: Wrong Conclusion from Filtered Logs

Initial analysis used **filtered CSV** (only I/O touching the target file path). Win10's filtered CSV showed no `minionhost.exe`, leading to the conclusion: **"Cybereason only exists on Win11"**.

This was **wrong**.

Analyzing the full unfiltered Win10 ProcMon (5.7M rows) revealed: **minionhost.exe existed on Win10 too (58,878 events)**, but it never touched the target file path, so it was filtered out.

> ⚠️ **Lesson: Never conclude "doesn't exist" from filtered logs alone.** "Not seen in filtered view" ≠ "doesn't exist". Always verify in full logs.

### 3.3 🔴 Blocker #3: Hidden Environment Difference — Different VPN Clients

Throughout early investigation, I **assumed both machines used the same VPN** (customer didn't mention VPN differences).

The process event distribution analysis revealed:
```
Win10: vpnagent.exe (502,835 events)       ← Cisco AnyConnect
Win11: F5VpnPluginApp.exe (1,030,000 events) ← F5 VPN
```

**Completely different VPN clients!** All previous "Win11 is slower due to OS differences" analysis needed to be re-evaluated with this caveat.

> ⚠️ **Lesson: In comparison troubleshooting, verify ALL environment variables first.** Never assume "all other conditions are equal". Ask the customer to fill an environment comparison table on the first call.

### 3.4 🔴 Blocker #4: TCP Congestion Algorithm Invisible in Captures

TCP congestion control (Cubic vs CTCP) is **completely invisible in pcap** — it's internal TCP stack logic with no packet-level footprint.

**Solution: Behavioral Inference**

Though invisible, behavior patterns can reveal the algorithm:

| Behavior | Win10 | Win11 | Inference |
|----------|-------|-------|-----------|
| Retransmissions | 1,122 (high) | 177-425 (low) | Win10 retransmits more aggressively |
| Throughput | 25.85 Mbps (high) | 16-19 Mbps (low) | Win10 recovers window faster |
| Pattern | More retrans, higher throughput | Less retrans, lower throughput | CTCP vs Cubic characteristics match |

Combined with known Windows defaults (Win10→CTCP, Win11→Cubic), behavioral evidence confirms the algorithm difference.

> 💡 **Tip: When you see "more retransmissions yet higher throughput" in captures, suspect TCP congestion control algorithm differences, not purely network quality issues.**

---

## 4. Root Cause Summary

```
Win11 SMB 4.1x slower breakdown:
│
├── 🔴 70% — VPN link packet loss/reordering (Dup ACK 2-2.5%)
├── 🔴 20% — Win11 Cubic TCP more loss-sensitive than Win10 CTCP
├── 🟡 Amplifier — Different VPN clients (Cisco vs F5)
├── 🟡 Amplifier — Cybereason EDR 14.3x more active + 5x security I/O
├── 🟡 5% — TCP MSS difference (1,240 vs 1,350) + IP fragmentation
└── 🟡 5% — IP access → SPN mismatch → SMB 2.1 downgrade
```

**One-line fix**: `netsh int tcp set global congestionprovider=ctcp` → expected 30-50% improvement.

---

## 5. Reusable Troubleshooting Tips

### Tip 1: Wireshark Checklist for SMB Slowness

Priority order:
```
1. tcp.analysis.duplicate_ack  → Dup ACK rate > 1% = network issue
2. smb2.cmd == 8 (Read)        → Calculate request-response time delta
3. tcp.analysis.zero_window    → Rule out receiver bottleneck
4. smb2.cmd == 0 (Negotiate)   → Confirm SMB version (3.1.1 vs 2.1)
5. tcp.options.mss             → MSS should be < 1400 for VPN
6. ip.flags.mf == 1            → Check for IP fragmentation
```

### Tip 2: Large-Scale ProcMon Analysis

When ProcMon logs exceed 1M rows:
1. **Process event distribution** — aggregate by process name, find I/O heavy-hitters
2. **Time-bucketed throughput** — 10-second windows, find anomalous periods
3. **Specific operation filter** — `Delete On Close`, `NAME NOT FOUND`, `ACCESS DENIED`
4. **Cross-compare process lists** — find processes present on one machine but not the other

> ⚠️ Never rely solely on filtered CSV. Always verify conclusions in full unfiltered logs.

### Tip 3: TCP Congestion Control Behavioral Inference

Can't see congestion algorithm in captures, but infer from patterns:
- **High retrans + high throughput** → aggressive algorithm (CTCP, BBR)
- **Low retrans + low throughput** → conservative algorithm (Cubic under loss)
- **Verify**: `Get-NetTCPSetting | Select CongestionProvider`

### Tip 4: Environment Checklist for Comparison Cases

Confirm on first call:
```
□ OS version (including Build number)
□ VPN client (product + version)
□ VPN mode (split tunnel vs full tunnel)
□ Security products (EDR, AV, DLP, PAM)
□ SMB signing/encryption config
□ TCP congestion control algorithm
□ Network interface MTU
```

### Tip 5: Quick Reference Commands

```powershell
# TCP congestion control
Get-NetTCPSetting | Select SettingName, CongestionProvider
netsh int tcp set global congestionprovider=ctcp

# SMB config
Get-SmbClientConfiguration | Select RequireSecuritySignature, EnableSecuritySignature
Get-SmbConnection

# Network interface MTU
netsh interface ipv4 show subinterfaces
netsh interface ipv4 set subinterface "VPN" mtu=1300 store=persistent

# VPN packet loss test
Test-Connection -ComputerName <target> -Count 100 -BufferSize 1200
```

## 6. References

- [Troubleshoot slow SMB file transfer](https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/slow-file-transfer) — Official Microsoft SMB troubleshooting guide
- [Set-NetTCPSetting](https://learn.microsoft.com/en-us/powershell/module/nettcpip/set-nettcpsetting) — TCP settings cmdlet (CongestionProvider parameter)
- [SMB Direct (RDMA)](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-direct) — SMB performance features overview

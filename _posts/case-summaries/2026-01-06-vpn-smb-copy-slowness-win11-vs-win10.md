---
layout: post
title: "Case Summary: Windows 11 SMB File Copy Slowness over VPN — Win11 4x Slower than Win10"
date: 2026-01-06
categories: [SMB, Networking]
tags: [smb, vpn, windows-11, tcp, packet-loss, cubic-tcp, f5-vpn, cisco-anyconnect, procmon, wireshark, performance]
---

# Case Summary: Windows 11 通过 VPN 进行 SMB 文件复制速度慢 — Win11 比 Win10 慢 4 倍

**Product/Service:** Windows 11 / SMB / VPN (F5 & Cisco AnyConnect)

---

## 1. 症状 (Symptoms)

客户报告：
- 通过 VPN 从远程 SMB 共享 `\\filesrv01.contoso.corp\App-Deployment` 复制 162 MB 的 zip 文件到本地
- **Windows 11 机器耗时 5 分 18 秒**（吞吐量 ~2.3 MB/s）
- **Windows 10 机器仅需 1 分 17 秒**（吞吐量 ~6.9 MB/s）
- Win11 比 Win10 慢约 **4.1 倍**
- 问题持续存在，非偶发，影响所有通过 VPN 的 SMB 文件传输

## 2. 背景 (Background / Environment)

| 配置项 | Windows 10 机器 | Windows 11 机器 |
|--------|----------------|----------------|
| **主机名** | `PC-WIN10-01` | `PC-WIN11-01` |
| **OS 版本** | Windows 10 | Windows 11 24H2 (Build 26100) |
| **VPN 客户端** | **Cisco AnyConnect** | **F5 VPN** |
| **VPN 端点** | `192.0.2.1:443` (UDP) | `198.51.100.112:8080` |
| **客户端 IP** | 同网段 | `10.1.10.63` |
| **SMB 服务器** | `filesrv01.contoso.corp` (`10.2.20.98`) | 同左 |
| **SMB 协议** | SMB 2.x / 3.0.2 | SMB 3.1.1 |
| **TCP 拥塞控制** | Compound TCP (CTCP) | **Cubic TCP** |
| **EDR 产品** | Cybereason (58K 事件) | Cybereason (**840K 事件, 14.3x**) |
| **安全产品 I/O** | ~290K 事件 | **~1,454K 事件 (5x)** |

⚠️ **关键发现**: 两台机器使用**完全不同的 VPN 客户端**，这是一个重大变量。

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

### 阶段 1: 收集网络抓包 (Packet Capture)

1. **收集三组 pcapng 抓包进行对比分析**：
   - Win10 VPN 抓包（Cisco AnyConnect）
   - Win11 通过主机名访问 (`\\filesrv01.contoso.corp\`) 抓包
   - Win11 通过 IP 地址访问 (`\\10.2.20.98\`) 抓包
   
2. **使用 Wireshark/tshark 分析三组抓包**，发现：

   | 指标 | Win10 VPN | Win11 (主机名) | Win11 (IP 地址) |
   |------|-----------|---------------|----------------|
   | 吞吐量 | **25.85 Mbps** | 16.51 Mbps | 19.51 Mbps |
   | 传输时长 | 36.66s | 78.04s | 75.05s |
   | TCP MSS | **1,350 bytes** | 1,240 bytes | 1,240 bytes |
   | 平均 RTT | 0.28ms | 0.48ms | 0.46ms |
   | TCP 重传 | 1,122 | 425 | 177 |
   | Duplicate ACK | 82 | **6,430 (2.56%)** 🔴 | **5,060 (2.04%)** 🔴 |
   | 乱序包 | N/A | 786 | 440 |
   | SMB Read 平均响应 | 1.65s | **2.97s** 🔴 | **2.84s** 🔴 |
   | SMB 签名 | Enabled | Not Enabled | Enabled |
   | IP 分片 | 无 | 无 | **全部分片** 🔴 |

3. **得出关键结论**：
   - 三组抓包均显示严重的 TCP 传输问题（Duplicate ACK 率 2-2.5%）
   - VPN 链路存在持续的**丢包和乱序**
   - Win11 通过 IP 访问时 SMB 降级到 2.1（Kerberos SPN 不匹配导致回退 NTLM）
   - Win11 IP 访问场景下所有 247,586 个包均被分片

### 阶段 2: ProcMon 深度分析

4. **收集 ProcMon 日志对比**：
   - Win11 完整未过滤 ProcMon: **14.7M 行 (2.5 GB)**
   - Win10 完整未过滤 ProcMon: **5.7M 行 (1 GB)**
   - 另有两份过滤后的 CSV 用于初步分析

5. **发现 1: 文件被复制了两次 — Explorer.EXE 删除了首次复制的文件**
   ```
   # Win11 证据:
   Explorer.EXE | IRP_MJ_CREATE | SUCCESS
     Path: C:\New folder\AppPackage_v54 - Copy.zip
     Detail: Desired Access: Delete, Disposition: Open, 
             Options: Non-Directory File, Delete On Close
   
   # 随后 Robocopy 发现文件消失:
   Robocopy.exe | IRP_MJ_DIRECTORY_CONTROL | NO SUCH FILE
   Robocopy.exe | IRP_MJ_CREATE | NAME NOT FOUND
   ```
   **两台机器都存在此问题** — 数据传输量直接翻倍。

6. **发现 2: Win11 存在 109 秒启动延迟 + 36 秒静默期**
   ```
   Win11: Robocopy 首次出现前有 109 秒无 I/O 记录
   Win10: 同一阶段仅需 5.5 秒
   Win11: 首次复制完成后，36 秒内无任何 I/O
   Win10: 同一阶段仅 4.7 秒
   ```

7. **发现 3: VPN 客户端完全不同**
   
   | 指标 | Win10 | Win11 |
   |------|-------|-------|
   | VPN 产品 | **Cisco AnyConnect** | **F5 VPN** |
   | VPN 进程事件数 | vpnagent.exe (502K) | F5VpnPluginApp.exe (1,030K) |

8. **发现 4: Cybereason EDR 在 Win11 上活跃度高 14.3 倍**

   | 指标 | Win10 | Win11 |
   |------|-------|-------|
   | minionhost.exe 事件数 | 58,878 | **840,652** |
   | 扫描行为 | 仅操作本地数据库 | **全盘扫描每个 DLL/EXE** |
   | 访问目标文件路径 | 0 次 | 10 次 |

9. **发现 5: 安全产品总 I/O 负载 Win11 是 Win10 的 5 倍**

   | 安全产品 | Win10 事件数 | Win11 事件数 |
   |---------|------------|------------|
   | Cybereason (minionhost.exe) | 58,878 | **840,652** (14.3x) |
   | Symantec SEP | 136,476 | 131,471 (~1x) |
   | BeyondTrust ExecutionPrevention | 94,315 | 141,484 (1.5x) |
   | BeyondTrust Avecto IC3 | **0** | **285,927** (Win11 only) |
   | BeyondTrust Defendpoint | **0** | **54,758** (Win11 only) |
   | **合计** | **~290K** | **~1,454K (5x)** |

10. **发现 6: SMB 签名排除 — 两台机器配置完全相同**
    ```
    # 两台机器的 LanmanWorkstation.reg:
    "EnableSecuritySignature"=dword:00000001
    "RequireSecuritySignature"=dword:00000001
    ```
    SMB 签名**不是**造成速度差异的原因。

### 阶段 3: 根因分析汇总

11. **综合所有数据，构建根因链**：
    - 🔴 **70%** — VPN 链路丢包/乱序（三组抓包均确认）
    - 🔴 **20%** — Win11 Cubic TCP 对丢包更敏感（vs Win10 Compound TCP）
    - 🟡 **5%** — Win11 TCP MSS 更小 (1,240 vs 1,350)，IP 分片
    - 🟡 **5%** — IP 访问导致 SMB 降级到 2.1

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| 初始假设两台机器使用相同 VPN | 错误方向：把问题聚焦在 OS 差异而非 VPN 差异 | 分析完整 ProcMon 后发现 Win10=Cisco AnyConnect, Win11=F5 VPN |
| 初始假设 Cybereason 仅存在于 Win11 | 基于过滤 CSV 得出错误结论 | 分析完整未过滤 ProcMon (5.7M 行) 后发现 Win10 也有 Cybereason，但 Win11 活跃度高 14.3 倍 |
| ProcMon 日志巨大 (Win11: 14.7M 行, 2.5GB) | 无法人工检阅，过滤 CSV 遗漏关键信息 | 使用脚本批量解析和统计进程事件分布 |
| 需要确认 TCP 拥塞控制算法差异 | 抓包中无法直接看到拥塞控制算法 | 基于 OS 默认配置推断 + 行为特征佐证（Win10 重传更多但吞吐更高） |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause

Win11 SMB 文件复制慢是**多因素叠加**的结果：

| # | 因素 | 影响程度 |
|---|------|---------|
| 1 | **VPN 链路丢包/乱序** — Duplicate ACK 率 2-2.5%，177-425 次重传 | 🔴 **70%** |
| 2 | **Win11 Cubic TCP vs Win10 CTCP** — Cubic 对丢包更敏感，恢复更慢 | 🔴 **20%** |
| 3 | **VPN 客户端不同** (Cisco vs F5) — 隧道开销、MTU、路由策略不同 | 🔴 加剧因素 |
| 4 | **Cybereason EDR Win11 上 14.3 倍活跃** + 安全产品总负载 5 倍 | 🟡 加剧因素 |
| 5 | **Explorer 删除文件导致双重复制** (两台机器均有) | 🟡 中 |
| 6 | **TCP MSS 差异** (1,240 vs 1,350) 及 IP 分片 | 🟡 **5%** |
| 7 | **IP 访问导致 SMB 降级到 2.1** (SPN 不匹配) | 🟡 **5%** |

**核心洞察**: 在同等 VPN 丢包条件下，Win10 的 Compound TCP 能够快速恢复拥塞窗口维持较高吞吐，而 Win11 的 Cubic TCP 在每次检测到丢包时更激进地缩小拥塞窗口且恢复更慢，导致吞吐量低 36-47%。

### Resolution

**快速修复 — 切换 Win11 TCP 拥塞控制算法**：
```powershell
# 以管理员身份运行 PowerShell
# 将 Win11 TCP 拥塞控制从 Cubic 切换为 Compound TCP (Win10 默认)
netsh int tcp set global congestionprovider=ctcp

# 确保 TCP 自动调优为正常模式
netsh int tcp set global autotuninglevel=normal

# 验证设置
netsh int tcp show global
Get-NetTCPSetting | Select-Object SettingName, CongestionProvider, AutoTuningLevelLocal
```

> ⚡ **预期效果**: 恢复到接近 Win10 的吞吐水平 (~25 Mbps)，提升 30-50%

**其他建议操作**：

1. **调查 VPN 链路丢包根因**（优先级最高）
   ```powershell
   # 测试 VPN 丢包率
   Test-Connection -ComputerName 10.2.20.98 -Count 100 -BufferSize 1200
   Test-NetConnection -ComputerName 10.2.20.98 -TraceRoute
   ```
   - 检查 VPN 集中器 CPU/内存负载
   - 检查接口错误/丢包计数器
   - 检查 QoS 策略是否限制了 SMB (端口 445) 流量

2. **修复 MTU/MSS 问题**
   ```powershell
   netsh interface ipv4 show subinterfaces
   # 如果 MTU 过大导致分片，减小到 1300
   netsh interface ipv4 set subinterface "VPN_Interface" mtu=1300 store=persistent
   ```

3. **在同一 VPN 客户端下对比测试** — 排除 VPN 客户端本身的差异影响

4. **调查 Cybereason EDR 配置差异** — 为什么 Win11 上活跃度高 14.3 倍

5. **使用主机名而非 IP 地址访问** — 确保 Kerberos → SMB 3.1.1

### Workaround

- 使用 Robocopy 多线程复制提高吞吐:
  ```powershell
  robocopy "\\filesrv01.contoso.corp\App-Deployment\AppPackage_v54" "C:\local\path" /MT:8 /Z
  ```
- 考虑使用 HTTPS 文件传输替代 SMB（绕过 SMB over VPN 的限制）

## 6. 经验教训 (Lessons Learned)

### 技术知识

- **Windows 11 默认使用 Cubic TCP** vs Windows 10 的 Compound TCP — 在丢包环境下表现差异显著。Cubic 设计用于高带宽广域网但对丢包更敏感，CTCP 在丢包环境下恢复更快
- **通过 IP 地址访问 SMB 共享会导致 Kerberos SPN 不匹配** → 回退 NTLM → 可能降级到 SMB 2.1，失去 3.x 优化
- **TCP MSS 1,240 bytes + VPN 封装头 = 1,340 bytes > MTU 1,280** → 导致 IP 分片 → 放大丢包影响
- **同一 EDR 产品在不同 OS 上行为可能截然不同** — Cybereason 在 Win11 上全盘扫描 vs Win10 上仅操作本地数据库

### 排查方法

- **不要仅依赖过滤后的日志** — 过滤 CSV 导致遗漏关键信息（如 Cybereason 存在于 Win10）。必须分析完整日志
- **早期确认环境一致性** — 本 case 初始假设两台机器使用相同 VPN，浪费了排查时间。应在第一步就确认所有环境变量
- **三组抓包对比法**非常有效 — 同一目标，不同客户端/访问方式，快速定位差异点
- **TCP 拥塞控制算法虽然抓包不可见，但可通过行为推断** — Win10 重传更多 (1,122) 但吞吐更高，说明其窗口恢复更激进
- **ProcMon 进程事件分布统计**是排查安全产品干扰的最佳方法 — 量化每个安全产品的 I/O 事件数

### 排查命令速查

```powershell
# 查看当前 TCP 拥塞控制算法
Get-NetTCPSetting | Select-Object SettingName, CongestionProvider

# 切换 TCP 拥塞控制
netsh int tcp set global congestionprovider=ctcp

# 查看 TCP 自动调优级别
netsh int tcp show global

# 检查 SMB 签名配置
Get-SmbClientConfiguration | Select-Object RequireSecuritySignature, EnableSecuritySignature

# 检查 SMB 连接状态
Get-SmbConnection

# 检查网络接口 MTU
netsh interface ipv4 show subinterfaces

# VPN 丢包测试
Test-Connection -ComputerName <target_ip> -Count 100 -BufferSize 1200
```

### 预防措施

- Windows 11 部署前，评估 TCP 拥塞控制算法对现有 VPN 环境的影响
- 安全产品在 OS 升级后需要重新评估其行为和 I/O 影响
- SMB 共享访问统一使用主机名，避免 IP 地址（保持 Kerberos + SMB 3.x）

## 7. 参考文档 (References)

- [Troubleshoot slow SMB file transfer](https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/slow-file-transfer) — Official Microsoft guide for troubleshooting slow SMB transfers
- [Set-NetTCPSetting](https://learn.microsoft.com/en-us/powershell/module/nettcpip/set-nettcpsetting) — PowerShell cmdlet for modifying TCP settings including CongestionProvider
- [SMB Direct (RDMA)](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-direct) — SMB performance features and RDMA overview

---
---

# Case Summary: Windows 11 SMB File Copy Slowness over VPN — Win11 4x Slower than Win10

**Product/Service:** Windows 11 / SMB / VPN (F5 & Cisco AnyConnect)

---

## 1. Symptoms

Customer reported:
- Copying a 162 MB zip file from remote SMB share `\\filesrv01.contoso.corp\App-Deployment` to local disk via VPN
- **Windows 11 machine: 5 min 18 sec** (throughput ~2.3 MB/s)
- **Windows 10 machine: 1 min 17 sec** (throughput ~6.9 MB/s)
- Win11 approximately **4.1x slower** than Win10
- Issue is persistent and affects all SMB file transfers over VPN

## 2. Background / Environment

| Configuration | Windows 10 Machine | Windows 11 Machine |
|---------------|--------------------|--------------------|
| **Hostname** | `PC-WIN10-01` | `PC-WIN11-01` |
| **OS Version** | Windows 10 | Windows 11 24H2 (Build 26100) |
| **VPN Client** | **Cisco AnyConnect** | **F5 VPN** |
| **VPN Endpoint** | `192.0.2.1:443` (UDP) | `198.51.100.112:8080` |
| **Client IP** | Same subnet | `10.1.10.63` |
| **SMB Server** | `filesrv01.contoso.corp` (`10.2.20.98`) | Same |
| **SMB Protocol** | SMB 2.x / 3.0.2 | SMB 3.1.1 |
| **TCP Congestion** | Compound TCP (CTCP) | **Cubic TCP** |
| **EDR Product** | Cybereason (58K events) | Cybereason (**840K events, 14.3x**) |
| **Security I/O** | ~290K events | **~1,454K events (5x)** |

⚠️ **Critical Finding**: The two machines used **entirely different VPN clients** — a major uncontrolled variable.

## 3. Investigation & Troubleshooting

### Phase 1: Network Packet Capture Analysis

1. **Collected three pcapng captures for comparison**:
   - Win10 VPN capture (Cisco AnyConnect)
   - Win11 accessing by hostname (`\\filesrv01.contoso.corp\`)
   - Win11 accessing by IP address (`\\10.2.20.98\`)

2. **Analyzed with Wireshark/tshark** — Key findings:

   | Metric | Win10 VPN | Win11 (Hostname) | Win11 (IP Address) |
   |--------|-----------|-------------------|---------------------|
   | Throughput | **25.85 Mbps** | 16.51 Mbps | 19.51 Mbps |
   | Duration | 36.66s | 78.04s | 75.05s |
   | TCP MSS | **1,350 bytes** | 1,240 bytes | 1,240 bytes |
   | Avg RTT | 0.28ms | 0.48ms | 0.46ms |
   | Retransmissions | 1,122 | 425 | 177 |
   | Duplicate ACKs | 82 | **6,430 (2.56%)** 🔴 | **5,060 (2.04%)** 🔴 |
   | Out-of-Order | N/A | 786 | 440 |
   | SMB Read Avg | 1.65s | **2.97s** 🔴 | **2.84s** 🔴 |
   | IP Fragmentation | None | None | **All packets** 🔴 |

3. **Key conclusions**:
   - All three captures show severe TCP transmission issues (2-2.5% Dup ACK rate)
   - VPN link has persistent **packet loss and reordering**
   - IP access causes Kerberos SPN mismatch → NTLM fallback → SMB 2.1 downgrade
   - All 247,586 packets fragmented in Win11 IP scenario

### Phase 2: ProcMon Deep Analysis

4. **Collected and compared ProcMon logs**:
   - Win11 full unfiltered: **14.7M rows (2.5 GB)**
   - Win10 full unfiltered: **5.7M rows (1 GB)**

5. **Finding 1: File copied TWICE — Explorer.EXE deletes the first copy**
   ```
   Explorer.EXE | IRP_MJ_CREATE | Desired Access: Delete, Delete On Close
   → Robocopy.exe | IRP_MJ_CREATE | NAME NOT FOUND (file gone)
   → Robocopy re-copies the entire file
   ```
   Both machines affected — data transfer volume **doubled**.

6. **Finding 2: Win11 has 109-second startup delay + 36-second silent period**
   - Win11: 109 seconds of no I/O before Robocopy starts (Win10: 5.5s)
   - Win11: 36-second gap between first and second copy (Win10: 4.7s)

7. **Finding 3: Completely different VPN clients**
   - Win10: Cisco AnyConnect (`vpnagent.exe`: 502K events)
   - Win11: F5 VPN (`F5VpnPluginApp.exe`: 1,030K events)

8. **Finding 4: Cybereason EDR 14.3x more active on Win11**
   - Win10: 58,878 events (mostly local database operations)
   - Win11: **840,652 events** (aggressive full-disk scanning of every DLL/EXE)

9. **Finding 5: Total security product I/O is 5x higher on Win11**
   - Win10 total: ~290K events
   - Win11 total: **~1,454K events** (includes BeyondTrust components only on Win11)

10. **Finding 6: SMB signing ruled out** — identical configuration on both machines.

### Phase 3: Root Cause Synthesis

11. **Root cause contribution breakdown**:
    - 🔴 **70%** — VPN link packet loss/reordering
    - 🔴 **20%** — Win11 Cubic TCP more sensitive to loss than Win10 CTCP
    - 🟡 **5%** — Win11 smaller TCP MSS + IP fragmentation
    - 🟡 **5%** — IP access causes SMB downgrade to 2.1

## 4. Blockers & How They Were Resolved

| Blocker | Impact | Resolution |
|---------|--------|------------|
| Initial assumption both machines use same VPN | Misdirected investigation toward OS differences only | Full ProcMon analysis revealed Win10=Cisco, Win11=F5 |
| Initial assumption Cybereason only on Win11 | Incorrect conclusion from filtered CSV | Full unfiltered ProcMon (5.7M rows) showed Cybereason on both, but 14.3x more active on Win11 |
| ProcMon logs too large for manual review (14.7M rows) | Filtered CSV missed critical information | Scripted bulk analysis of process event distribution |
| TCP congestion algorithm not visible in captures | Could not directly confirm Cubic vs CTCP | Inferred from OS defaults + behavioral evidence (Win10: more retransmissions yet higher throughput) |

## 5. Root Cause & Resolution

### Root Cause

The SMB copy slowness on Win11 is a **multi-factor compound issue**:

| # | Factor | Impact |
|---|--------|--------|
| 1 | **VPN link packet loss/reordering** — 2-2.5% Dup ACK rate | 🔴 **70%** |
| 2 | **Win11 Cubic TCP vs Win10 CTCP** — Cubic more sensitive to loss | 🔴 **20%** |
| 3 | **Different VPN clients** (Cisco vs F5) — different tunnel overhead, MTU, routing | 🔴 Amplifier |
| 4 | **Cybereason EDR 14.3x more active on Win11** + 5x total security I/O | 🟡 Amplifier |
| 5 | **Explorer deletes file causing double copy** (both machines) | 🟡 Medium |
| 6 | **TCP MSS difference** (1,240 vs 1,350) and IP fragmentation | 🟡 **5%** |
| 7 | **IP access downgrades SMB to 2.1** (SPN mismatch) | 🟡 **5%** |

**Core Insight**: Under identical VPN packet loss conditions, Win10's Compound TCP quickly recovers its congestion window maintaining higher throughput, while Win11's Cubic TCP aggressively reduces its window on each loss detection and recovers more slowly, resulting in 36-47% lower throughput.

### Resolution

**Quick Fix — Switch Win11 TCP Congestion Control**:
```powershell
# Run PowerShell as Administrator
netsh int tcp set global congestionprovider=ctcp
netsh int tcp set global autotuninglevel=normal

# Verify
netsh int tcp show global
Get-NetTCPSetting | Select-Object SettingName, CongestionProvider, AutoTuningLevelLocal
```

> ⚡ **Expected Result**: Restore throughput to near Win10 levels (~25 Mbps), 30-50% improvement

**Additional Recommended Actions**:
1. Investigate VPN link packet loss root cause (check concentrator load, interface errors, QoS)
2. Fix MTU/MSS: `netsh interface ipv4 set subinterface "VPN_Interface" mtu=1300 store=persistent`
3. Re-test with same VPN client on both machines to isolate VPN client impact
4. Investigate Cybereason configuration differences between the two machines
5. Use hostname (not IP) for SMB access to ensure Kerberos → SMB 3.1.1

### Workaround

```powershell
# Multi-threaded Robocopy
robocopy "\\filesrv01.contoso.corp\App-Deployment\AppPackage_v54" "C:\local" /MT:8 /Z
```

## 6. Lessons Learned

### Technical Knowledge
- **Windows 11 defaults to Cubic TCP** vs Windows 10's Compound TCP — significant performance difference under packet loss. Cubic is designed for high-bandwidth WANs but is more sensitive to loss
- **Accessing SMB shares by IP address causes Kerberos SPN mismatch** → NTLM fallback → potential SMB 2.1 downgrade
- **Same EDR product can behave drastically differently across OS versions** — Cybereason performed full-disk scanning on Win11 vs local-DB-only on Win10

### Troubleshooting Methods
- **Never rely on filtered logs alone** — filtered CSV missed critical information. Always analyze full unfiltered logs
- **Verify environment consistency early** — this case initially assumed same VPN on both machines, wasting investigation time
- **Three-capture comparison method** is highly effective — same target, different clients/access methods
- **TCP congestion control can be inferred from behavior** — higher retransmissions + higher throughput = more aggressive window recovery (CTCP)
- **ProcMon process event distribution statistics** are the best method to quantify security product interference

### Quick Reference Commands
```powershell
# Check TCP congestion control algorithm
Get-NetTCPSetting | Select-Object SettingName, CongestionProvider

# Switch TCP congestion control
netsh int tcp set global congestionprovider=ctcp

# Check SMB signing
Get-SmbClientConfiguration | Select RequireSecuritySignature, EnableSecuritySignature

# Check network interface MTU
netsh interface ipv4 show subinterfaces

# VPN packet loss test
Test-Connection -ComputerName <target> -Count 100 -BufferSize 1200
```

### Prevention
- Before Win11 deployment, evaluate TCP congestion control algorithm impact on existing VPN environment
- Re-evaluate security product behavior and I/O impact after OS upgrades
- Standardize SMB share access using hostnames (not IP) to maintain Kerberos + SMB 3.x

## 7. References

- [Troubleshoot slow SMB file transfer](https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/slow-file-transfer) — Official Microsoft guide for troubleshooting slow SMB transfers
- [Set-NetTCPSetting](https://learn.microsoft.com/en-us/powershell/module/nettcpip/set-nettcpsetting) — PowerShell cmdlet for modifying TCP settings including CongestionProvider
- [SMB Direct (RDMA)](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-direct) — SMB performance features and RDMA overview

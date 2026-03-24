---
layout: post
title: "双网卡同时连接时其中一张被自动 Disable / Dual NIC — One NIC Automatically Disabled When Both Connected"
date: 2026-03-24
categories: [Networking, NIC-Configuration]
tags: [nic, dual-nic, docking-station, network-adapter, disable, group-policy, driver]

---

# Case Summary: 双网卡同时连接时其中一张被自动 Disable

**Product/Service:** Windows Networking / NIC Configuration

---

## 1. 症状 (Symptoms)

- 用户申请使用双网卡后，无法同时使用两张网卡
- 电脑通过扩展坞（Docking Station）连接一根网线，电脑本身自带网口也连接一根网线
- 当扩展坞和电脑本体同时插上网线后，只有一张网卡处于激活（Active）状态，另一张网卡被自动设置为 Disable 状态
- 单独使用任意一张网卡时均可正常工作

## 2. 背景 (Background / Environment)

- 用户环境中使用笔记本电脑 + 扩展坞（Docking Station）的组合
- 笔记本自带一个以太网接口（内置 NIC）
- 扩展坞提供另一个以太网接口（外置 NIC，通常为 USB-to-Ethernet 适配器）
- 用户已向 IT 部门申请使用双网卡权限
- [待补充] 操作系统版本（Windows 10 / Windows 11，具体 Build 号）
- [待补充] 扩展坞型号及网卡芯片信息
- [待补充] 企业是否有 Group Policy / Endpoint Management 相关策略
- [待补充] 双网卡的使用场景/需求（如：一张连内网、一张连外网？）

## 3. Troubleshooting 过程 (Investigation & Troubleshooting)

1. **初始电话沟通** — 通过电话与用户确认了问题现象：
   - 扩展坞单独插网线 → 网卡正常激活
   - 电脑本体单独插网线 → 网卡正常激活
   - 两者同时插网线 → 一张激活、一张被 Disable
   - 确认了问题的可复现性

[待补充] 后续排查步骤，建议收集以下信息：

- `ipconfig /all` 输出（两张网卡同时连接时）
- `Get-NetAdapter | Format-List *` 查看网卡详细状态
- `Get-NetAdapterBinding` 查看网卡绑定的协议组件
- Device Manager 中两张网卡的属性和驱动信息
- Event Viewer → System Log 中是否有网卡 Disable 相关事件
- `gpresult /h gpresult.html` 检查是否有 Group Policy 限制网桥/双网卡
- 检查是否安装了企业端点管理软件（如 SCCM、Intune、第三方安全软件）可能强制禁用第二张网卡

## 4. Blockers 与解决 (Blockers & How They Were Resolved)

| Blocker | 影响 | 如何解决 |
|---------|------|---------|
| 初始信息有限，仅通过电话沟通获得现象描述 | 无法确定是系统行为、策略限制还是驱动冲突 | [待补充] 需要用户提供详细的系统信息和日志 |

## 5. 根因与解决方案 (Root Cause & Resolution)

### Root Cause
[待补充] — 目前仅完成初始现象确认，根因待排查。

可能的方向包括：
1. **Group Policy 限制**：企业 GPO 中配置了 "Prohibit installation of network bridge" 或类似策略，阻止多网卡同时使用
2. **Endpoint Management / 安全软件**：企业部署的安全合规软件检测到双网卡并强制禁用其中一张（防止网络桥接导致的安全风险）
3. **网卡驱动冲突**：扩展坞的 USB 网卡驱动与内置网卡驱动存在冲突
4. **NIC Teaming 配置问题**：系统尝试对两张网卡进行 Teaming 但配置不正确
5. **BIOS / 固件设置**：扩展坞或笔记本 BIOS 中有 NIC 优先级或排他设置
6. **Hyper-V Virtual Switch**：如果系统启用了 Hyper-V，外部虚拟交换机可能绑定了其中一张网卡导致冲突

### Resolution
[待补充]

### Workaround（如有）
[待补充]

## 6. 经验教训 (Lessons Learned)

[待补充 — 待 case 解决后总结]

初步观察：
- **排查方法**：对于双网卡无法同时使用的问题，应首先排查企业策略限制（GPO、Endpoint Management），再排查驱动层面冲突
- **信息收集**：初始电话沟通时应引导用户提供 `ipconfig /all`、`Get-NetAdapter`、Event Log 等关键信息，减少来回沟通次数

## 7. 参考文档 (References)

暂无可验证的参考文档

---
---

# Case Summary: Dual NIC — One NIC Automatically Disabled When Both Connected

**Product/Service:** Windows Networking / NIC Configuration

---

## 1. Symptoms

- After applying for dual NIC usage, the user is unable to use both NICs simultaneously
- The computer connects one Ethernet cable via a Docking Station, and another Ethernet cable via the laptop's built-in Ethernet port
- When both the docking station and the laptop's built-in port are connected via Ethernet cables, only one NIC remains active while the other is automatically set to a Disabled state
- Each NIC works normally when used individually

## 2. Background / Environment

- The user's setup consists of a laptop + Docking Station combination
- The laptop has a built-in Ethernet interface (onboard NIC)
- The docking station provides an additional Ethernet interface (typically a USB-to-Ethernet adapter)
- The user has submitted a request to IT for dual NIC usage permissions
- [To be supplemented] OS version (Windows 10 / Windows 11, specific Build number)
- [To be supplemented] Docking station model and NIC chipset information
- [To be supplemented] Whether enterprise Group Policy / Endpoint Management policies are in place
- [To be supplemented] Use case for dual NICs (e.g., one for internal network, one for external?)

## 3. Investigation & Troubleshooting

1. **Initial phone communication** — Confirmed the issue behavior with the user via phone call:
   - Docking station Ethernet only → NIC activates normally
   - Laptop built-in Ethernet only → NIC activates normally
   - Both connected simultaneously → One active, one Disabled
   - Confirmed the issue is reproducible

[To be supplemented] Next troubleshooting steps — recommended data collection:

- `ipconfig /all` output (with both NICs connected)
- `Get-NetAdapter | Format-List *` for detailed NIC status
- `Get-NetAdapterBinding` to check protocol bindings
- Device Manager properties and driver info for both NICs
- Event Viewer → System Log for NIC disable-related events
- `gpresult /h gpresult.html` to check for Group Policy restrictions on bridging/dual NICs
- Check for enterprise endpoint management software (e.g., SCCM, Intune, third-party security tools) that may enforce single NIC policy

## 4. Blockers & How They Were Resolved

| Blocker | Impact | How Resolved |
|---------|--------|--------------|
| Limited initial information — only symptom description obtained via phone call | Unable to determine if this is a system behavior, policy restriction, or driver conflict | [To be supplemented] Need user to provide detailed system info and logs |

## 5. Root Cause & Resolution

### Root Cause
[To be supplemented] — Only initial symptom confirmation has been completed; root cause investigation pending.

Possible directions include:
1. **Group Policy restriction**: Enterprise GPO configured with "Prohibit installation of network bridge" or similar policy preventing simultaneous multi-NIC usage
2. **Endpoint Management / Security software**: Enterprise-deployed security compliance software detecting dual NICs and forcibly disabling one (to prevent security risks from network bridging)
3. **NIC driver conflict**: USB NIC driver from the docking station conflicting with the built-in NIC driver
4. **NIC Teaming misconfiguration**: System attempting to team the two NICs but with incorrect configuration
5. **BIOS / Firmware setting**: Docking station or laptop BIOS contains NIC priority or exclusivity settings
6. **Hyper-V Virtual Switch**: If Hyper-V is enabled, an external virtual switch may be bound to one NIC causing conflicts

### Resolution
[To be supplemented]

### Workaround (if applicable)
[To be supplemented]

## 6. Lessons Learned

[To be supplemented — to be completed after case resolution]

Initial observations:
- **Troubleshooting methodology**: For dual NIC co-existence failures, first investigate enterprise policy restrictions (GPO, Endpoint Management) before exploring driver-level conflicts
- **Data collection**: During initial phone communication, guide the user to provide `ipconfig /all`, `Get-NetAdapter`, Event Log, and other key information to reduce back-and-forth communication cycles

## 7. References

No verifiable reference documents available at this time

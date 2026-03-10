---
layout: post
title: "TSG: Windows Map Network Drive Issues"
date: 2026-03-10
categories: [TSG, SMB]
tags: [tsg, map-drive, net-use, smb, credential-manager, smbglobalmapping, network-drive, file-explorer]
type: "tsg"
---

# TSG: Windows 映射网络驱动器问题排查指南

**Product/Service:** Windows (File Explorer / net use / SmbGlobalMapping)
**Category:** SMB / Network Drive Mapping
**Severity Guidance:** P2-重要（影响用户日常文件访问）/ P3-一般（配置指导类）
**Last Updated:** 2026-03-10

---

## 1. 适用场景 (When to Use This TSG)

当客户报告与映射网络驱动器相关的问题时，使用本 TSG：

- **典型症状**：
  - 映射的网络驱动器在重启后消失
  - 映射的网络驱动器重启后显示红叉（❌），无法访问
  - 映射网络驱动器时弹出"访问被拒绝"或凭据窗口
  - 不同用户看到不同的映射驱动器（或看不到）
  - `net use` 命令结果不一致（有时持久化，有时不持久化）
  - SmbGlobalMapping 创建后其他用户无法使用

- **适用条件**：
  - Windows 10 / Windows 11 / Windows Server 2016+
  - 使用 File Explorer GUI、`net use` 命令或 `New-SmbGlobalMapping` 映射驱动器
  - SMB 共享目标为标准 SMB 服务器（包括 Windows Server、NAS、Azure Files 等）

- **不适用场景**：
  - DFS Namespace 相关问题（SmbGlobalMapping 不支持 DFS）
  - SMB 性能问题（参考 SMB 性能 TSG）
  - 共享权限配置问题（NTFS/Share Permission 本身的问题）
  - VPN 场景下 SMB 连接中断（网络层问题）

## 2. 快速检查清单 (Quick Checklist)

在深入排查之前，先确认以下基本项：

- [ ] **确认映射方法** — 客户使用的是 File Explorer GUI、`net use` 还是 `New-SmbGlobalMapping`？
- [ ] **确认映射参数** — 是否勾选了 "Reconnect at sign-in" / 使用了 `/persistent:yes` / 设置了 `-Persistent $true`？
- [ ] **确认凭据保存** — 是否勾选了 "Remember my credentials" / 使用了 `/savecred` / 凭据是否保存在 Credential Manager 中？
- [ ] **确认 UNC 路径可达** — 在 File Explorer 中直接输入 `\\<server>\<share>` 是否可以访问？
- [ ] **确认用户身份** — 登录账户是域账户还是本地账户？该账户是否有权限访问目标共享？
- [ ] **确认注册表** — 对应的注册表键是否存在？（见数据收集部分）

> 如果 UNC 路径本身不可达，问题不在映射驱动器层面，需先排查网络连接和 SMB 访问。

## 3. 数据收集 (Data Collection)

### 必须收集

| 数据项 | 收集命令/方法 | 说明 |
|--------|-------------|------|
| 当前映射状态 | `net use` | 查看当前所有映射驱动器及其状态 |
| Per-User 注册表 | `reg query HKCU\Network` | 查看通过 GUI/net use 保存的持久化映射信息 |
| Per-Machine 注册表 | `reg query "HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings"` | 查看 SmbGlobalMapping 的持久化信息 |
| Credential Manager | `cmdkey /list` | 列出所有保存的凭据 |
| 当前登录用户 | `whoami /all` | 确认登录身份和权限 |
| SMB 连接状态 | `Get-SmbConnection` | 查看当前 SMB 连接详情 |

### 可选收集（根据情况）

| 数据项 | 收集命令/方法 | 适用场景 |
|--------|-------------|---------|
| SmbGlobalMapping 列表 | `Get-SmbGlobalMapping` | 使用 SmbGlobalMapping 方式时 |
| 特定驱动器注册表详情 | `reg query HKCU\Network\<DriveLetter>` | 需要查看某个映射的详细配置 |
| 共享权限 | `Get-SmbShareAccess -Name <ShareName>`（在服务器端执行） | 怀疑权限问题时 |
| 网络凭据文件 | `dir /a C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials` | 排查 SmbGlobalMapping 凭据时 |

## 4. 诊断决策树 (Diagnostic Decision Tree)

### Step 1: 确认映射方法

**询问客户或检查：**

```powershell
# 检查 per-user 映射（File Explorer / net use）
reg query HKCU\Network 2>$null

# 检查 per-machine 映射（SmbGlobalMapping）
reg query "HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings" 2>$null

# 检查当前映射
net use
```

**判断：**
- 如果 `HKCU\Network` 下有对应驱动器号 → 使用的是 **File Explorer / net use**（Per-User），继续 Step 2
- 如果 `HKLM\...\GlobalMappings` 下有对应驱动器号 → 使用的是 **SmbGlobalMapping**（Per-Machine），跳到 Step 5
- 如果两处都没有 → 映射未持久化保存，跳到 **解决方案 A**

### Step 2: 检查 Per-User 映射的注册表详情

**检查方法：**
```powershell
# 替换 L 为实际驱动器号
reg query HKCU\Network\L
```

**预期输出 vs 异常输出：**
- ✅ 正常：存在 `RemotePath`、`UserName`、`ProviderName` 等键值 → 继续 Step 3
- ❌ 异常：注册表键不存在 → 映射信息丢失，转到 **解决方案 A**
- ❌ 异常：`RemotePath` 指向错误路径 → 转到 **解决方案 B**

### Step 3: 检查凭据是否保存

**检查方法：**
```powershell
# 列出保存的凭据，查找目标服务器
cmdkey /list
```

**判断：**
- ✅ 存在目标服务器的凭据条目 → 继续 Step 4
- ❌ 没有目标服务器的凭据 → 转到 **解决方案 C**

### Step 4: 验证凭据是否有效

**检查方法：**
```powershell
# 尝试直接访问 UNC 路径
Test-Path "\\<server>\<share>"

# 或在 File Explorer 中直接输入 \\<server>\<share>
```

**判断：**
- ✅ 可以正常访问 → 映射应该正常工作。如果仍有问题，转到 **解决方案 D**
- ❌ 弹出凭据窗口或访问被拒绝 → 保存的凭据已失效，转到 **解决方案 C**
- ❌ 路径不可达（网络错误）→ 非映射问题，需排查网络连接和 SMB 服务

### Step 5: 检查 SmbGlobalMapping 状态

**检查方法：**
```powershell
Get-SmbGlobalMapping
```

**判断：**
- ✅ 映射存在且 Status 为 OK → 如果仍有问题，转到 **解决方案 E**
- ❌ 映射不存在 → 转到 **解决方案 F**
- ❌ 映射存在但 Status 异常 → 转到 **解决方案 F**

> 决策树遵循"先确认方法 → 再检查持久化 → 再验证凭据"的逻辑顺序。

## 5. 解决方案 (Solutions)

### 解决方案 A: 映射未持久化（重启后驱动器消失）

**根因说明：**
映射网络驱动器时未启用持久化选项，映射信息未保存到注册表，因此系统重启后映射关系丢失。

**修复步骤：**

**方法一：File Explorer GUI**
1. 打开 File Explorer → 右键 "This PC" → "Map Network Drive"
2. 选择驱动器号，输入共享路径 `\\<server>\<share>`
3. **勾选** "Reconnect at sign-in" ✅
4. 如需使用其他凭据，勾选 "Connect using different credentials"
5. 点击 Finish，在凭据窗口中 **勾选** "Remember my credentials" ✅

**方法二：net use 命令**
```powershell
# 先删除已有映射（如有）
net use L: /delete

# 使用持久化参数重新映射
# 方式 1：使用 /savecred（系统会提示输入凭据）
net use L: \\<server>\<share> /persistent:yes /savecred

# 方式 2：先保存凭据，再映射
cmdkey /add:<server> /user:<domain\username> /pass:<password>
net use L: \\<server>\<share> /persistent:yes /savecred
```

**验证方法：**
```powershell
# 确认注册表已保存
reg query HKCU\Network\L

# 确认凭据已保存
cmdkey /list | findstr /i "<server>"
```
预期结果：注册表中存在 `RemotePath` 键值，Credential Manager 中存在对应凭据。

**风险提示：**
- `/savecred` 和 `/user:` 参数不能在同一行 `net use` 命令中同时使用，否则会报冲突错误
- `/persistent` 参数未指定时，会继承该机器上上次使用的值，这可能导致不同设备上的行为不一致

### 解决方案 B: 注册表中的 RemotePath 错误

**根因说明：**
注册表中保存的共享路径不正确，可能由于服务器更名、共享路径变更等原因导致。

**修复步骤：**
1. 删除现有映射：
   ```powershell
   net use L: /delete
   ```
2. 使用正确路径重新映射：
   ```powershell
   net use L: \\<correct-server>\<correct-share> /persistent:yes /savecred
   ```

**验证方法：**
```powershell
reg query HKCU\Network\L /v RemotePath
```
预期结果：`RemotePath` 显示正确的 UNC 路径。

### 解决方案 C: 凭据丢失或失效（驱动器显示红叉）

**根因说明：**
映射路径信息保存在注册表中，但 Credential Manager 中的凭据丢失或已过期。系统重启后尝试恢复连接时无法认证，导致驱动器显示红叉（❌）。

常见凭据丢失原因：
- 映射时未勾选 "Remember my credentials"
- 凭据被手动从 Credential Manager 删除
- 密码已更改导致保存的凭据失效
- 组策略清除了保存的凭据

**修复步骤：**
1. 更新保存的凭据：
   ```powershell
   # 删除旧凭据（如有）
   cmdkey /delete:<server>

   # 添加新凭据
   cmdkey /add:<server> /user:<domain\username> /pass:<password>
   ```
2. 测试访问：
   ```powershell
   net use L: /delete
   net use L: \\<server>\<share> /persistent:yes /savecred
   ```

**验证方法：**
```powershell
cmdkey /list | findstr /i "<server>"
Test-Path "L:\"
```
预期结果：Credential Manager 中存在凭据，驱动器可正常访问。

**风险提示：** 如果目标共享使用 NTLM 认证，且登录账户为本地账户，当本地账户密码恰好与映射凭据密码相同时，即使未保存凭据也可能认证成功。这是 NTLM 的特殊行为，不应依赖此机制。

### 解决方案 D: 注册表和凭据均正常但驱动器仍无法使用

**根因说明：**
映射配置完整，但可能存在网络层面或 SMB 会话层面的问题。

**修复步骤：**
1. 断开并重新连接：
   ```powershell
   net use L: /delete
   net use L: \\<server>\<share> /persistent:yes
   ```
2. 如果失败，清除 SMB 会话缓存：
   ```powershell
   # 清除到目标服务器的现有会话
   net use \\<server> /delete

   # 重新映射
   net use L: \\<server>\<share> /persistent:yes /savecred
   ```
3. 检查 SMB 客户端服务：
   ```powershell
   Get-Service LanmanWorkstation
   # 如果服务未运行
   Start-Service LanmanWorkstation
   ```

**验证方法：**
```powershell
net use
Get-SmbConnection | Where-Object { $_.ServerName -eq "<server>" }
```

### 解决方案 E: SmbGlobalMapping 存在但用户无法使用

**根因说明：**
SmbGlobalMapping 是 Per-Machine 级别的映射，主要用于 Windows Containers 场景。需要注意：
- SmbGlobalMapping **不支持** DFS Namespace folder shares
- 支持 standalone 和 failover cluster SMB shares

**修复步骤：**
1. 确认当前 SmbGlobalMapping 状态：
   ```powershell
   Get-SmbGlobalMapping
   ```
2. 确认是否为 DFS 路径（如果是，则不支持，需改用 `net use`）
3. 如需重建映射：
   ```powershell
   Remove-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Force
   $creds = Get-Credential
   New-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Credential $creds -LocalPath "Z:" -Persistent $true
   ```

**验证方法：**
```powershell
Get-SmbGlobalMapping
Test-Path "Z:\"
```

### 解决方案 F: SmbGlobalMapping 重启后消失

**根因说明：**
创建 SmbGlobalMapping 时未设置 `-Persistent $true`，或凭据文件损坏。

SmbGlobalMapping 的持久化信息保存在：
- 注册表：`HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings\<DriveLetter>`
- 凭据文件：`C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials`

**修复步骤：**
1. 重新创建映射并确保持久化：
   ```powershell
   $User = "<Domain\UserName>"
   $PWord = ConvertTo-SecureString -String '<password>' -AsPlainText -Force
   $Creds = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $User, $PWord
   New-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Credential $Creds -LocalPath "H:" -Persistent $true
   ```

**验证方法：**
```powershell
# 确认注册表已保存
reg query "HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings\H"

# 确认凭据文件存在
dir /a "C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials"
```

**风险提示：**
- 如果不指定 `-Persistent` 参数，默认会继承上次使用的值
- 脚本中明文保存密码有安全风险，生产环境建议使用 Azure Key Vault 或其他安全存储

### Workaround

如果持久化映射反复出现问题，可以考虑通过 **登录脚本（Logon Script）** 或 **计划任务（Scheduled Task）** 在用户登录时自动执行映射命令：

```powershell
# 登录脚本示例
net use L: \\<server>\<share> /persistent:no /user:<domain\username> <password>
```

> 此方案的缺点是密码可能需要硬编码在脚本中，需注意安全性。

## 6. 升级指引 (Escalation Guidance)

当以下条件满足时，应考虑升级：

- 注册表和凭据均正确配置，但映射驱动器仍无法在重启后恢复——可能涉及 SMB 客户端或 MUP/MRxSMB 驱动问题
- SmbGlobalMapping 持久化后凭据文件反复丢失——可能涉及 lsass.exe 凭据存储问题
- 大规模部署场景下映射行为不一致——可能涉及 Group Policy 或域环境配置
- NTLM 认证行为异常——可能涉及安全策略或认证协议问题

**升级时需提供的信息：**
- 已完成的排查步骤和结果（按本 TSG 逐步记录）
- `net use`、`reg query HKCU\Network`、`cmdkey /list` 的输出
- 映射方式及具体参数
- 问题是否可稳定复现，复现步骤
- 客户环境信息（OS 版本、域/工作组、认证方式）

## 7. 相关知识 (Related Knowledge)

### 三种映射方法对比

| 特性 | File Explorer GUI | net use | SmbGlobalMapping |
|------|------------------|---------|-----------------|
| 作用范围 | Per-User | Per-User | Per-Machine |
| 持久化注册表 | `HKCU\Network\<Drive>` | `HKCU\Network\<Drive>` | `HKLM\...\GlobalMappings\<Drive>` |
| 凭据存储 | Credential Manager | Credential Manager | `NetworkService\Credentials` |
| DFS 支持 | ✅ | ✅ | ❌ |
| Failover Cluster 支持 | ✅ | ✅ | ✅ |
| 主要用途 | 终端用户 | 脚本/批处理 | Windows Containers |

### 关键注册表路径说明

- `HKCU\Network\<DriveLetter>` — 包含 `RemotePath`（共享路径）、`UserName`（映射凭据账户，可能为空表示使用登录凭据）、`ProviderName` 等
- `HKLM\...\GlobalMappings\<DriveLetter>` — SmbGlobalMapping 的持久化信息

### 常见误区

- **误区 1**：认为 `net use` 不加 `/persistent` 参数默认不持久化 → 实际会继承上次使用的值
- **误区 2**：认为 `/savecred` 和 `/user:` 可以一起使用 → 会报冲突错误，需分开操作
- **误区 3**：认为 SmbGlobalMapping 适用于所有场景 → 不支持 DFS Namespace
- **误区 4**：本地账户密码相同时认证"意外成功" → 这是 NTLM 认证的特殊行为，不应依赖

## 8. 参考资料 (References)

- [New-SmbGlobalMapping (SmbShare) | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/smbshare/new-smbglobalmapping) — SmbGlobalMapping cmdlet 官方文档
- [Net use | Microsoft Learn](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/gg651155(v=ws.11)) — Net use 命令参考文档

## 修订历史 (Revision History)

| 日期 | 版本 | 修改内容 | 来源 |
|------|------|---------|------|
| 2026-03-10 | 1.0 | 初始版本 | 基于 Windows Map Network Drive 技术文档创建 |

---

# TSG: Windows Map Network Drive Issues

**Product/Service:** Windows (File Explorer / net use / SmbGlobalMapping)
**Category:** SMB / Network Drive Mapping
**Severity Guidance:** P2-Important (impacts daily file access) / P3-Normal (configuration guidance)
**Last Updated:** 2026-03-10

---

## 1. When to Use This TSG

Use this TSG when customers report issues related to mapped network drives:

- **Typical Symptoms:**
  - Mapped network drive disappears after reboot
  - Mapped network drive shows a red cross (❌) and is inaccessible after reboot
  - "Access denied" error or credential prompt when mapping a drive
  - Different users see different mapped drives (or none at all)
  - Inconsistent `net use` behavior (sometimes persistent, sometimes not)
  - SmbGlobalMapping created but other users cannot access it

- **Applicable Conditions:**
  - Windows 10 / Windows 11 / Windows Server 2016+
  - Drive mapped via File Explorer GUI, `net use` command, or `New-SmbGlobalMapping`
  - SMB share target is a standard SMB server (Windows Server, NAS, Azure Files, etc.)

- **Not Applicable:**
  - DFS Namespace issues (SmbGlobalMapping does not support DFS)
  - SMB performance issues (refer to SMB performance TSG)
  - Share permission configuration issues (NTFS/Share Permission problems)
  - VPN-related SMB connection drops (network layer issues)

## 2. Quick Checklist

Before deep investigation, verify these basics:

- [ ] **Confirm mapping method** — Did the customer use File Explorer GUI, `net use`, or `New-SmbGlobalMapping`?
- [ ] **Confirm mapping parameters** — Was "Reconnect at sign-in" checked / `/persistent:yes` used / `-Persistent $true` set?
- [ ] **Confirm credential saving** — Was "Remember my credentials" checked / `/savecred` used / Are credentials in Credential Manager?
- [ ] **Confirm UNC path reachability** — Can `\\<server>\<share>` be accessed directly in File Explorer?
- [ ] **Confirm user identity** — Is the logon account a domain or local account? Does it have permission to access the target share?
- [ ] **Confirm registry** — Does the corresponding registry key exist? (See Data Collection section)

> If the UNC path itself is unreachable, the issue is not at the drive mapping layer — investigate network connectivity and SMB access first.

## 3. Data Collection

### Required

| Data Item | Collection Command | Purpose |
|-----------|-------------------|---------|
| Current mapping status | `net use` | View all mapped drives and their status |
| Per-User registry | `reg query HKCU\Network` | View persistent mapping info saved via GUI/net use |
| Per-Machine registry | `reg query "HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings"` | View SmbGlobalMapping persistence info |
| Credential Manager | `cmdkey /list` | List all saved credentials |
| Current logon user | `whoami /all` | Confirm logon identity and privileges |
| SMB connection status | `Get-SmbConnection` | View current SMB connection details |

### Optional (Situational)

| Data Item | Collection Command | When Needed |
|-----------|-------------------|-------------|
| SmbGlobalMapping list | `Get-SmbGlobalMapping` | When SmbGlobalMapping method is used |
| Specific drive registry details | `reg query HKCU\Network\<DriveLetter>` | When detailed config of a specific mapping is needed |
| Share permissions | `Get-SmbShareAccess -Name <ShareName>` (run on server) | When permission issues are suspected |
| Network credential files | `dir /a C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials` | When investigating SmbGlobalMapping credentials |

## 4. Diagnostic Decision Tree

### Step 1: Identify the Mapping Method

**Ask the customer or check:**

```powershell
# Check per-user mappings (File Explorer / net use)
reg query HKCU\Network 2>$null

# Check per-machine mappings (SmbGlobalMapping)
reg query "HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings" 2>$null

# Check current mappings
net use
```

**Decision:**
- If `HKCU\Network` contains the drive letter → **File Explorer / net use** (Per-User) → Continue to Step 2
- If `HKLM\...\GlobalMappings` contains the drive letter → **SmbGlobalMapping** (Per-Machine) → Jump to Step 5
- If neither location has the entry → Mapping was not persisted → Go to **Solution A**

### Step 2: Check Per-User Mapping Registry Details

**How to check:**
```powershell
# Replace L with the actual drive letter
reg query HKCU\Network\L
```

**Expected vs Abnormal Output:**
- ✅ Normal: Contains `RemotePath`, `UserName`, `ProviderName` keys → Continue to Step 3
- ❌ Abnormal: Registry key does not exist → Mapping info lost → Go to **Solution A**
- ❌ Abnormal: `RemotePath` points to wrong path → Go to **Solution B**

### Step 3: Check if Credentials Are Saved

**How to check:**
```powershell
# List saved credentials and look for the target server
cmdkey /list
```

**Decision:**
- ✅ Credential entry exists for the target server → Continue to Step 4
- ❌ No credential for the target server → Go to **Solution C**

### Step 4: Validate Credential Effectiveness

**How to check:**
```powershell
# Try to access the UNC path directly
Test-Path "\\<server>\<share>"

# Or type \\<server>\<share> directly in File Explorer
```

**Decision:**
- ✅ Access succeeds normally → Mapping should work. If still failing → Go to **Solution D**
- ❌ Credential prompt or access denied → Saved credentials are invalid → Go to **Solution C**
- ❌ Path unreachable (network error) → Not a mapping issue, investigate network and SMB service

### Step 5: Check SmbGlobalMapping Status

**How to check:**
```powershell
Get-SmbGlobalMapping
```

**Decision:**
- ✅ Mapping exists and Status is OK → If still failing → Go to **Solution E**
- ❌ Mapping does not exist → Go to **Solution F**
- ❌ Mapping exists but Status is abnormal → Go to **Solution F**

## 5. Solutions

### Solution A: Mapping Not Persisted (Drive Disappears After Reboot)

**Root Cause:**
The persistence option was not enabled when mapping the network drive. The mapping information was not saved to the registry, so it is lost after system reboot.

**Fix Steps:**

**Option 1: File Explorer GUI**
1. Open File Explorer → Right-click "This PC" → "Map Network Drive"
2. Select a drive letter, enter the share path `\\<server>\<share>`
3. **Check** "Reconnect at sign-in" ✅
4. If using different credentials, check "Connect using different credentials"
5. Click Finish, and in the credential prompt **check** "Remember my credentials" ✅

**Option 2: net use command**
```powershell
# Delete existing mapping if any
net use L: /delete

# Remap with persistence
# Method 1: Use /savecred (system will prompt for credentials)
net use L: \\<server>\<share> /persistent:yes /savecred

# Method 2: Save credentials first, then map
cmdkey /add:<server> /user:<domain\username> /pass:<password>
net use L: \\<server>\<share> /persistent:yes /savecred
```

**Verification:**
```powershell
# Confirm registry is saved
reg query HKCU\Network\L

# Confirm credentials are saved
cmdkey /list | findstr /i "<server>"
```
Expected: Registry contains `RemotePath` key, Credential Manager shows the credential entry.

**Risk Note:**
- `/savecred` and `/user:` parameters **cannot** be used together in the same `net use` command — this causes a conflict error
- If `/persistent` is omitted, it inherits the last used value on the machine, which may cause inconsistent behavior across devices

### Solution B: Incorrect RemotePath in Registry

**Root Cause:**
The share path saved in the registry is incorrect, possibly due to server rename, share path change, etc.

**Fix Steps:**
1. Delete the existing mapping:
   ```powershell
   net use L: /delete
   ```
2. Remap with the correct path:
   ```powershell
   net use L: \\<correct-server>\<correct-share> /persistent:yes /savecred
   ```

**Verification:**
```powershell
reg query HKCU\Network\L /v RemotePath
```
Expected: `RemotePath` shows the correct UNC path.

### Solution C: Credentials Lost or Expired (Drive Shows Red Cross)

**Root Cause:**
The mapping path information is saved in the registry, but the credentials in Credential Manager are missing or expired. The system fails to authenticate when attempting to restore the connection after reboot, resulting in a red cross (❌) on the drive.

Common causes of credential loss:
- "Remember my credentials" was not checked during mapping
- Credentials were manually deleted from Credential Manager
- Password was changed, invalidating saved credentials
- Group Policy cleared saved credentials

**Fix Steps:**
1. Update saved credentials:
   ```powershell
   # Delete old credentials (if any)
   cmdkey /delete:<server>

   # Add new credentials
   cmdkey /add:<server> /user:<domain\username> /pass:<password>
   ```
2. Test access:
   ```powershell
   net use L: /delete
   net use L: \\<server>\<share> /persistent:yes /savecred
   ```

**Verification:**
```powershell
cmdkey /list | findstr /i "<server>"
Test-Path "L:\"
```
Expected: Credential Manager shows the entry, drive is accessible.

**Risk Note:** If the target share uses NTLM authentication and the logon account is a local account, authentication may succeed even without saved credentials if the local account password happens to match the mapped credential password. This is a special NTLM behavior and should not be relied upon.

### Solution D: Registry and Credentials Are Fine but Drive Still Fails

**Root Cause:**
The mapping configuration is complete, but there may be network-level or SMB session-level issues.

**Fix Steps:**
1. Disconnect and reconnect:
   ```powershell
   net use L: /delete
   net use L: \\<server>\<share> /persistent:yes
   ```
2. If that fails, clear the SMB session cache:
   ```powershell
   # Clear existing sessions to the target server
   net use \\<server> /delete

   # Remap
   net use L: \\<server>\<share> /persistent:yes /savecred
   ```
3. Check the SMB client service:
   ```powershell
   Get-Service LanmanWorkstation
   # If the service is not running
   Start-Service LanmanWorkstation
   ```

**Verification:**
```powershell
net use
Get-SmbConnection | Where-Object { $_.ServerName -eq "<server>" }
```

### Solution E: SmbGlobalMapping Exists but Users Cannot Use It

**Root Cause:**
SmbGlobalMapping is a per-machine level mapping, primarily designed for Windows Containers scenarios. Key limitations:
- SmbGlobalMapping **does not support** DFS Namespace folder shares
- It supports standalone and failover cluster SMB shares

**Fix Steps:**
1. Verify the current SmbGlobalMapping status:
   ```powershell
   Get-SmbGlobalMapping
   ```
2. Confirm whether the path is a DFS path (if so, it's not supported — use `net use` instead)
3. If rebuilding is needed:
   ```powershell
   Remove-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Force
   $creds = Get-Credential
   New-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Credential $creds -LocalPath "Z:" -Persistent $true
   ```

**Verification:**
```powershell
Get-SmbGlobalMapping
Test-Path "Z:\"
```

### Solution F: SmbGlobalMapping Disappears After Reboot

**Root Cause:**
The `-Persistent $true` parameter was not set when creating SmbGlobalMapping, or the credential file is corrupted.

SmbGlobalMapping persistence is stored in:
- Registry: `HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings\<DriveLetter>`
- Credential file: `C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials`

**Fix Steps:**
1. Recreate the mapping with persistence:
   ```powershell
   $User = "<Domain\UserName>"
   $PWord = ConvertTo-SecureString -String '<password>' -AsPlainText -Force
   $Creds = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $User, $PWord
   New-SmbGlobalMapping -RemotePath "\\<server>\<share>" -Credential $Creds -LocalPath "H:" -Persistent $true
   ```

**Verification:**
```powershell
# Confirm registry saved
reg query "HKLM\SYSTEM\CurrentControlSet\Control\NetworkProvider\GlobalMappings\H"

# Confirm credential file exists
dir /a "C:\Windows\ServiceProfiles\NetworkService\AppData\Local\Microsoft\Credentials"
```

**Risk Note:**
- If `-Persistent` is not specified, it defaults to the last used value
- Storing passwords in plaintext in scripts is a security risk — use Azure Key Vault or other secure storage in production

### Workaround

If persistent mapping repeatedly fails, consider using a **Logon Script** or **Scheduled Task** to automatically run the mapping command at user logon:

```powershell
# Logon script example
net use L: \\<server>\<share> /persistent:no /user:<domain\username> <password>
```

> The downside of this approach is that the password may need to be hardcoded in the script — take security precautions accordingly.

## 6. Escalation Guidance

Consider escalation when:

- Registry and credentials are correctly configured but the mapped drive still fails to restore after reboot — may involve SMB client or MUP/MRxSMB driver issues
- SmbGlobalMapping credential files repeatedly disappear — may involve lsass.exe credential storage issues
- Inconsistent mapping behavior in large-scale deployment — may involve Group Policy or domain environment configuration
- Abnormal NTLM authentication behavior — may involve security policy or authentication protocol issues

**Information to provide when escalating:**
- Completed troubleshooting steps and results (documented per this TSG)
- Output of `net use`, `reg query HKCU\Network`, `cmdkey /list`
- Mapping method and specific parameters used
- Whether the issue is consistently reproducible, with repro steps
- Customer environment info (OS version, domain/workgroup, authentication method)

## 7. Related Knowledge

### Comparison of Three Mapping Methods

| Feature | File Explorer GUI | net use | SmbGlobalMapping |
|---------|------------------|---------|-----------------|
| Scope | Per-User | Per-User | Per-Machine |
| Persistence Registry | `HKCU\Network\<Drive>` | `HKCU\Network\<Drive>` | `HKLM\...\GlobalMappings\<Drive>` |
| Credential Storage | Credential Manager | Credential Manager | `NetworkService\Credentials` |
| DFS Support | ✅ | ✅ | ❌ |
| Failover Cluster Support | ✅ | ✅ | ✅ |
| Primary Use Case | End users | Scripts/Batch | Windows Containers |

### Key Registry Paths

- `HKCU\Network\<DriveLetter>` — Contains `RemotePath` (share path), `UserName` (mapping credential account, may be empty indicating logon credentials are used), `ProviderName`, etc.
- `HKLM\...\GlobalMappings\<DriveLetter>` — SmbGlobalMapping persistence information

### Common Misconceptions

- **Misconception 1:** `net use` without `/persistent` defaults to non-persistent → It actually inherits the last used value
- **Misconception 2:** `/savecred` and `/user:` can be used together → This causes a conflict error; they must be used separately
- **Misconception 3:** SmbGlobalMapping works for all scenarios → It does not support DFS Namespace
- **Misconception 4:** Authentication "unexpectedly succeeds" when local account passwords match → This is special NTLM behavior and should not be relied upon

## 8. References

- [New-SmbGlobalMapping (SmbShare) | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/smbshare/new-smbglobalmapping) — Official SmbGlobalMapping cmdlet documentation
- [Net use | Microsoft Learn](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/gg651155(v=ws.11)) — Net use command reference

## Revision History

| Date | Version | Changes | Source |
|------|---------|---------|--------|
| 2026-03-10 | 1.0 | Initial version | Created from Windows Map Network Drive technical document |

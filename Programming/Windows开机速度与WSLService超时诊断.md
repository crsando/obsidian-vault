---
date: 2026-08-18
tags:
  - Windows
  - WSL
  - 系统诊断
  - 性能优化
status: 已完成初步定位
---

## 摘要

本次排查针对 **Windows 开机速度偏慢**。结论不是 CPU、内存或 NVMe 性能不足，而是三个阶段叠加：

1. **UEFI/BIOS 阶段**通常约 **18.1 秒**，最近记录中有一次达到 **66.6 秒**，符合 AM5 平台偶发完整 DDR5 内存训练的特征。
2. **WSLService 在快速启动时反复发生 30 秒事务响应超时**。深入排查发现，超时与 **Windows 快速启动恢复旧会话、Hyper-V 清理 WSL 虚拟网卡**高度相关。
3. 登录后有 **19 个启用的启动项**，同时系统盘空闲率只有 **7.6%**，会进一步增加桌面进入后的资源争抢。

当前最值得优先处理的是：**关闭 Windows 快速启动并复测 WSLService 超时**，其次才是精简启动项、释放系统盘空间和更新 BIOS。

## 机器环境

| 项目 | 状态 |
|---|---|
| 操作系统 | Windows 10 Pro 22H2，Build 19045 |
| CPU | AMD Ryzen 7 9800X3D |
| 内存 | 61.6 GB，DDR5 平台 |
| 主板 | MSI MAG B650M MORTAR WIFI（MS-7D76） |
| BIOS | A.L0 / 7D76vAL，固件日期 2025-04-30 |
| 系统盘 | 930.9 GB，剩余 70.5 GB（7.6%） |
| 物理磁盘 | CT1000P1SSD8、Samsung 990 PRO 1TB，均为 Healthy |
| TRIM | 已启用 |
| Windows 快速启动 | 已启用 |
| WSL | 2.7.11.0，默认 Ubuntu / WSL 2 |
| WSLService | `AUTO_START`，最终状态 Running |

## 排查过程

### 1. 排除基础硬件瓶颈

首先读取系统、CPU、内存、磁盘健康、系统盘空间、TRIM 和快速启动配置。

结果：

- Ryzen 7 9800X3D 和 64GB 内存不存在明显性能短板。
- 两块 NVMe 均报告 `Healthy / OK`。
- TRIM 正常开启。
- Windows 快速启动已开启。
- C 盘只剩 **70.5GB / 7.6%**，虽然不是本次 30 秒卡顿的直接原因，但已经低于比较舒服的系统盘余量。

因此，排查重点转向 **固件初始化、Windows 服务和登录启动项**。

### 2. 分析固件阶段耗时

通过 `Microsoft-Windows-Kernel-Boot` 的 Event ID 30 读取固件上报指标。

最近 10 次结果：

- 8 次约 **18.1-18.3 秒**
- 1 次 **19.9 秒**
- 1 次 **66.6 秒**

典型记录中：

- `LoadOSImageStart`：约 15.9 秒
- `ExitBootServicesExit`：约 18.1 秒

说明按下电源后，约前 16 秒主要花在 UEFI 的硬件初始化；之后约 2 秒完成 Windows Boot Manager 交接。

对这台 **AM5 + 64GB DDR5** 机器，主要嫌疑是：

- DDR5 内存训练
- EXPO 参数校验
- PCIe 显卡和两块 NVMe 枚举
- USB、网卡、TPM、Secure Boot 初始化

其中 66.6 秒那次很像完整内存训练。正常 18 秒不算故障，但仍有优化空间。

### 3. 检查 Windows 启动性能日志

`Microsoft-Windows-Diagnostics-Performance/Operational` 通道处于启用状态，但当前没有 Event ID 100 历史汇总。

因此现有日志无法完整拆分：

- Windows 内核加载耗时
- 驱动加载耗时
- 用户配置文件加载耗时
- 桌面出现后的 Post Boot 耗时

后续如需精确到线程或函数，需要使用 Windows Performance Recorder 抓取跨重启 ETL。

### 4. 发现 WSLService 30 秒超时

在 System 日志中反复发现 Service Control Manager Event ID 7011：

> A timeout (30000 milliseconds) was reached while waiting for a transaction response from the WSLService service.

初次统计结果：

- 累计同类记录：**65 次**
- 2026-08-18 本次启动再次出现
- 近期多次启动均有记录
- WSLService 最终能够进入 Running 状态

这表明不是简单的服务启动失败，而是 **服务处理 SCM 控制事务时没有在 30 秒内响应**。

### 5. 排除 WSL 程序损坏和自定义配置

进一步检查：

- `wslservice.exe` 版本：`2.7.11.0`
- 数字签名：Microsoft Corporation，有效
- 默认发行版：Ubuntu，WSL 2
- Ubuntu 当前可正常运行
- WSLService 无显式依赖服务
- 用户目录不存在 `.wslconfig`

由此排除：

- WSLService 可执行文件签名异常
- 明显的 WSL 安装损坏
- `.wslconfig` 中 mirrored networking 等自定义网络配置
- Ubuntu 发行版本身无法启动

### 6. 串联 WSL 超时前后的事件

对最新一次 7011 前后两分钟的事件按毫秒排序，得到关键时间线：

| 相对顺序 | 事件 |
|---|---|
| 1 | Kernel-Boot 上报启动指标，BootType 为 1 |
| 2 | Hyper-V VmSwitch Event 234：WSL 虚拟 NIC 成功从端口断开 |
| 3 | Hyper-V VmSwitch Event 233：WSL 虚拟 NIC 删除成功 |
| 4 | 约 290 毫秒后，Service Control Manager Event 7011：WSLService 事务等待满 30 秒 |

相关虚拟网卡 ID：

```text
85C947C2-013F-4B92-9B0D-B50BCB697C5C--FB9D979A-B8D1-4A19-B915-751B22D1614A
```

当前系统仍存在并正常启用：

- `vEthernet (WSL)`
- `vSwitch (WSL)`
- `vEthernet (Default Switch)`
- `vSwitch (Default Switch)`

这说明问题发生在 **启动时清理或重建上一轮 WSL/Hyper-V 网络状态**，而不是虚拟网卡永久损坏。

### 7. 对比快速启动和完整启动

将 Kernel-Boot Event ID 27 的 `BootType` 与之后两分钟内的 WSLService 7011 进行关联。

最近 100 次启动的结果：

| 启动类型 | 次数 | WSLService 超时 |
|---|---:|---:|
| BootType 0，完整冷启动/完整重启 | 8 | **0** |
| BootType 1，快速/混合启动 | 92 | **53** |

所有 WSLService 超时都出现在 `BootType=1`。

虽然并非每次快速启动都会报错，但完整启动没有出现一次。这是目前最强的相关性证据。

## 当前根因判断

### 高置信度结论

**Windows 快速启动恢复旧系统会话时，WSLService 需要清理上一轮 WSL 轻量虚拟机或 Hyper-V 虚拟网络状态；该清理事务偶发等待满 30 秒，最终在虚拟 NIC 被断开、删除后触发 SCM Event ID 7011。**

这解释了以下现象：

- WSLService 最终仍然正常运行
- Ubuntu 后续可以正常使用
- 没有服务崩溃或签名错误
- 超时紧跟 Hyper-V VmSwitch 虚拟网卡删除事件
- 完整启动不出现，快速启动才出现

### 尚未完全确定的细节

标准 Windows 事件日志没有记录 SCM 向 WSLService 发出的具体控制码，也没有保存 WSLService 内部线程调用栈。因此暂时无法确认究竟卡在：

- HNS 网络对象清理
- Hyper-V VmSwitch 端口删除等待
- WSL 轻量虚拟机终止
- WSLService 内部锁或 RPC 等待

要精确到函数，需要抓取一次跨启动 ETW。

## 修复与验证方案

### 方案一：关闭快速启动，优先推荐

管理员 PowerShell：

```powershell
Set-ItemProperty `
  "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Power" `
  -Name HiberbootEnabled `
  -Value 0
```

恢复快速启动：

```powershell
Set-ItemProperty `
  "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Power" `
  -Name HiberbootEnabled `
  -Value 1
```

这台机器使用 NVMe，快速启动带来的收益有限。如果关闭快速启动能稳定消除 30 秒超时，总启动体验反而会更快。

### 方案二：关机前主动停止 WSL

先保存 Linux 内任务，然后：

```powershell
wsl --shutdown
```

再正常关机并开机。如果超时消失，可进一步确认是上一轮 WSL 运行状态在混合关机过程中没有清理干净。

### 方案三：把 WSLService 改为延迟自动启动

```powershell
sc.exe config WSLService start= delayed-auto
```

恢复：

```powershell
sc.exe config WSLService start= auto
```

该方案主要减少登录阶段竞争，可能只是把清理延后，不一定消除根因，因此优先级低于关闭快速启动。

### 重启后验证命令

```powershell
Get-WinEvent -FilterHashtable @{
  LogName   = "System"
  Id        = 7011
  StartTime = (Get-Date).AddMinutes(-10)
} -ErrorAction SilentlyContinue |
Where-Object Message -Like "*WSLService*" |
Select-Object TimeCreated, Message
```

没有输出，说明本次启动没有发生 WSLService 事务超时。

### ETW 深度跟踪

管理员 PowerShell 开启跨重启跟踪：

```powershell
mkdir C:\wsl-trace -ErrorAction SilentlyContinue

wpr.exe -boottrace `
  -addboot GeneralProfile `
  -addboot CPU `
  -addboot DiskIO `
  -addboot Network `
  -filemode
```

正常关机再开机，出现超时后停止并保存：

```powershell
wpr.exe -boottrace -stopboot `
  C:\wsl-trace\wsl-boot.etl `
  "WSLService 7011 timeout"
```

ETL 可以继续分析 `WSLService`、HNS、Hyper-V VmSwitch 的线程等待、CPU、磁盘和网络活动。

## 其他影响开机体验的问题

### 登录启动项

确认启用的启动项共 19 个：

`BaiduYunDetect`、`BaiduYunGuanjia`、`bm4200Monitor`、`cloudmusic`、`PicGo`、`ctfmon`、`Dell Display Manager`、雷神、`ExpressVPNNotificationService`、`GlazeWM`、Copilot、`Launch LCore`、`LGHUB`、`Logitech Download Assistant`、`OneDrive`、`Send to OneNote`、`Twitch`、`Wechat`、`Weixin`。

优先考虑停用但不卸载：

- `BaiduYunDetect`
- `cloudmusic`
- `PicGo`
- `Twitch`
- `Send to OneNote`
- `Logitech Download Assistant`

### 系统盘空间

当前空闲率仅 **7.6%**。建议至少保持 **15%**，目标为空闲约 140GB，即再释放约 70GB。

### BIOS 和内存训练

当前 BIOS 为 `7D76vAL`，MSI 官方支持页在本次查询时提供 `7D76vAO2`（2026-07-06）。更新后可检查：

- `Memory Context Restore`：Enabled
- `Power Down Enable`：Auto 或 Enabled
- EXPO 是否稳定
- 无用的 PXE、网络启动和启动设备是否关闭

BIOS 更新可能重置 EXPO、虚拟化、风扇曲线和启动顺序，操作前应记录现有设置。

## 建议执行顺序

1. 保存 WSL 工作负载，执行 `wsl --shutdown`。
2. 关闭 Windows 快速启动。
3. 完整重启并检查新的 Event ID 7011。
4. 若问题消失，维持快速启动关闭并观察几天。
5. 若完整启动仍出现超时，抓取 WPR Boot ETL。
6. 再精简非必要启动项，并把 C 盘空闲率恢复到 15% 以上。
7. 最后更新 BIOS，处理 AM5 偶发长时间内存训练。

## 注意事项

- 不要执行 `wsl --unregister Ubuntu`，该命令会删除 Ubuntu 发行版及其数据。
- 不要在有重要 Linux 任务运行时执行 `wsl --shutdown`。
- 暂时不建议删除 HNS 数据库或重置全部 Hyper-V 网络，这会影响 Docker、Hyper-V 和其他虚拟网络，应在 ETW 证明 HNS 状态损坏后再考虑。
- 关闭快速启动不会关闭 WSL，也不会删除休眠文件；这里只修改 `HiberbootEnabled`。

## 来源

- 本机 Windows System 事件日志
  - `Microsoft-Windows-Kernel-Boot` Event ID 27、30、32
  - `Service Control Manager` Event ID 7011
  - `Microsoft-Windows-Hyper-V-VmSwitch` Event ID 233、234
- 本机 CIM、服务、启动注册表、计划任务、磁盘健康与网络适配器数据
- WSL 本机命令：`wsl --version`、`wsl --status`、`wsl --list --verbose`
- MSI 官方主板支持页：https://www.msi.com/Motherboard/MAG-B650M-MORTAR-WIFI/support
- 排查日期：2026-08-18

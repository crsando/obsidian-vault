---
title: OnePlus 5 OxygenOS 刷机、Root 与 Wi-Fi 排查记录
date: 2026-08-30
device: OnePlus 5
model: ONEPLUS A5000
tags:
  - Android
  - OnePlus5
  - OxygenOS
  - Magisk
  - LSPosed
  - WiFi
---

# OnePlus 5 OxygenOS 刷机、Root 与 Wi-Fi 排查记录

## 今日最终状态

- 设备：OnePlus 5，型号 `ONEPLUS A5000`，序列号已在本次会话中核验。
- 系统：OxygenOS 10.0.1，GLO 069。
- 当前 Build：`OnePlus5Oxygen_23.J.69_GLO_069_2010292138`。
- Android：10，增量版本 `2010292059`。
- Bootloader：已解锁，状态为 `unlocked: yes`。
- Magisk：v30.7，Root 验证成功。
- Zygisk：已开启。
- LSPosed：v1.9.2，`lspd`、`zygiskd64`、`zygiskd32` 正常运行。
- Wi-Fi：已恢复，`wlan0` 正常，已连接 `QM702`，Supplicant 状态 `COMPLETED`。
- 微信：`8.0.48`，versionCode `2589`，已安装。
- GodHook：v1.4.6，已安装并配置微信作用域。
- GodHook 注入：已确认加载到微信主进程和 `com.tencent.mm:push`。
- GodHook 当前未完成初始化，`appValid=false`，5888 端口未监听。

Root 最终验证：

```text
uid=0(root) gid=0(root) groups=0(root) context=u:r:magisk:s0
Magisk version code: 30700
ro.boot.verifiedbootstate=orange
```

## 系统升级过程

起初系统是中国版 H₂OS：

```text
ro.build.version.ota=OnePlus5Hydrogen_23.K.62_062_2005291726
ro.build.display.id=ONEPLUS A5000_23_200529
ro.build.version.incremental=2005291648
ro.rom.version=10.0.0
```

此前使用的 GLO 067 包与该系统不完全匹配，后来确认设备实际为 H₂OS 062。通过官方 OnePlus OTA 查询接口确认后续官方版本为 H₂OS 065，但本次目标是切换到 OxygenOS 10.0.1。

使用的完整 OxygenOS 包：

```text
OnePlus5Oxygen_23_OTA_069_all_2010292138_6700.zip
```

包内 metadata：

```text
ota-id=OnePlus5Oxygen_23.J.69_GLO_069_2010292138
pre-device=OnePlus5
post-build-incremental=2010292059
post-security-patch-level=2020-09-01
wipe=0
```

包校验：

```text
MD5: 8f41d1cca57e8a91879f4b09088103b4
SHA-256: 660db8238cde8734ff6b4b05a0f92f9a7986952b640363e257c6875cb3002954
```

通过 Stock Recovery 的 `Apply update from ADB` 执行 sideload。传输到约 94% 时 Mac 端 ADB 报 `Undefined error: 0`，但 Recovery 屏幕随后显示安装成功，说明实际安装已经完成。

升级后第一次启动曾卡在启动动画。日志显示：

```text
UserValidator: UserHandle{0} is lock
LauncherApplication: Kill self process
```

完成一次 PIN 解锁后，OxygenOS 正常进入桌面。

## Wi-Fi 故障与修复

在 H₂OS 期间，Wi-Fi 无法启动，表现为：

```text
没有 wlan0
Wi-Fi is disabled
Cannot start IWifi: 9
Failed to start vendor HAL
Failed to create ClientInterface
wifi driver unloaded
```

关闭/开启 Wi-Fi、临时禁用 LSPosed、刷回原厂 067 boot 做对照，均没有恢复 Wi-Fi，因此排除了 GodHook、LSPosed 和 Magisk 修补本身的直接影响。

完整刷入 OxygenOS 10.0.1 后，Wi-Fi 恢复：

```text
wlan0 UP
Wi-Fi is enabled
SSID: QM702
Supplicant state: COMPLETED
numSetupClientInterfaceFailureDueToHal=0
```

这说明问题很可能来自原 H₂OS 系统与此前使用的 GLO 067 boot/固件组合不匹配，或者旧系统的 Wi-Fi 固件状态异常。完整 OTA 同时更新了 system、vendor、boot、modem、DSP、Bluetooth 和 Qualcomm 固件。

## Root 恢复过程

完整 OTA 会覆盖 `boot` 分区，所以原先的 Magisk Root 和 LSPosed 被清除。

使用 069 包内匹配的原厂 boot 镜像：

```text
OnePlus5Oxygen_23_OTA_069_all_2010292138_6700/boot.img
```

原厂 069 boot SHA-256：

```text
ffa5acb2c897ef940f9b133620097e4504f8fa61a384fb3c98baf841f26bc7c0
```

通过手机 Magisk v30.7 的 `Select and patch a file` 修补。第一次生成的文件是 0 字节，未使用；重新从 Magisk 安装页执行后成功显示：

```text
Stock boot image detected
Patching ramdisk
Repacking boot image
All done!
```

修补镜像保存于项目：

```text
outputs/magisk_patched-069.img
```

修补镜像信息：

```text
大小：22484266 bytes
SHA-256：4956fca25b3a27d85235896af6b2d65052ad5be1bf5411beb08d86b16882354
```

刷入命令：

```bash
fastboot flash boot outputs/magisk_patched-069.img
fastboot reboot
```

刷入返回：

```text
Sending 'boot' (21957 KB) OKAY
Writing 'boot' OKAY
```

## LSPosed 与 GodHook

069 OTA 后重新安装：

```text
outputs/modules/LSPosed-v1.9.2-7024-zygisk-release.zip
outputs/apks/WeChat-8.0.48-2589-arm64.apk
outputs/apks/GodHook-1.4.6.apk
```

开启 Zygisk 后重启，重新把 GodHook 注册为 LSPosed 模块，并将作用域限制为：

```text
cc.godhook.webapi -> com.tencent.mm
```

日志确认：

```text
Loading legacy module cc.godhook.webapi
Loading class cc.godhook.webapi.hook.HookEntry_YukiHookXposedInit
```

主微信进程和推送进程均成功加载 GodHook。

但 GodHook 初始化时仍然是：

```text
appValid=false
5888 端口未监听
```

其网络初始化依赖 `app.godhook.top`，当前设备对该域名解析失败。因此目前只确认了安装和 LSPosed 注入，尚未进行真实微信消息收发测试，也没有配置任何模型 API Key。

## 当前项目文件

- 完整 OxygenOS OTA：`OnePlus5Oxygen_23_OTA_069_all_2010292138_6700.zip`
- 解压后的 069 固件：`OnePlus5Oxygen_23_OTA_069_all_2010292138_6700/`
- 069 原厂 boot：`OnePlus5Oxygen_23_OTA_069_all_2010292138_6700/boot.img`
- 069 Magisk boot：`outputs/magisk_patched-069.img`
- Magisk APK：`tools/Magisk-v30.7.apk`
- 微信 APK：`outputs/apks/WeChat-8.0.48-2589-arm64.apk`
- GodHook APK：`outputs/apks/GodHook-1.4.6.apk`
- LSPosed ZIP：`outputs/modules/LSPosed-v1.9.2-7024-zygisk-release.zip`
- LSPosed 配置备份和临时数据库：`tmp/`

## 后续注意事项

- 不要重新锁定 Bootloader。
- 不要把 067 boot 刷回 069 OxygenOS。
- OTA 升级前先准备对应版本的原厂 boot 镜像。
- GodHook 的 5888 服务恢复前，不要暴露端口到公网。
- 微信机器人测试只使用专门测试账号，并仅处理获得授权的消息。
- 暂不进行群发、自动加好友、红包、转账或其他高风险操作。

## 后续诊断：GodHook 替代方案（2026-08-30）

已决定放弃 GodHook。当前设备通过 USB ADB 正常连接，GodHook v1.4.6 的 INTERNET、ACCESS_NETWORK_STATE 和 ACCESS_WIFI_STATE 权限均已授予；但设备和 Mac 对 app.godhook.top 的解析均返回 NXDOMAIN，Cloudflare DNS、Google DNS、Quad9 交叉验证结果一致。设备侧 ping 报 unknown host，5888 端口未监听，因此不是 Root、LSPosed 注入或 Wi-Fi 故障。

下一步改为设计并实现本地微信桥接器：先用 Android AccessibilityService 做固定微信 8.0.48 的只读文本监听和适配自检，再做默认关闭、白名单、串行队列、限流和三态确认的文本发送。暂不读取微信私有数据库、不使用私有网络协议、不开放公网端口；LSPosed 仅作为后续可选适配层。设计稿见：outputs/oneplus5-wechat-bridge-design.md

## Phase 0：微信适配探测器（2026-08-30）

已开始实现 GodHook 替代方案的 Phase 0。当前工作区新增 Kotlin Android 项目 `WeChat Bridge Probe`：仅监听 `com.tencent.mm` 的 Accessibility 窗口/内容变化，显示无障碍状态、Android/微信版本和脱敏 UI 诊断摘要；支持暂停采集与清空诊断。明确不发送消息、不使用剪贴板、不截图、不读取微信私有数据库、不 Hook、不联网。

项目文件位于当前工作区，设计稿和最终交付说明见 `outputs/oneplus5-wechat-bridge-design.md`。当前 Mac 缺少 JDK、Gradle 和 Android SDK，尚未生成 APK；下一步安装构建环境后执行 `assembleDebug`，再通过 USB 安装到 OnePlus 5 做真实 UI 探测。

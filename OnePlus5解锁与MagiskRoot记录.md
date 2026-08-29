---
title: OnePlus 5 解锁 Bootloader 与 Magisk Root 记录
date: 2026-08-29
device: OnePlus 5
model: ONEPLUS A5000
tags:
  - Android
  - OnePlus5
  - Magisk
  - Root
---

# OnePlus 5 解锁 Bootloader 与 Magisk Root 记录

## 最终结果

本次已完成 OnePlus 5 的 Bootloader 解锁和 Magisk Root：

- ADB、Fastboot 工具可用。
- 设备：OnePlus 5，型号 `ONEPLUS A5000`，序列号 `a166595`。
- Bootloader 已解锁：`unlocked: yes`。
- Magisk v30.7 已安装。
- 修补后的 boot 镜像已刷入 `boot` 分区。
- Root 验证成功：`uid=0(root)`，SELinux context 为 `u:r:magisk:s0`。

最终验证输出：

```text
uid=0(root) gid=0(root) groups=0(root) context=u:r:magisk:s0
Magisk version code: 30700
ro.boot.verifiedbootstate: orange
```

## 设备与系统信息

```text
型号：ONEPLUS A5000
产品：OnePlus5 / cheeseburger
Android：10
Build：ONEPLUS A5000_23_200529
系统显示版本：ONEPLUS A5000_23_200529
```

初次读取到的解锁相关属性：

```text
ro.oem_unlock_supported: 1
ro.boot.flash.locked: 1
```

解锁前 Fastboot：

```text
unlocked: no
secure: yes
get_unlock_ability: 1
```

解锁后 Fastboot：

```text
unlocked: yes
secure: yes
```

## 电脑工具

电脑为 macOS Apple Silicon，工具版本：

```text
adb 1.0.41
Android Debug Bridge version 37.0.1-15733141
fastboot 37.0.1-15733141
```

手机开启开发者选项后，开启「OEM 解锁」和「USB 调试」，并接受 USB 调试 RSA 授权。macOS 能识别设备，ADB 和 Fastboot 均可正常通信。

## Bootloader 解锁过程

1. 设置 → 关于手机 → 连续点击「版本号」7 次，打开开发者选项。
2. 开发者选项中开启「OEM 解锁」和「USB 调试」。
3. 接受电脑的 USB 调试授权。
4. 使用 ADB 重启到 Bootloader。
5. 执行：

```bash
fastboot flashing unlock
```

6. 在手机屏幕上确认解锁。
7. 手机恢复出厂并重启。

解锁会清除全部用户数据；本次已确认并完成清除。

## 固件与原厂 boot 镜像

使用的是 OnePlus 5 OxygenOS 10.0.0 完整 OTA 067：

```text
OnePlus5Oxygen_23_OTA_067_all_2005130007_4e90d4036.zip
```

包内 metadata：

```text
ota-id=OnePlus5Oxygen_23.J.67_GLO_067_2005130007
pre-device=OnePlus5
post-sdk-level=29
post-security-patch-level=2020-04-05
post-build-incremental=2005122320
```

原厂镜像：

```text
OnePlus5Oxygen_23_OTA_067_all_2005130007_4e90d4036/boot.img
SHA-256: fd9ca20e7654e2124b6c292dec44d17d4f20e1d3b8fa719bdd673c902cd0a2eb
```

注意：手机当前 Build 字符串为 `200529`，OTA 067 的包内增量版本为 `200513`。067 与当前 OxygenOS 10.0.0 更接近，但不是完全相同的增量 Build；原厂镜像已保留用于恢复。

## Magisk 修补与刷入

官方 Magisk v30.7 已安装：

```text
tools/Magisk-v30.7.apk
SHA-256: e0d32d2123532860f97123d927b1bb86c4e08e6fd8a48bfc6b5bee0afae9ebd5
```

在手机 Magisk 中使用「选择并修补一个文件」修补原厂 067 `boot.img`，生成修补镜像。电脑保存为：

```text
outputs/magisk_patched-067.img
```

修补镜像信息：

```text
类型：Android boot image
大小：22484266 bytes
SHA-256：29ce2b00cf37759d16220b172eca3b3370d64385000d0c44b8693bb2c9d568ac
```

刷入命令：

```bash
fastboot flash boot outputs/magisk_patched-067.img
fastboot reboot
```

刷入结果：

```text
Sending 'boot' (21957 KB) OKAY
Writing 'boot' OKAY
```

随后通过 `adb shell su -c id` 验证成功。首次验证时 Magisk 需要对 ADB shell 授权，授权后返回 root。

## 项目文件

- 原厂 `boot.img`：`OnePlus5Oxygen_23_OTA_067_all_2005130007_4e90d4036/boot.img`
- 修补 `boot.img`：`outputs/magisk_patched-067.img`
- Magisk APK：`tools/Magisk-v30.7.apk`
- 完整 OTA ZIP：`OnePlus5Oxygen_23_OTA_067_all_2005130007_4e90d4036.zip`

## 后续注意事项

- 不要重新锁定 Bootloader，否则可能无法启动并再次清除数据。
- OTA 升级可能覆盖修补后的 boot 分区，升级后要重新检查 Magisk。
- 不要混用其他 Build 的 boot 镜像。
- 出现启动问题时，使用保留的原厂 `boot.img` 从 Fastboot 回刷恢复。
- Magisk 模块应逐个安装，出现异常时先禁用最近安装的模块。
- 已解锁设备的 Verified Boot 状态为 `orange`，这是预期状态。
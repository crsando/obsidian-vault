---
title: 网络电视（IPTV）与各大 TV 平台配置指南及精选频道源汇总
date: 2026-09-18
tags:
  - IPTV
  - Kodi
  - 电视盒子
  - AppleTV
  - 流媒体
  - 指南
---

# 网络电视（IPTV）与各大 TV 平台配置指南及精选频道源汇总

本笔记系统整理了网络电视（IPTV）在各大主流电视系统（Android TV / Apple TV / 三星 Tizen / LG webOS）下的软件选择、安装、M3U 源导入与电子节目单（EPG）配置方法，并汇总了目前长期维护、高质量的国内外经典电视频道直链与开源聚合订阅源。

---

## 目录
- [[#一、各大 TV 平台安装与导入使用指南]]
  - [[#1. Android TV / 安卓电视盒子（索尼、小米、外贸盒子等）]]
    - 方案 A：Kodi（全能家庭影院 + 电视一体）
    - 方案 B：TiviMate（电视端专有交互天花板）
  - [[#2. Apple TV (tvOS)]]
    - 首选推荐：APTV
    - 备选方案：SenPlayer / Snappy
  - [[#3. 三星 (Tizen) & LG (webOS) 电视]]
    - 推荐方案：SS IPTV（网页免打字绑定法）
  - [[#4. 跨平台快速测试工具]]
- [[#二、精选经典与受欢迎的电视频道盘点（含具体地址）]]
  - [[#1. 慢电视 & 治愈挂机类（Slow TV & Ambient）]]
  - [[#2. 经典影视 & 动漫 24 小时轮播台（电子榨菜专区）]]
  - [[#3. 高清纪录片 & 文化地理（Documentary & Nature）]]
  - [[#4. 极限运动 & 氛围音乐（Sports & Music）]]
- [[#三、主流开源聚合订阅源与 EPG 资源]]
  - [[#1. 全球最大合法开源项目：iptv-org]]
  - [[#2. 国内央视 / 卫视高清项目：fanmingming/live]]
  - [[#3. 综合轮播与多平台聚合：YanG-1989/m3u]]
  - [[#4. 常用 EPG 节目单与台标源]]
- [[#四、实操避坑与使用技巧]]

---

## 一、各大 TV 平台安装与导入使用指南

### 1. Android TV / 安卓电视盒子（索尼、小米、外贸盒子等）

安卓生态具有最高的自由度，既可以使用功能强大的全能媒体中心 Kodi，也可以选用极度贴合传统电视遥控操作的 TiviMate。

#### 方案 A：Kodi（全能影音 + 电视一体）
Kodi 内置了 PVR（个人视频录像机）架构，配合官方内置的 `PVR IPTV Simple Client` 插件，可以直接将网络电视直播整合进主菜单。

1. **安装 Kodi**：
   - 电视自带应用商店（如 Google Play Store）直接搜索 `Kodi` 安装；
   - 或用电脑访问 [Kodi 官方下载页](https://kodi.tv/download/) 下载 Android ARM 版本的 `.apk` 文件，拷入 U 盘插到电视上安装。
2. **启用与配置 PVR IPTV Simple Client 插件**：
   - 进入 Kodi 主界面，点击左上角齿轮图标进入 **设置 (Settings)**；
   - 进入 **插件 (Add-ons)** -> **从库安装 (Install from repository)** -> 选择 **Kodi Add-on repository**；
   - 选择 **PVR 客户端 (PVR clients)**，往下拉找到 **PVR IPTV Simple Client**，点击并选择右下角 **安装 (Install)**；
   - 安装完成后点击该插件，选择 **配置 (Configure)**：
     - **常规 (General)** 选项卡：
       - *位置 (Location)*：选择 `远程路径 (Internet address)`；
       - *播放列表 URL (Playlist URL)*：填入你的 `.m3u` 或 `.m3u8` 订阅地址（如本地文件则选择 `本地路径`）；
     - **EPG 设置 (EPG Settings)** 选项卡：
       - *位置 (Location)*：选择 `远程路径 (Internet address)`；
       - *XMLTV URL*：填入 EPG 电子节目单地址（如 `https://live.fanmingming.com/e.xml`）；
     - **频道图标 (Channel Logos)** 选项卡：
       - 图标位置选择 `来自 M3U 内 XMLTV` 或设置基准 URL。
3. **启用主页电视入口**：
   - 确定保存设置后，Kodi 会提示重启。完全退出并重启 Kodi；
   - 主界面左侧导航栏会自动点亮 **电视 (TV)** 模块，点入即可浏览频道、电子节目单、分类与台标。

#### 方案 B：TiviMate（电视端专有交互天花板）
如果电视只用来收看网络电视直播，TiviMate 的操作体验比 Kodi 更轻快丝滑，完全还原高端广电/电信机顶盒的遥控体验。

1. **安装**：
   - 官网下载 APK（或通过 Google Play），使用 U 盘侧载到安卓电视安装。
2. **导入配置**：
   - 打开 TiviMate -> 点击 **添加播放列表 (Add Playlist)** -> 选择 **M3U 播放列表 (M3U Playlist)**；
   - 填入 M3U 订阅 URL，并为其命名；
   - 在弹出的 EPG 选项中填入配套的 XMLTV 链接；
   - 导入后自动按标签分组（央视、卫视、电影轮播、纪录片等），支持遥控器数字键直接切台、方向键快速预览浮窗。

---

### 2. Apple TV (tvOS)

tvOS 无法直接通过官方商店安装未签名的 Kodi（需通过 Xcode 或企业证书签名侧载且每 7 天需重签，维护成本高）。在 Apple TV 上，原生定制的 IPTV 播放器体验远超侧载。

#### 首选推荐：APTV
APTV 是 iOS/tvOS 生态中极具设计美感且性能出色的网络电视工具。

1. **安装**：
   - 在 Apple TV 的 App Store 中直接搜索并下载 `APTV`。
2. **导入源与设置**：
   - 打开 APTV，点击右上角 **“+”号（添加配置）**；
   - 选择 **从链接添加**，粘贴 `.m3u` 订阅链接并自定义名称；
   - 支持填写专属 EPG 链接与 User-Agent 自定义；
   - **iCloud 同步**：如果 iPhone 或 iPad 同时也安装了 APTV，可在手机上直接粘贴链接，通过 iCloud 自动秒级同步至 Apple TV，省去在电视端输入的繁琐。
3. **日常使用**：
   - 完美适配 Siri Remote 遥控器的触控手势滑动切台；
   - 原生支持 tvOS 系统级“画中画（PiP）”功能，可一边看电视一边操作其他应用。

#### 备选方案：SenPlayer / Snappy
- **SenPlayer**：全能流媒体播放器，同时支持 WebDAV 挂载网盘、本地 SMB 共享与 IPTV M3U 播放；
- **Snappy**：简洁轻量的 tvOS IPTV 客户端。

---

### 3. 三星 (Tizen) & LG (webOS) 电视

三星与 LG 采用各自专有的智能电视操作系统（非 Android），无法运行普通 `.apk` 文件。最通用、免改系统区域的方案是使用应用商店自带的 **SS IPTV**。

#### 推荐方案：SS IPTV（网页免遥控打字导入法）
1. **安装 App**：
   - 打开三星 **Samsung Smart TV Apps** 或 LG **Content Store**；
   - 搜索并安装 `SS IPTV`。
2. **免打字绑定（代码配对）**：
   - 启动电视上的 SS IPTV，点击右上角的 **设置（齿轮图标）**；
   - 在 **常规 (General)** 子页面中，点击 **获取代码 (Get code)**，屏幕上会显示一个有时效性的临时代码（如 `D7K2A`）；
   - 在电脑或手机浏览器打开 SS IPTV 官方控制台：[https://ss-iptv.com/en/users/playlist](https://ss-iptv.com/en/users/playlist)；
   - 输入电视上显示的临时代码，点击 **Add Device（绑定设备）**；
   - 切换到 **External Playlists（外部播放列表）** 标签页，点击 **Add Item**：
     - *Source*：粘贴你的 M3U 订阅 URL；
     - *Title*：自定义列表名称；
     - 点击 **Save** 保存。
3. **电视端同步与播放**：
   - 返回电视端 SS IPTV 主界面，点击右上角或底部的 **刷新 (Refresh)** 按钮；
   - 刚添加的播放列表瓷贴会出现在主页，遥控器点击即可载入全部频道。

---

### 4. 跨平台快速测试工具
在把源配置进电视前，建议先在电脑端确认直播流能否稳定播放：
- **macOS**：推荐使用 [IINA](https://iina.io/)（菜单栏 -> 文件 -> 打开 URL 即可测试 m3u8，或直接把 m3u 文件拖入当播放列表）
- **Windows**：推荐使用 [PotPlayer](https://potplayer.daum.net/)（按 `Ctrl + U` 粘贴地址，或将 `.m3u` 拖入播放列表窗口）

---

## 二、精选经典与受欢迎的电视频道盘点（含具体地址）

以下整理了在 IPTV 玩家社群中口碑极佳、适合在家庭大屏幕上长期收看/挂机的经典频道直链：

### 1. 慢电视 & 治愈挂机类（Slow TV & Ambient）
大屏幕作为家居环境音和动态风景画卷的最佳选择。

| 频道名称 | 简介与特色 | 直播流地址 / 形式 |
| :--- | :--- | :--- |
| **NASA TV Public HD** | 国际空间站看地球日出日落、太空行走与航天任务直播 | `https://ntv1.akamaized.net/hls/live/2014075/NASA-NTV1-HLS/master.m3u8` |
| **NASA TV Media** | NASA 官方新闻发布会、工程与高清深空观测素材展示 | `https://ntv2.akamaized.net/hls/live/2014076/NASA-NTV2-HLS/master.m3u8` |
| **Space Live by Sen** | 4K 太空遥感与地球实时动态镜头，纯净慢直播 | `https://linear-1224.frequency.stream/dist/lg-uk/1224/hls/master/playlist.m3u8` |
| **WildEarth** | 非洲野生动物保护区越野车实地跟拍、水塘蹲守动物直播 | `https://dqga3jatxofgx.cloudfront.net/WildEarth.m3u8` |
| **Stingray ZenLIFE** | 4K 自然风景（海浪、林间细雨、雪山）伴随白噪音与冥想音乐 | `https://lotus.stingray.com/manifest/zenlife-zen001-montreal/samsungtvplus/master.m3u8` |
| **MyZen TV** | 欧洲知名身心治愈生活方式台，瑜伽、风景、健康慢生活 | `https://cdn-ue1-prod.tsv2.amagi.tv/linear/amg01255-secomcofites-my-myzen-en-plex/playlist.m3u8` |
| **NRK 慢电视** | 挪威国家广播公司经典的卑尔根铁路第一视角、峡湾轮船慢漫游 | 收录于 `iptv-org` 的 `relax.m3u` 分类源中 |

---

### 2. 经典影视 & 动漫 24 小时轮播台（电子榨菜专区）
无需挑选集数，任何时间切入都能无脑下饭。

- **周星驰经典电影专台**：24 小时无间断循环播放《大话西游》《九品芝麻官》《唐伯虎点秋香》《功夫》等全集。
- **林正英僵尸片专台**：香港经典灵幻动作片、一眉道人系列全天轮播。
- **下饭剧单剧专台**：
  - 《甄嬛传》24 小时轮播台
  - 《武林外传》24 小时轮播台
  - 《亮剑》24 小时轮播台
  - 《家有儿女》24 小时轮播台
  - 《老友记 (Friends)》24 小时英文原声轮播台
- **经典怀旧动漫台**：
  - 《蜡笔小新》《名侦探柯南》《猫和老鼠》《哆啦A梦》中文配音全天循环。
- **获取方式**：此类轮播源多由爱好者推流聚合，最稳定全套集合见下方 `YanG-1989/m3u` 的 `Gather.m3u`。

---

### 3. 高清纪录片 & 文化地理（Documentary & Nature）

| 频道名称 | 简介与特色 | 地址 / 来源 |
| :--- | :--- | :--- |
| **NHK World-Japan** | 制作水准极高的日本文化、街头美食、铁道与自然纪录，全英文解说 | 官方高清流：`https://nhkwlive-ojp.akamaized.net/hls/live/2003459/nhkwlive-ojp-en/index.m3u8` <br>官方网站：[NHK World](https://www3.nhk.or.jp/nhkworld/) |
| **BBC Earth** | BBC 顶级自然与生态纪录片流（地球脉动、蓝色星球系列） | `https://amg00793-amg00793c6-xumo-us-2669.playouts.now.amagi.tv/BBCStudios-BBCEarthA-hls/playlist.m3u8` |
| **Autentic History** | 欧美严谨历史风云、二战实录与古代文明考证 | `https://9e754fa707344ccca6d84955c8fcaf36.mediatailor.us-east-1.amazonaws.com/v1/master/44f73ba4d03e9607dcd9bebdcb8494d86964f1d8/RlaxxTV-eu_AutenticHistory/playlist.m3u8` |
| **CCTV-9 纪录 & CCTV-10 科教** | 国内文博探索、国家地理与中国美食纪录（需支持 IPv6） | 收录于 `fanmingming/live`，画质达 1080p 高码率 |

---

### 4. 极限运动 & 氛围音乐（Sports & Music）

- **Red Bull TV（红牛 TV）**：
  - 全球顶级极限运动赛事官方流，包含翼装飞行、山地速降车、自由潜水、BMX、滑板锦标赛等，摄影机位与剪辑水准极高，完全免费合法；
  - 官方直链源：`https://rbmn-live.akamaized.net/hls/live/590964/BoRB-AT/master.m3u8`
- **Stingray CMusic / Deluxe Lounge**：
  - 经典影视原声交响乐 MV、小众爵士、黑胶复古流行乐轮播，适合作为聚会背景音乐。

---

## 三、主流开源聚合订阅源与 EPG 资源

建议直接在播放器中填入以下长期维护的开源订阅链接，播放器会自动按分类整理频道并定期自动更新失效地址。

### 1. 全球最大合法开源项目：iptv-org
GitHub Star 超过 90k 的开源电视源仓库，汇聚全球公开合法的电视广播信号：
- **项目仓库**：[GitHub - iptv-org/iptv](https://github.com/iptv-org/iptv)
- **精选分类 M3U 订阅直链**：
  - 慢电视与治愈系 (Relax)：`https://iptv-org.github.io/iptv/categories/relax.m3u`
  - 自然与人文纪录片 (Documentary)：`https://iptv-org.github.io/iptv/categories/documentary.m3u`
  - 科教与航天 (Science)：`https://iptv-org.github.io/iptv/categories/science.m3u`
  - 电影专区 (Movies)：`https://iptv-org.github.io/iptv/categories/movies.m3u`
  - 连续剧与肥皂剧 (Series)：`https://iptv-org.github.io/iptv/categories/series.m3u`
  - 音乐与现场 (Music)：`https://iptv-org.github.io/iptv/categories/music.m3u`
  - 体育频道 (Sports)：`https://iptv-org.github.io/iptv/categories/sports.m3u`
  - 全球总索引 (All in One)：`https://iptv-org.github.io/iptv/index.m3u`
- **地区订阅直链**：
  - 中国 (CN)：`https://iptv-org.github.io/iptv/countries/cn.m3u`
  - 中国香港 (HK)：`https://iptv-org.github.io/iptv/countries/hk.m3u`
  - 日本 (JP)：`https://iptv-org.github.io/iptv/countries/jp.m3u`
  - 美国 (US)：`https://iptv-org.github.io/iptv/countries/us.m3u`

---

### 2. 国内央视 / 卫视高清项目：fanmingming/live
国内非常知名的高质量直播源，基于公共官方源与优质 IPv6 提取，画质极清且抗波动能力强：
- **项目仓库**：[GitHub - fanmingming/live](https://github.com/fanmingming/live)
- **IPv6 高清电视订阅源**：
  - 直链：`https://live.fanmingming.com/tv/m3u/ipv6.m3u`
  - GitHub 备用直链：`https://raw.githubusercontent.com/fanmingming/live/main/tv/m3u/ipv6.m3u`
- **国家广播电台音频订阅源**：
  - 直链：`https://live.fanmingming.com/radio/m3u/index.m3u`
- **配套 EPG 电子节目单**：
  - XMLTV 链接：`https://live.fanmingming.com/e.xml`

---

### 3. 综合轮播与多平台聚合：YanG-1989/m3u
涵盖主流卫视、全天候经典剧轮播专区、赛事专栏的综合仓库：
- **项目仓库**：[GitHub - YanG-1989/m3u](https://github.com/YanG-1989/m3u)
- **精简版聚合订阅源**：
  - `https://raw.githubusercontent.com/YanG-1989/m3u/main/Gather.m3u`
- **特色内容**：包含周星驰、成龙动作、武侠影院、多部高分国产经典剧轮播。

---

### 4. 常用 EPG 节目单与台标源
EPG 用于在电视机上呈现当前节目名、下个时段预告及时间轴进度条：
- **范明明 EPG**：`https://live.fanmingming.com/e.xml`（国内央卫视匹配度最高）
- **112114 EPG 服务**：`http://epg.112114.xyz/pp.xml`
- **iptv-org EPG**：`https://iptv-org.github.io/epg/guides/cn.xml`（国际源匹配）

---

## 四、实操避坑与使用技巧

1. **解决遥控器输入长链接的痛点**：
   - **Android TV**：手机下载并连接《Google TV》或《Android TV Remote》，在手机端复制 URL 直接粘贴至电视输入框；
   - **Apple TV**：开启局域网蓝牙，输入框弹出时旁边的 iPhone/iPad 会自动跳出键盘推送，直接通过通用剪贴板一键粘贴；
   - **三星/LG 电视**：使用上文介绍的 SS IPTV 网页端代码配对法，在电脑端浏览器直接粘贴保存。
2. **IPv6 对国内高清源的决定性影响**：
   - 很多高码率 4K/1080p 央视卫视源（如 `fanmingming/live`）走的是电信、移动、联通的 IPv6 专线通道，不消耗普通公网带宽且几乎不卡顿；
   - **前提**：需确保家用光猫及主路由器已开启 **IPv6**（可通过手机或电视浏览器访问 `test-ipv6.com` 测试）。
3. **海外电视频道的网络环境**：
   - 像 Pluto TV、BBC Earth、部分美区 FAST 频道存在 Geo-block（地理区域限制），路由器端或电视盒子端需要具备分流代理环境才能稳定获取分片。
4. **M3U 源的失效周期与更新**：
   - 公开的网络电视源具有一定的时效性。建议在播放器（Kodi / TiviMate / APTV）中开启 **“启动时刷新播放列表”** 或设置为每天自动拉取更新，避免由于流媒体 Token 变动导致黑屏。

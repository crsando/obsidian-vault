随着任天堂对模拟器社区的法律打击（Yuzu、Ryujinx 官方项目及后续的 Sudachi、Suyu 等分支均在 2024 至 2026 年间受到不同程度的 DMCA 下架），目前 Switch 模拟器已进入“地下/存档”时代。但这并不影响我们在 Steam Deck 上继续玩 Switch 游戏。以下是针对 Steam Deck 运行 Switch 模拟器的全面研究与配置指南。

---

## 一、 2026 年 Switch 模拟器现状与选择

在 Steam Deck 上运行 Switch 游戏，目前最主流的两大选择是 **Yuzu / Sudachi** 和 **Ryujinx (龙神)**。它们各有优缺点：

| 模拟器 | 优势 | 劣势 | 推荐适用场景 |
| :--- | :--- | :--- | :--- |
| **Yuzu / Sudachi** | 针对 Steam Deck 的 APU 性能优化极佳。CPU 占用低、省电、发热少，像《超级马里奥：奥德赛》、《塞尔达：荒野之息》等游戏能跑满 60 帧。 | 官方开发已被任天堂叫停，后续新发售的 Switch 游戏可能存在兼容性问题或画面贴图错误。 | 绝大多数老游戏、中轻度独立游戏，以及追求续航和稳定 60 帧的场景。 |
| **Ryujinx (龙神)** | 兼容性极强，代码还原度极高，画面错误极少。目前依然有活跃的第三方分支（如 GreemDev 的 Ryujinx 分支）在暗中更新，支持最新的游戏。 | 相比 Yuzu 更吃 CPU 性能。在 Steam Deck 上运行某些重度大作时，帧率可能偏低，且掉电非常快。 | Yuzu 运行有画面 Bug 的新游戏，或追求完美还原度、不介意续航的玩家。 |

---

## 二、 核心配置与安装步骤

### 1. 准备 EmuDeck (一键式配置管理)
**EmuDeck** 是 Steam Deck 上最强大的模拟器合集安装与管理工具，它能自动帮你搞定快捷键、按键映射、画质预设和目录结构。
- **安装方法**：
  1. 按 Steam 键进入“电源”，切换到 **桌面模式 (Desktop Mode)**。
  2. 用浏览器访问 [EmuDeck 官网](https://www.emudeck.com/)，下载 EmuDeck 安装器。
  3. 运行安装器，推荐选择 **Custom Mode (自定义模式)**。
  4. 在模拟器勾选界面中勾选 **Ryujinx** 或 **Yuzu**（如果需要单独手动装，也可以在后面用 AppImage 替换）。
  5. 安装程序会在你的内置存储或 SD 卡上创建一个叫 `Emulation/` 的大文件夹。

---

### 2. 配置 Key 和 固件 (Firmware) —— 核心步骤
由于版权法限制，模拟器本身是不自带系统密钥和固件的，必须手动导入：
- **获取 `prod.keys`（密钥文件）**：
  - **存放路径**：
    - **Ryujinx**：`Emulation/bios/ryujinx/keys/prod.keys`
    - **Yuzu**：`Emulation/bios/yuzu/keys/prod.keys`
- **获取 Firmware（固件，通常是 `.zip` 压缩包）**：
  - **Ryujinx**：启动 Ryujinx，在顶部菜单栏点击 `Tools` -> `Install Firmware` -> 选择从 `.zip` 或文件夹安装，选择下载好的固件压缩包即可。
  - **Yuzu / Sudachi**：把固件压缩包里的所有 `.nca` 文件，解压到 `/home/deck/.config/yuzu/nand/system/Contents/registered/`（如果是 EmuDeck，它会自动映射软链接，你也可以放入 EmuDeck 指定的 `bios/yuzu/firmware` 目录下）。

> 💡 **小提示**：Keys 和固件的版本一定要配套。如果要运行较新的游戏，建议下载最新的系统固件（例如 v18.0.0 或 v19.0.0+）。

---

### 3. 游戏本体与 DLC/更新档的放置规范
- **游戏本体 (ROMs)**：
  - 支持格式：`.xci` / `.nsp`。
  - 存放路径：`Emulation/roms/switch/`。
- **游戏更新包和 DLC**：
  - ⚠️ **千万不要把 DLC 和更新包放到 `roms/switch/` 目录下！** 否则 Steam ROM Manager 会把它们当成独立游戏导入，导致你的 Steam 库里出现几十个同名且无法运行的图标。
  - **正确做法**：在 `Emulation/storage/ryujinx/patchesAndDlc`（Ryujinx）中新建对应文件夹放入。
  - **导入方式**：在模拟器界面中右键游戏图标 -> 选择 `Manage Title Updates` (管理更新) 或 `Manage DLC` (管理DLC) -> 点击 `Add` 找到你放更新档的目录，勾选后点击 Save 即可。

---

### 4. 导入 Steam 游戏库 (Gaming Mode)
1. 在桌面模式打开 EmuDeck，点击 **Steam ROM Manager**。
2. 在左侧 Parser 列表中，勾选 **Nintendo Switch Ryujinx** (或 Yuzu)。
3. 点击右上角的 **Preview** -> **Parse**，它会自动扫描你的游戏并联网匹配精美的游戏封面。
4. 确认封面无误后，点击 **Save to Steam**。
5. 返回 Steam 游戏模式，你就可以在“非 Steam 游戏”或“收藏夹”中直接启动 Switch 游戏了。

---

## 三、 Steam Deck 专属优化技巧 (流畅游戏的关键)

Switch 模拟器非常吃硬件资源，如果想在 Steam Deck 上获得稳定的游戏体验，请务必进行以下调整：

1. **GPU 后端锁定 Vulkan**
   - 在模拟器的 Graphics 设置中，Graphics Backend 一定要选 **Vulkan**。Vulkan 在 SteamOS 下拥有远超 OpenGL 的着色器编译速度，能极大减少游戏过程中的卡顿 (Stuttering)。

2. **调整 Steam Deck 的 VRAM (显存) 大小**
   - 默认情况下，Steam Deck 的 VRAM 限制为 1G，其余动态分配，这会导致模拟器经常因为显存不足卡顿甚至闪退。
   - **调整方法**：
     1. 完全关机。
     2. 按住 `音量+` 键不放，按一下 `电源键`，听到“哔”声后松开。
     3. 进入 BIOS 菜单，选择 `Setup Utility` -> `Advanced`。
     4. 找到 `UMA Frame Buffer Size`，将其从 `1G` 修改为 `4G`。
     5. 保存并退出重启。

3. **启用陀螺仪 (Gyro)**
   - 很多 Switch 游戏（如《塞尔达》的射箭解密、《喷射战士》）需要体感。
   - EmuDeck 自带 **SteamDeckGyroDSU**。你可以通过 EmuDeck 的工具箱一键安装，然后在模拟器设置中开启 Motion 控制，就能直接用 Steam Deck 自身的陀螺仪来瞄准了。

4. **PowerTools 插件微调 (可选)**
   - 如果你安装了 Decky Loader，可以使用 **PowerTools** 插件。
   - 对于某些特别吃 CPU 单核性能的 Switch 游戏，可以在 PowerTools 里**关闭 SMT（超线程）**，这会让 CPU 物理核心跑得更满，对某些游戏（如《宝可梦》系列）能提升 3-5 帧的稳定性。

5. **帧率与分辨率微调**
   - 建议在模拟器中将 Resolution 设为 **1x (720p/1080p)**。如果遇到卡顿的游戏（例如《王国之泪》），可以尝试：
     - 在模拟器里把 Resolution 调为 **0.75x**。
     - 在 Steam 快捷菜单 (三个点键) 中，将 Steam Deck 屏幕的刷新率锁定在 **30Hz / 30帧**。在 30 帧下，游戏运行的平滑度会比在 35-45 帧波动要好得多。

---

## 四、 常见问题及排查 (Q&A)

### Q1: 打开游戏提示“Key area keys are missing”或无法识别游戏？
- **原因**：你的 `prod.keys` 版本太老，或者放置路径不对。
- **解决**：确保 `prod.keys` 放在正确的文件夹内（Ryujinx 为 `Emulation/bios/ryujinx/keys/`，Yuzu 为 `Emulation/bios/yuzu/keys/`）。另外，如果游戏是最新出的，需要更新对应版本的密钥。

### Q2: 游戏卡在加载界面 (Loading Screen) 或闪退？
- **原因**：常见于未安装对应版本的系统固件 (Firmware) 或着色器缓存 (Shader Cache) 损坏。
- **解决**：
  1. 确认是否已安装 Firmware。
  2. 尝试右键游戏，选择 `Cache Management` -> `Purge Shader Cache`（清除着色器缓存）后重新进游戏。
  3. 确认 UMA Frame Buffer Size 已改成了 4G。

### Q3: 使用外接手柄时，陀螺仪不起作用？
- **原因**：外接手柄（如 Switch Pro、PS5 手柄）的陀螺仪与 Steam Input 冲突。
- **解决**：在 Steam 游戏模式下，点击对应游戏 -> 控制器设置 -> 齿轮图标 -> 选择 **Disable Steam Input** (禁用 Steam 输入)，让模拟器直接读取外接手柄的硬件体感信号。

---
**来源参考与文档**：
- EmuDeck Ryujinx 官方配置：[EmuDeck Ryujinx Wiki](https://emudeck.github.io/emulators/steamos/ryujinx/)
- EmuDeck Yuzu 进阶技巧：[EmuDeck Yuzu Wiki](https://manual.emudeck.com/tricks/yuzu/)
- 2026 年 Steam Deck 运行 Switch 模拟器现状研究：[SwitchROM101](https://switchrom101.com/steam-deck-switch-emulation-in-2026-everything-you-need-to-know/)
- 记录时间：2026-07-12

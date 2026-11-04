---
date: 2026-11-04
tags:
  - Neovim
  - 配置
  - dotfiles
  - macOS
status: 已落地
---

## 摘要

一套极简 Neovim 配置：Tokyo Night 配色、四空格缩进、鼠标支持、平滑滚动。插件用 Neovim 0.12 自带的 `vim.pack` 声明式管理，配置目录本身是一个 git 仓库，因此在多台机器之间只需 `git clone` + 启动一次，插件自动装齐、版本完全一致。

核心设计原则：**`init.lua` 是唯一真相源，插件本体不进 git。**

---

## 为什么这样组织

改造前的问题：插件装在 `~/.local/share/nvim/site/pack/plugins/start/`，这是 Neovim 的**数据目录**，和 `~/.config/nvim/` 完全分离。就算把配置目录放进 git，换一台机器 clone 下来，插件还是得手动一个个装。

解决方式是把 `packadd` 那套手动加载换成 `vim.pack.add()` 声明：

```lua
vim.pack.add({
  { src = "https://github.com/folke/tokyonight.nvim" },
  { src = "https://github.com/karb94/neoscroll.nvim" },
})
```

Neovim 启动时自己检查磁盘，缺什么装什么。装完在 `~/.config/nvim/nvim-pack-lock.json` 写入每个插件的**精确 commit hash**——这个文件跟着 git 走，所以别的机器拉到的不是"最新版"，而是和你这台**完全同一个版本**，不会出现"我这好好的你那报错"。

`vim.pack` 的硬性要求：
- Neovim **0.12+**（0.11 及以前没有）
- 机器上要有 `git`
- 插件仓库的 tag 遵循 semver（`v1.2.0` 这种），否则只能用分支名

---

## 目录与文件

```
~/.config/nvim/
├── init.lua              # 全部配置，纳入 git
├── nvim-pack-lock.json   # 插件版本锁，纳入 git
└── .gitignore            # 排除 swap/备份文件
```

插件实际落在 `~/.local/share/nvim/site/pack/core/opt/`，**不纳入 git**，由 `vim.pack` 自动维护。

---

## 完整配置

### init.lua

```lua
-- Minimal Neovim: Tokyo Night, four-space indentation, mouse, and comments.
--
-- Plugins are declared below and installed by Neovim's built-in `vim.pack`
-- manager (0.12+). The exact revision of every plugin is pinned in
-- `nvim-pack-lock.json` next to this file, so cloning this config onto another
-- machine reproduces the same plugin set with no manual install step.

vim.opt.termguicolors = true
vim.opt.background = "dark"

vim.pack.add({
  { src = "https://github.com/folke/tokyonight.nvim" },
  { src = "https://github.com/karb94/neoscroll.nvim" },
})

require("tokyonight").setup({
  style = "night",
  transparent = false,
})
vim.cmd.colorscheme("tokyonight")

-- Defaults; filetypes such as Makefiles may retain their own indentation rules.
vim.opt.expandtab = true
vim.opt.tabstop = 4
vim.opt.shiftwidth = 4
vim.opt.softtabstop = 4
vim.opt.autoindent = true
vim.opt.mouse = "a"

-- Ghostty sends Ctrl-_ (0x1f) for Command+/.
-- Use Neovim's built-in, filetype-aware comment operator.
vim.keymap.set("n", "<C-_>", "gcc", { remap = true, desc = "Toggle line comment" })
vim.keymap.set("x", "<C-_>", "gc", { remap = true, desc = "Toggle selected line comments" })

-- Smooth scrolling (neoscroll.nvim)
local neoscroll = require("neoscroll")
neoscroll.setup({
  mappings = {}, -- disable defaults, we define our own below
  hide_cursor = true,
  stop_eof = true,
  respect_scrolloff = false,
  cursor_scrolls_alone = true,
  easing = "quadratic",
})

local scroll_keys = {
  -- Half-page: the daily driver for reading code.
  ["<C-d>"] = function() neoscroll.ctrl_d({ duration = 200, easing = "quadratic" }) end,
  ["<C-u>"] = function() neoscroll.ctrl_u({ duration = 200, easing = "quadratic" }) end,
  -- Full-page, kept for when you really want to jump.
  ["<C-f>"] = function() neoscroll.ctrl_f({ duration = 320, easing = "quadratic" }) end,
  ["<C-b>"] = function() neoscroll.ctrl_b({ duration = 320, easing = "quadratic" }) end,
  -- Fine single-line nudge, cursor stays put.
  ["<C-e>"] = function() neoscroll.scroll(0.1, { move_cursor = false, duration = 90 }) end,
  ["<C-y>"] = function() neoscroll.scroll(-0.1, { move_cursor = false, duration = 90 }) end,
  -- Keep the target line centred after centring commands.
  ["zt"] = function() neoscroll.zt({ half_win_duration = 150 }) end,
  ["zz"] = function() neoscroll.zz({ half_win_duration = 150 }) end,
  ["zb"] = function() neoscroll.zb({ half_win_duration = 150 }) end,
}

for key, func in pairs(scroll_keys) do
  vim.keymap.set({ "n", "x" }, key, func, { desc = "Smooth scroll" })
end

-- Trackpad / mouse wheel: 3 lines per notch, short animation so rapid
-- two-finger swipes chain together instead of queueing up and feeling laggy.
vim.keymap.set({ "n", "x" }, "<ScrollWheelUp>", function()
  neoscroll.scroll(-3, { move_cursor = false, duration = 60, easing = "linear" })
end, { desc = "Smooth wheel scroll up" })
vim.keymap.set({ "n", "x" }, "<ScrollWheelDown>", function()
  neoscroll.scroll(3, { move_cursor = false, duration = 60, easing = "linear" })
end, { desc = "Smooth wheel scroll down" })
```

### nvim-pack-lock.json

不要手改，由 `vim.pack` 维护。当前内容：

```json
{
  "plugins": {
    "neoscroll.nvim": {
      "rev": "c8d29979cb0cb3a2437a8e0ae683fd82f340d3b8",
      "src": "https://github.com/karb94/neoscroll.nvim"
    },
    "tokyonight.nvim": {
      "rev": "cdc07ac78467a233fd62c493de29a17e0cf2b2b6",
      "src": "https://github.com/folke/tokyonight.nvim"
    }
  }
}
```

### .gitignore

```gitignore
*.swp
*.swo
*~
.DS_Store
```

---

## 平滑滚动映射表

| 按键 | 作用 | 动画时长 |
| --- | --- | --- |
| `Ctrl+d` / `Ctrl+u` | 半屏上下翻 | 200ms |
| `Ctrl+f` / `Ctrl+b` | 整屏上下翻 | 320ms |
| `Ctrl+e` / `Ctrl+y` | 单行微调（光标不动） | 90ms |
| `zt` / `zz` / `zb` | 行定位到顶 / 中 / 底 | 150ms |
| 触摸板 / 滚轮 | 每次 3 行 | 60ms |

几个刻意的取舍：

- **滚轮只给 60ms**：触摸板会连续发高频滚轮事件，动画太长会排队，滑起来反而粘手
- **滚轮设 `move_cursor = false`**：双指滑动只动视窗，光标不跟着跑，避免误操作
- **缓动用 `quadratic`**：起停有加减速，比默认的 `linear` 顺眼很多
- **接管 `zt` / `zz` / `zb`**：这三个原生是瞬跳，接管后和整体手感统一

---

## 新机器接入

```bash
# 1. 装 neovim 0.12+
brew install neovim

# 2. 克隆配置
git clone <仓库地址> ~/.config/nvim

# 3. 启动，插件自动安装
nvim
```

没有第四步。首次启动会打印：

```
vim.pack: Installing plugins (0/2)
vim.pack: 100% Installing plugins (2/2)
```

装完直接用。

---

## 日常维护

### 加插件

1. 往 `vim.pack.add({...})` 里加一行
2. 重启 Neovim，自动安装
3. `git add -A && git commit`

### 更新插件

```vim
:lua vim.pack.update()
```

会开一个 tab 展示每个插件的 diff，确认后才写盘。想先看不动手就加 `{ offline = true }`。

### 冻结某个插件不跟随更新

把 `init.lua` 里该插件的 `version` 设成锁文件里的当前 `rev`（形如 `abc12345`），重启即可。

### 回滚一次更新

```bash
git checkout HEAD -- nvim-pack-lock.json
```

然后 `:restart`，再执行 `:lua vim.pack.update({ '插件名' }, { offline = true, target = 'lockfile' })`。

### 跨机同步

改完配置 `git commit && git push`，另一台 `git pull`，重启 Neovim。锁文件保证两边插件版本一致。

---

## 注意事项

- **所有机器必须 Neovim 0.12+**。如果某台机台是旧版本（比如 Ubuntu 源里的 0.9），`vim.pack` 不存在，整个配置起不来。要么升级，要么退回 `lazy.nvim` 那套写法。
- `nvim-pack-lock.json` **必须提交**，这是跨机版本一致的关键。
- 插件目录 `~/.local/share/nvim/` **不要**纳入 git，那是机器本地状态。
- 换机器时如果从旧配置迁移，记得把 `~/.local/share/nvim/site/pack/` 下的旧目录清掉，避免和 `vim.pack` 托管目录（`pack/core/opt`）冲突。

---

## 相关

- `:help vim.pack` — 内置插件管理器文档
- `:help pack` — Neovim 的 package 机制
- `:help neoscroll-options` — 平滑滚动全部可调参数

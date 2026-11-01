---
date: 2026-11-01
tags:
  - network
  - 旁路由
  - shellcrash
  - mihomo
status: 待部署
device: NanoPi R4S · 192.168.5.5
---

# 旁路由 ShellCrash 梳理与 mihomo 自管配置方案

> [!info] 当前状态（2026-11-01 23:30）
> 新配置和部署脚本已经上传到旁路由，**还没执行**。ShellCrash 仍在用旧配置跑，网络正常。
> 方案文件在 Alma 工作区 `shellcrash-usage-improvements/outputs/`：`PLAN.md`、`config.yaml`、`deploy.sh`

> [!warning] 敏感信息不入库
> 订阅链接和硅谷节点的 UUID 只存在于旁路由的 `/root/alma-new-config.yaml`（权限 600）和 Alma 工作区里，本笔记**有意不记录**，因为这个 vault 会同步到 GitHub。

## 1. 环境

| 项目 | 值 |
|---|---|
| 设备 | NanoPi R4S，Armbian 26.8.3 (Ubuntu 24.04)，aarch64 |
| 网卡 / IP | `enp1s0`，192.168.5.5/24；主路由 192.168.5.1 |
| ShellCrash | `/etc/ShellCrash`，systemd 服务 `shellcrash`，命令 `crash` 进菜单 |
| 内核 | mihomo (meta) v1.19.17，运行于 `/tmp/ShellCrash/CrashCore` |
| 模式 | Tproxy + iptables，DNS mix（fake-ip + 国内真实 IP） |
| 面板 | `http://192.168.5.5:9999/ui`（zashboard） |
| dae | 已安装（`/etc/dae/config.dae`），服务 disabled，未使用 |

## 2. ShellCrash 到底是什么

ShellCrash 本身**不是代理**，只是一堆 shell 脚本，负责管理 mihomo 内核：

**订阅 + 自定义规则 → 拼成 config.yaml → 启动内核 → 加 iptables 把局域网流量劫持过去**

关键点：`config.yaml` **只在启动时生成一次**。改了 `yamls/rules.yaml` 之后必须重启才生效。

日常操作速查：

| 想做什么 | 方法 |
|---|---|
| 切节点、看连接、看日志 | 面板 `:9999/ui` |
| 启停、看状态 | `systemctl start/stop/restart/status shellcrash` |
| 改模式、防火墙、定时任务 | SSH 上去执行 `crash` |
| ShellCrash 自身日志 | `/tmp/ShellCrash/ShellCrash.log` |

## 3. 今天发现的问题

1. **新加的规则没生效**：`rules.yaml` 是 21:29 改的，但当前运行的配置是凌晨 03:00 定时重启时生成的。另外 `icloud.com.cn` 重复了一遍，`alma.now` 这条域名规则加了 `no-resolve`，没有意义。
2. **订阅早就失效了**：旧链接 302 → `/xxx` → 403。本地节点一直是 **2026-03-24** 的缓存，凌晨 1 点的"热更新订阅"任务一直在**静默失败**。
3. **只代理了 5 个端口**：iptables 只劫持目标端口 22/80/443/8080/8443，其他端口的流量全部直连，不经过分流规则。
4. **没有自动切换**：只有一个手动选择组，固定在"新加坡 01"，这个节点一挂就断网。
5. **IPv6 半开**：DNS 设置了 `ipv6: true`，但没有 ip6tables 规则，IPv6 流量会绕过代理（局域网实际有没有 IPv6 还没确认）。
6. **面板没有密码**：`secret` 为空，9999 端口对整个局域网开放。
7. **订阅自带的杂项多**：运行配置 457KB、9053 条规则，还有一行 Mac 网卡名 `interface-name: en0`，说明这份配置原本是给 Mac 客户端写的。

## 4. 新的使用方式：ShellCrash 只管启停，配置自己写

ShellCrash 有个开关 `disoverride=1`（菜单里叫"禁用配置覆写"）。打开后启动时直接把 `yamls/config.yaml` 原样软链接给内核，不再拿订阅拼配置（代码见 `starts/bfstart.sh`）。

分工：
- **ShellCrash**：启停内核、iptables 透明代理、定时任务
- **config.yaml（手写）**：节点来源、策略组、规则、DNS

> [!important] 必须和 ShellCrash 防火墙对上的参数
> `tproxy-port: 7893` · `redir-port: 7892` · `mixed-port: 7890` · `dns.listen: :1053` · `fake-ip-range: 198.18.0.0/15` · `routing-mark: 7894`（= 0x1ed6，不对会回环）· `external-controller: :9999`

## 5. 分流设计

```
所有流量
 ├─ 局域网 / Tailscale / 自定义直连 ─────────→ DIRECT
 ├─ AI 服务 → 🤖 AI → 🇺🇸 硅谷-链式 (vmess)
 │                     └─ dialer-proxy: 🔗 链式前置 → ♻️ 新加坡优先
 ├─ 国内域名 / 国内 IP ──────────────────────→ DIRECT
 └─ 其余 → 🚀 节点选择 → ♻️ 新加坡优先 (fallback)
                          ├─ 🇸🇬 新加坡 (url-test)  ← 优先
                          └─ 🇭🇰 香港   (url-test)  ← 新加坡全挂才切
```

- **节点来源**：`proxy-providers` 引用机场订阅（新链接需带 `?target=clash`），每天自动刷新，只取节点，过滤掉"套餐到期""订阅时间"这类信息节点。套餐到期 **2027-03-28**，流量每月 29 日重置，总量 360G。
- **硅谷节点**：vmess，`43.159.134.126:52703`，从 Shadowrocket 导入。
  - 坑：Shadowrocket 导出的 JSON 里，`uuid` 字段只是它内部的记录 ID，**真正的 vmess UUID 在 `password` 字段里**。
- **AI 规则**：Anthropic/Claude、OpenAI/ChatGPT、Gemini/AI Studio/NotebookLM、xAI/Grok、Perplexity、Mistral、OpenRouter、Cursor、Copilot、Poe、Meta AI、ElevenLabs、Midjourney、Suno、Windsurf。旁路由的 GeoSite.dat 里没有 `category-ai` 分类，所以全部用显式域名写。
- **有意不走 AI 链**：aigocode、dragoncode、ofox.io 这类国内中转 API 保持直连。

## 6. 验证结果（隔离实例，没有碰运行中的服务）

在 `/tmp` 里另起一个 CrashCore 实例，端口错开（17890/19999），不接管 DNS、不加防火墙规则，测完已删除。

| 检查 | 结果 |
|---|---|
| `CrashCore -t` 语法检查 | ✅ |
| 新订阅 | ✅ 约 3 秒拉取完成，可用节点 29 个；新加坡、香港、美国各 3 个 |
| 普通流量出口 | ✅ 新加坡 Zenlayer，约 60ms |
| claude.ai / chatgpt.com 出口 | ✅ `43.159.134.126`，loc=US，colo=SJC，链路约 168–187ms |
| 百度直连 | ✅ 200，约 0.12s |

## 7. 部署（自己执行）

旁路由上已经放好：
- `/root/alma-deploy.sh`（700）：备份 → 预下载订阅 → 写入配置和开关 → 重启 → 健康检查，检查不通过**自动回滚**
- `/root/alma-new-config.yaml`（600）

```bash
ssh root@192.168.5.5
nohup bash /root/alma-deploy.sh >/root/alma-deploy.log 2>&1 &
tail -f /root/alma-deploy.log      # 看到 "🎉 部署成功" 即完成
```

脚本做的事情：
- 设置 `disoverride=1`
- 清空失效的旧订阅 `Https=`
- 删除定时任务 105（每天 1 点热更新订阅）
- 把 `web_save` 改成新的策略组默认值

回滚：
```bash
bash /root/alma-deploy.sh rollback /root/shellcrash-backup-<时间>
```

最后手段：把设备的网关和 DNS 改回主路由 `192.168.5.1`，可以直连上网（没有代理）。

### 断网影响

- 只有重启那一步会断，**约 10–15 秒**。期间用旁路由做 DNS 的设备完全无法解析域名，长连接（视频会议、下载、SSH、AI 流式回复）会断开。
- 如果自动回滚被触发，最坏情况总共断 30–60 秒。
- 切换后 AI 服务的出口从新加坡变成美国，Claude/OpenAI 可能要求重新登录或验证。

### 部署后验证

- [ ] 面板里能看到 ♻️ 新加坡优先 / 🤖 AI / 🔗 链式前置
- [ ] 浏览器打开 `https://claude.ai/cdn-cgi/trace`，结果里有 `loc=US`
- [ ] 普通网站、国内网站访问正常

## 8. 后续建议（每项都要再重启一次，分开做）

- [ ] **防火墙改成全部端口**：在 `crash` 菜单 → 防火墙设置里改。不改的话，走非标准端口的 AI 流量不会经过链式代理。
- [ ] **设置面板密码**：填 `config.yaml` 里的 `secret`。
- [ ] **处理 IPv6**：先确认主路由有没有下发 IPv6。有的话，要么在主路由关掉，要么开启 ShellCrash 的 IPv6 透明代理，否则 AI 服务可能暴露真实 IP。
- [ ] 以后改规则的流程：改 `config.yaml` → 在旁路由上 `CrashCore -t` 检查 → 备份 → 替换 → `systemctl restart shellcrash`。

## 9. 经验

- ShellCrash 的订阅更新失败**不会报错**，要定期看面板里节点的"订阅获取时间"。
- 改了 ShellCrash 的任何配置都要重启才生效，这是它"难用"的主要原因。
- 验证新配置时，用隔离实例（错开端口、不碰防火墙）测完再上线，可以做到零风险。
- mihomo 的 http provider 首次下载是异步的，刚启动时节点组可能是空的，测试时要多等几秒。

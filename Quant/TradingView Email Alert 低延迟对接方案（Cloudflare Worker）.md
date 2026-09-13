# TradingView Email Alert 低延迟对接方案（Cloudflare Email Routing + Worker）

> **目标**：将 TradingView 的 Email 警报以最低延迟（< 1~2s）转化为 HTTP Webhook 推送到独立部署的交易/处理程序。  
> **核心链路**：`TradingView SMTP` ➔ `Cloudflare MX / Email Routing` ➔ `Cloudflare Worker (解析 MIME / JSON)` ➔ `HTTP POST` ➔ `独立部署服务 (FastAPI / Express / Go)`。

---

## 一、 核心优势与延迟表现

1. **延迟极低**：Cloudflare 全球 Anycast 边缘接收 SMTP，Worker 在边缘节点就近触发并转发，流式/内存解析，无任何磁盘读写或数据库开销（处理耗时通常 < 150ms）。
2. **零服务器维护**：无需自建并维护 25 端口 SMTP 服务器，免去防垃圾邮件、黑名单、TLS 证书等繁琐运维。
3. **免费额度充足**：Cloudflare Email Routing 与 Worker 免费层即可支撑每天数万次警报触发。

---

## 二、 步骤 1：Cloudflare Email Routing 配置

### 1. 准备专属子域名
建议使用独立二级/三级域名（如 `alert.yourdomain.com`），避免与主域名日常企业邮箱冲突。

### 2. 开启 Email Routing 并配置 DNS MX 记录
1. 登录 Cloudflare Dashboard，选择你的域名。
2. 进入左侧菜单 **Email** ➔ **Email Routing**。
3. 如果未开启，点击 **Get started** 开启服务。
4. Cloudflare 会提示自动添加必要的 DNS 记录（包括 3 条 MX 记录与 1 条 TXT SPF 记录），点击 **Add records automatically**。
   - `isaac.mx.cloudflare.net` (Priority 13)
   - `amir.mx.cloudflare.net` (Priority 49)
   - `linda.mx.cloudflare.net` (Priority 9)
   - `v=spf1 include:_spf.mx.cloudflare.net ~all`

---

## 三、 步骤 2：创建并部署 Cloudflare Worker

### 方式 A：通过 Cloudflare 控制台直接创建（最简单）

1. 在 Cloudflare Dashboard 进入 **Workers & Pages** ➔ **Create application** ➔ **Create Worker**。
2. 命名为 `tradingview-alert-receiver`，点击 **Deploy**。
3. 点击 **Edit code**，将下方 Worker 代码粘贴进去。

### Worker 源码（JavaScript / ES Module）：

```javascript
import PostalMime from 'postal-mime';

export default {
  async email(message, env, ctx) {
    const startTime = Date.now();
    const rawEmail = await new Response(message.raw).arrayBuffer();
    
    // 解析 MIME 邮件
    const parser = new PostalMime();
    const parsedEmail = await parser.parse(rawEmail);

    const fromAddress = message.from; // 例如 noreply@tradingview.com
    const subject = message.headers.get('subject') || parsedEmail.subject || '';
    const textBody = (parsedEmail.text || '').trim();

    // 基础安全过滤：校验发件人来源
    if (!fromAddress.includes('tradingview.com') && !fromAddress.includes('noreply@tradingview.com')) {
      console.warn(`[Blocked] Unauthorized sender: ${fromAddress}`);
      return;
    }

    // 尝试提取正文中的 JSON
    let alertData = null;
    try {
      alertData = JSON.parse(textBody);
    } catch (e) {
      // 若正文非纯 JSON，保留原始文本与提取信息
      alertData = {
        raw_text: textBody,
        subject: subject,
        parsed_as_json: false
      };
    }

    // 构造推送到独立程序的 Payload
    const payload = {
      source: "tradingview_email",
      timestamp: Date.now(),
      sender: fromAddress,
      subject: subject,
      data: alertData,
      process_latency_ms: Date.now() - startTime
    };

    // 目标独立程序 Webhook 地址（支持在 Worker 环境变量中配置 TARGET_WEBHOOK_URL）
    const targetUrl = env.TARGET_WEBHOOK_URL || "https://api.yourdomain.com/webhook/tradingview";
    const webhookSecret = env.WEBHOOK_SECRET || "your-custom-secret-key";

    try {
      const response = await fetch(targetUrl, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "X-Alert-Secret": webhookSecret,
          "User-Agent": "Cloudflare-Worker-Alert-Gateway/1.0"
        },
        body: JSON.stringify(payload)
      });

      if (!response.ok) {
        console.error(`Forward failed: HTTP ${response.status} - ${await response.text()}`);
      } else {
        console.log(`Alert forwarded successfully in ${Date.now() - startTime}ms`);
      }
    } catch (err) {
      console.error(`Forward request error: ${err.message}`);
    }
  }
};
```

> **依赖说明**：在 Worker 设置中的 `package.json`（或通过 Wrangler）添加 `"postal-mime": "^2.3.0"`。如果在 Web 编辑器中无法直接 import npm 包，可以使用轻量正则提取正文，或者在本地用 `wrangler` 部署。

---

### 方式 B：使用 Wrangler 本地工程化部署（推荐）

1. 初始化工程：
   ```bash
   npm create cloudflare@latest tv-alert-worker -- --type hello-world-esm
   cd tv-alert-worker
   npm install postal-mime
   ```
2. 修改 `wrangler.toml`：
   ```toml
   name = "tv-alert-worker"
   main = "src/index.js"
   compatibility_date = "2026-09-01"

   [vars]
   TARGET_WEBHOOK_URL = "https://your-server-domain.com/api/v1/tv-alert"
   WEBHOOK_SECRET = "your_secure_auth_token_here"
   ```
3. 部署：
   ```bash
   npx wrangler deploy
   ```

---

## 四、 步骤 3：绑定 Email Routing 到 Worker

1. 返回 Cloudflare Dashboard ➔ **Email** ➔ **Email Routing** ➔ **Routing Rules**。
2. 点击 **Create custom address**：
   - **Custom address**：例如输入 `tv-alert@alert.yourdomain.com`（或主域名下前缀）。
   - **Action**：选择 **Send to a Worker**。
   - **Destination**：选择刚才创建的 Worker（`tradingview-alert-receiver` 或 `tv-alert-worker`）。
3. 保存并启用规则。

---

## 五、 步骤 4：独立程序端接收示例（FastAPI / Node.js）

### Python (FastAPI) 示例：

```python
from fastapi import FastAPI, Header, HTTPException, Request
from pydantic import BaseModel
from typing import Optional, Any
import time

app = FastAPI()

AUTH_SECRET = "your_secure_auth_token_here"

class AlertPayload(BaseModel):
    source: str
    timestamp: int
    sender: str
    subject: str
    data: Any
    process_latency_ms: Optional[int] = 0

@app.post("/api/v1/tv-alert")
async def handle_tradingview_alert(
    payload: AlertPayload,
    x_alert_secret: Optional[str] = Header(None)
):
    # 1. 签名/秘钥校验
    if x_alert_secret != AUTH_SECRET:
        raise HTTPException(status_code=401, detail="Unauthorized")

    received_at = int(time.time() * 1000)
    total_relay_latency = received_at - payload.timestamp
    
    alert_info = payload.data
    print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] 收到警报: {alert_info} (中转耗时: {total_relay_latency}ms)")

    # 2. 业务处理（如触发下单、消息推送、开平仓逻辑）
    # execute_trade(alert_info)

    return {"status": "ok", "relay_latency_ms": total_relay_latency}
```

---

## 六、 步骤 5：TradingView 端的警报配置

1. **设置通知邮箱**：
   - 进入 TradingView **Profile Settings** ➔ **SMS / Email** 设置，将通知接收邮箱绑定为你创建的 `tv-alert@alert.yourdomain.com`（完成邮件验证）。
2. **创建警报（Alert）**：
   - 在通知方式中勾选：**Send email**（发送电子邮件）。
   - 在 **Message** 中直接填入标准 JSON：
     ```json
     {
       "ticker": "{{ticker}}",
       "exchange": "{{exchange}}",
       "action": "BUY",
       "price": {{close}},
       "strategy": "EMA_Cross_15M",
       "time": "{{timenow}}"
     }
     ```
   - 这样 Worker 收到邮件时，正文无需任何复杂正则解析，直接 `JSON.parse` 即可无损还原结构化数据。

---

## 七、 生产落地关键优化与注意事项

1. **冷启动与网络延迟**：
   - Cloudflare Workers 是 V8 隔离区启动，几乎零冷启动（< 5ms）。
   - 下游独立服务器建议部署在 **AWS us-east-1 (Virginia)** 或 **Cloudflare 节点就近机房**，进一步消除网络 RTT。
2. **防重放与幂等处理**：
   - 邮件重试或 TradingView 短时抖动可能发送重复警报。独立程序端建议基于 `ticker + time + action` 维护一个 5 秒内的 Redis 幂等去重 Key。
3. **安全防护**：
   - Worker 与独立程序之间务必携带 `X-Alert-Secret` Token，或通过 Cloudflare Tunnel / mTLS / IP 白名单限制仅允许 Cloudflare 边缘 IP 访问你的 Webhook 接口。

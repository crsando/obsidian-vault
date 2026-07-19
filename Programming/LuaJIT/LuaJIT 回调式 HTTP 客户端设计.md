# LuaJIT 回调式 HTTP 客户端设计

## 1. 目标与范围

本库为 LuaJIT 提供接近 Python `requests` 的 HTTP 客户端体验，但异步接口固定采用 **libuv/luv 回调模式**，不提供 `async`、`await` 或 coroutine API。

- HTTP/HTTPS、重定向、Cookie、代理、TLS、压缩和连接复用由 **libcurl** 提供。
- 非阻塞 I/O 由 **libcurl multi socket API** 与 **libuv** 的 `uv_poll_t` / `uv_timer_t` 协作完成。
- 应用通过 `luv` 的 `uv.run()` 驱动同一个事件循环。
- V1 覆盖 GET、POST、PUT、PATCH、DELETE、HEAD、OPTIONS，支持 JSON、表单、raw body、超时、取消和 Session。

非目标：HTTP/3、WebSocket、浏览器兼容 API、自动重试、磁盘缓存、Promise/coroutine 调度器。

## 2. 总体架构

```text
Lua 应用
  |
  | requests.get/post(..., callback)
  v
requests.lua                 参数规范化、便捷方法、Response 对象
  |
  v
requests.core (Lua C 模块)   Lua C API、请求生命周期、资源清理
  |                         |
  |                         +-- libuv: uv_poll_t / uv_timer_t
  v
libcurl CURLM multi socket API
  |
  v
HTTP / HTTPS 网络连接
```

采用「Lua 门面 + 薄 C 核心」两层：Lua 层负责易用 API、URL 参数、可选 JSON 编解码；C 层负责 `curl_easy_setopt`、libcurl 回调、libuv handle、`luaL_ref` 与内存生命周期。变参 C API 和 native callback 不应暴露给 FFI 层直接维护。

## 3. 关键技术决策

| 问题 | 决策 | 原因 |
| --- | --- | --- |
| HTTP 引擎 | `CURLM` + multi socket API | 真正非阻塞，具备成熟 HTTP/TLS 能力。 |
| 事件循环 | `uv_poll_t` + `uv_timer_t` | socket 就绪时才推进 libcurl，不占用线程池。 |
| 回调风格 | `callback(err, response)` | 与 luv 使用方式一致，调用方清晰处理成功和失败。 |
| HTTP 4xx/5xx | 返回 `response`，不作为 `err` | HTTP 已成功完成；网络/传输失败才是 `err`。 |
| 线程模型 | 单线程 | `lua_State`、libuv handle、`CURLM` 均不跨线程。 |
| 连接复用 | 每 event loop 一个共享 `CURLM` | 利用 libcurl 的 connection cache；Cookie 不共享。 |

不应使用 `uv_queue_work()` 包装 `curl_easy_perform()`。那会将阻塞 I/O 移到后台，但失去 libcurl multi 的事件驱动、取消、限流和连接复用优势。

## 4. 事件循环与打包约束

核心模块使用 `uv_default_loop()`；Lua 应用通过 luv 的 default loop 执行：

```lua
local uv = require("luv")
local requests = require("requests")

requests.get("https://example.com", nil, function(err, response)
  -- 处理结果
end)

uv.run()
```

`luv` 与 `requests.core` 必须链接到**同一份 libuv 动态库实例**。不要让一边静态链接 libuv、另一边链接不同的动态库，否则可能出现两个 default loop，表现为请求永远不完成。发布包应共用同一份 `libuv.dll` / `libuv.so` / `libuv.dylib`，并在 CI 里做联合集成测试。

回调在执行 `uv.run()` 的同一 OS 线程中触发：

- 不允许工作线程直接操作 `lua_State`、`CURLM`、`CURL` 或 uv handles。
- completion callback 不能 yield，也不应执行长时间 CPU 任务。
- callback 内可安全发起新请求；新请求从下一轮 loop 开始推进。
- 库不调用 `uv_stop()` 或 `uv_loop_close()`，loop 的所有权属于应用/luv。

## 5. 公开 API

### 5.1 Client / Session

`Client` 相当于 Python 的 `requests.Session`，隔离 Cookie、默认配置和并发限制。

```lua
local requests = require("requests")

local api, err = requests.new({
  base_url = "https://api.example.com/v1/",
  headers = { Accept = "application/json" },
  user_agent = "my-service/1.0",
  connect_timeout = 3_000,
  timeout = 15_000,
  follow_redirects = true,
  max_redirects = 10,
  verify_tls = true,
  max_concurrency = 32,
})
```

顶层 `requests.get()`、`requests.post()` 等委托给惰性创建的 `requests.default`。长期运行服务推荐显式创建并关闭 Client。

```lua
client:close(function(err)
  -- Client 内所有请求已终止且回调已投递
end)
```

### 5.2 请求接口

```lua
local request, err = client:request("POST", url, opts, callback)

client:get(url, opts, callback)
client:post(url, opts, callback)
client:put(url, opts, callback)
client:patch(url, opts, callback)
client:delete(url, opts, callback)
client:head(url, opts, callback)
client:options(url, opts, callback)
```

参数校验失败、Client 已关闭等同步错误返回 `nil, err`，不会触发 callback。成功排队后立即返回 `Request`；最终 callback **恰好调用一次**。

```lua
local req = client:get("/users", { params = { page = 1 } }, function(err, response)
  if err then
    logger.error(err.kind, err.message)
    return
  end

  if not response.ok then
    logger.warn("HTTP", response.status_code)
    return
  end

  local data, json_err = response:json()
end)

req:cancel("view_closed")  -- 幂等
req:is_pending()
req:id()
```

### 5.3 opts 规范

```lua
{
  params = { page = 1, tag = { "lua", "http" } },
  headers = { ["Accept"] = "application/json" },
  header_list = { { "X-Trace-ID", "abc" } },
  data = "raw body",
  json = { name = "Ada" },
  form = { username = "ada", password = "..." },
  files = { -- V2
    avatar = { path = "avatar.png", content_type = "image/png" },
  },
  timeout = 15_000,
  connect_timeout = 3_000,
  follow_redirects = true,
  max_redirects = 10,
  verify_tls = true,
  ca_file = "/path/to/ca.pem",
  proxy = "http://127.0.0.1:7890",
  stream = false,
  on_data = function(chunk) end,
}
```

规则：

- `data`、`json`、`form` 互斥。
- `params` 的数组编码为重复 key，例如 `tag=lua&tag=http`。
- `headers` 用于唯一 header；需要重复 header 时使用 `header_list`。
- 全部 timeout 使用毫秒；`0` 表示不设置该项限制。
- `verify_tls` 默认 `true`。
- `stream = true` 时 `response.body` 为 `nil`，防止无界内存累积。

## 6. Response 与错误模型

### 6.1 Response

```lua
response.status_code       -- number，例如 200
response.url               -- 最终 URL
response.headers           -- 小写 header name -> 合并值
response.header_list       -- 保留原始顺序和重复项
response.body              -- string，stream 时为 nil
response.content_length    -- number 或 nil
response.elapsed_ms        -- number
response.redirect_count    -- number
response.ok                -- 200 <= status_code < 400
response.curl_code         -- 调试用途

response:text()            -- V1 返回原始 body
response:json()            -- value 或 nil, err
response:raise_for_status()
```

Header map 统一用小写 key；`header_list` 专门保留 `Set-Cookie` 等重复 header 的真实语义。

### 6.2 err

```lua
{
  kind = "timeout" | "connect" | "tls" | "dns" |
         "cancelled" | "curl" | "internal" | "closed" |
         "body_too_large",
  message = "human-readable message",
  curl_code = 28,       -- 可选
  request_id = 42,
  url = "https://...",
}
```

HTTP 404、429、500 不属于 `err`。例如调用方可通过 `response.status_code` 判断 429，或显式调用 `response:raise_for_status()`。这样不会把「拿到一个完整 HTTP 错误响应」和「网络不可达」混为一谈。

## 7. libcurl 与 libuv 集成

### 7.1 Engine

每个 libuv loop 维护一个 Engine：

```text
Engine
  CURLM *multi
  uv_loop_t *loop
  uv_timer_t curl_timer
  map<curl_socket_t, SocketWatcher>
  map<CURL *, RequestNative *>
  closing flag
```

所有 Client 共享 Engine 的 `CURLM`，因此可共享空闲连接与 keep-alive。Client 自己持有默认 options 与 Cookie jar，不共享 Cookie、Authorization 或 Lua callback。

### 7.2 Socket 映射

初始化 multi：

```text
CURLMOPT_SOCKETFUNCTION -> on_curl_socket
CURLMOPT_SOCKETDATA     -> Engine
CURLMOPT_TIMERFUNCTION  -> on_curl_timer
CURLMOPT_TIMERDATA      -> Engine
```

`on_curl_socket` 的映射：

| libcurl action | libuv 动作 |
| --- | --- |
| `CURL_POLL_IN` | `uv_poll_start(..., UV_READABLE, ...)` |
| `CURL_POLL_OUT` | `uv_poll_start(..., UV_WRITABLE, ...)` |
| `CURL_POLL_INOUT` | 同时监听读和写 |
| `CURL_POLL_REMOVE` | stop，再 `uv_close()` |

每个 socket 保存 `SocketWatcher`，并通过 `curl_multi_assign()` 建立关联。收到 `CURL_POLL_REMOVE` 后不可立即释放 watcher，必须在对应 `uv_close` callback 中释放。

### 7.3 Poll 与 timer

`uv_poll_t` 回调将 `UV_READABLE`、`UV_WRITABLE`、错误状态映射为 `CURL_CSELECT_IN`、`CURL_CSELECT_OUT`、`CURL_CSELECT_ERR`：

```text
curl_multi_socket_action(multi, socket_fd, curl_events, &running)
drain_completed_transfers(engine)
```

libcurl 的 timer callback 控制唯一的 `uv_timer_t`：

- timeout `< 0`：停止 timer。
- timeout `== 0`：启动 zero-delay one-shot timer。
- timeout `> 0`：启动对应毫秒数的 one-shot timer。

Timer 触发时调用：

```text
curl_multi_socket_action(multi, CURL_SOCKET_TIMEOUT, 0, &running)
drain_completed_transfers(engine)
```

每次 socket/timer 推进后只调用一次 `curl_multi_socket_action()`。后续 I/O 需求让 libcurl 通过 socket/timer callback 再次表达，不能手写忙循环。

### 7.4 完成处理

`drain_completed_transfers()` 循环读取 `curl_multi_info_read()`：

1. 处理 `CURLMSG_DONE`。
2. 通过 `CURLINFO_PRIVATE` 找回 `RequestNative`。
3. 获取 HTTP status、effective URL、重定向数和耗时。
4. `curl_multi_remove_handle()`。
5. 按 `CURLcode` 构造 `Response` 或 `err`。
6. 以 `lua_pcall` 调用 completion callback。
7. 解除 `luaL_ref`，释放 body buffer、header list、easy handle 和请求对象。

用户 callback 抛错不能破坏 Engine；异常记录到可配置 logger，清理流程仍要完成。

## 8. 请求生命周期

```text
new -> queued -> running -> completed -> callback_called -> released
                  |
                  +-> cancelling -> callback_called -> released
```

- callback 一旦调用，请求不得再次触发完成事件。
- `cancel()` 幂等；取消的最终 callback 形如 `err.kind = "cancelled"`。
- Client 关闭后拒绝新请求。
- Request userdata 被 GC 时不自动取消网络请求；自动取消的行为难以预测，`client:close()` 才是明确释放边界。
- callback 的 Lua registry ref 从排队开始持有，在最终 callback 返回后释放。

## 9. Session、Cookie 与连接复用

Client 对应 Session，负责：

- base URL、默认 headers、超时、TLS、代理。
- 独立 Cookie jar。
- `max_concurrency` 限制及 Client 内等待队列。
- active / queued / completed / failed / cancelled 统计。

每个启用 Cookie 的 Client 使用专属 `CURLSH`，每个 easy handle 使用 `CURLOPT_SHARE` 关联到该 Client。实现必须注册 `CURLSH` lock/unlock callback，即使 V1 为单线程，也要保留正确的同步边界。

连接缓存由共享 `CURLM` 持有。不同 Client 可复用相同 origin 的空闲 HTTP 连接，但绝不能共享 Cookie、默认 Authorization 或 Lua 引用。

## 10. Body 与 JSON

默认模式下，libcurl write callback 将数据追加到 C buffer。默认建议 `max_body_size = 16 MiB`；超限时中止传输并返回 `err.kind = "body_too_large"`。

流式模式：

```lua
client:get(url, {
  stream = true,
  on_data = function(chunk)
    file:write(chunk)
  end,
}, callback)
```

V1 要求 `on_data` 同步且不能 yield；返回 `false` 或抛错时终止请求。后续若需要背压，应新增 `pause()/resume()` 并调用 `curl_easy_pause()`，不能让 C write callback 阻塞。

JSON 由可注入 codec 提供：

```lua
requests.configure({ json = require("cjson.safe") })
```

`opts.json` 自动 encode 并补全 `Content-Type: application/json`；`response:json()` 调用 codec decode。未配置 codec 时返回 `nil, { kind = "json_unavailable" }`。

## 11. 关闭顺序

`client:close(callback)`：

1. 标记 closing，拒绝新请求。
2. 取消 Client 的 queued/running request。
3. 等这些 request 的 completion callback 全部投递。
4. 释放 Client `CURLSH`、默认 headers 和 Lua references。
5. 调用 close callback。

进程级 `requests.shutdown(callback)`：

1. 禁止新 Client 和请求。
2. 移除所有 easy handle，完成取消回调。
3. 停止 curl timer。
4. 对所有 `uv_poll_t` 调用 `uv_close()`。
5. 等全部 close callback 后再 `curl_multi_cleanup()`。
6. 关闭 timer handle；所有 handles 关闭后调用 shutdown callback。
7. 仅在整个进程不再使用 libcurl 时执行 `curl_global_cleanup()`。

## 12. 测试与验收

单元测试：URL encoding、参数冲突、headers、Response/err 映射、callback exactly-once、关闭状态。

集成测试使用本地 HTTP/HTTPS server，不依赖公网，覆盖：GET、POST JSON/form、重定向、chunked、压缩、DNS/连接/总超时、TLS 失败、取消、Cookie 隔离、100 并发请求、Client/Engine 关闭。

使用 ASan、Valgrind 或 Dr. Memory 检查 double-free、悬挂 Lua registry ref 和未关闭 uv handle。

首版验收标准：

- `luv.run()` 驱动下 100 个并发 HTTP 请求不阻塞事件循环。
- 成功、HTTP 4xx/5xx、传输失败、超时、取消、关闭各路径 callback 都恰好调用一次。
- 同 Client Cookie 可持久化，不同 Client 不泄露 Cookie/Authorization。
- `client:close()` 后不残留 easy handle、uv poll handle、timer 或 callback ref。
- 所有公开异步接口均为 callback，没有 coroutine/await 依赖。

## 13. 实现里程碑

### Phase 1：闭环

- Engine、`CURLM`、socket/timer 对接。
- `request`、GET、POST、raw body、JSON、headers、timeout。
- 非流式 Response、传输错误、取消与完整清理。

### Phase 2：Session

- Client defaults、base URL、Cookie jar。
- PUT/PATCH/DELETE/HEAD/OPTIONS。
- Client 并发上限、排队、代理、CA 文件。

### Phase 3：生产能力

- 流式下载、multipart 上传、可观测 hooks。
- 可选 pause/resume 背压。
- 独立重试模块：只默认覆盖幂等方法与可重放 body。

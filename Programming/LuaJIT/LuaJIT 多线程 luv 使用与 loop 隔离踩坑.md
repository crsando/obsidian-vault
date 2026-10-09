## 背景

场景：用 LuaJIT C API 配合 POSIX `pthread` 起很多个线程，**每个线程里建一个独立的 `lua_State`**（相当于每个线程跑一个独立 Lua VM），然后每个 VM 各自 `require "luv"`，在各自的 `luv.run()` 里跑事件循环。

直觉上会担心：`luv.so` 在进程里只加载一份，C 静态全局共享，多个 VM 同时 require 是不是会互相踩？结论是**不会**——这个模式是安全且推荐的。真正危险的是几个别的地方。

环境：macOS，LuaJIT 2.1，luv 1.52.1（Homebrew / luarocks 装的）。

---

## 一、为什么"各自 require"不冲突

### 1. `luv.so` 里没有可变静态全局

翻 luv 源码（`src/luv.c`），C 层面的 static 只有两类：

- `static const luaL_Reg luv_*_methods[]`——纯方法表，只读
- `static const char* luv_ctx_key = "luv_context"`——只读字符串指针

**没有一个可变的 static 全局**。所以"多线程同时 require 会踩 C 全局变量"这个担忧，在 luv 里不成立。

### 2. 每 VM 的状态存在各自的 Lua registry 里

关键函数：

```c
LUALIB_API luv_ctx_t* luv_context(lua_State* L) {
  luv_ctx_t* ctx;
  lua_pushstring(L, luv_ctx_key);
  lua_rawget(L, LUA_REGISTRYINDEX);   // 每个 lua_State 自己的 registry
  if (lua_isnil(L, -1)) {
    // 不存在则新建
    lua_pushstring(L, luv_ctx_key);
    ctx = (luv_ctx_t*)lua_newuserdata(L, sizeof(*ctx));
    memset(ctx, 0, sizeof(*ctx));
    lua_rawset(L, LUA_REGISTRYINDEX);
    // 内部 handle 表也挂在 registry
    lua_newtable(L);
    lua_setfield(L, LUA_REGISTRYINDEX, luv_handle_key);
  } else {
    ctx = (luv_ctx_t*)lua_touserdata(L, -1);
  }
  lua_pop(L, 1);
  return ctx;
}
```

`LUA_REGISTRYINDEX` 是**每个 `lua_State` 私有**的。所以每个 VM 拿到自己的 `luv_ctx_t`，里面装着**自己的 `uv_loop_t`**。

而 LuaJIT 的 `lua_State` 之间本来就是完全隔离的：`package.loaded`、GC、栈、registry 全都独立。没有 CPython GIL 那种问题。

### 3. luv 不会碰 `uv_default_loop()`

源码里 grep `uv_default_loop` 命中 **0 次**，`uv_library_shutdown` 也是 **0 次**。

`luaopen_luv` 的初始化逻辑：

```c
// loop is NULL, luv need to create an inner loop
if (ctx->loop == NULL) {
  // Setup the uv_loop meta table for a proper __gc
  luaL_newmetatable(L, "uv_loop.meta");
  lua_pushstring(L, "__gc");
  lua_pushcfunction(L, loop_gc);
  lua_settable(L, -3);
  lua_pop(L, 1);

  lua_pushstring(L, "_loop");
  loop = (uv_loop_t*)lua_newuserdata(L, sizeof(*loop));  // 这个 VM 专属的 loop
  luaL_getmetatable(L, "uv_loop.meta");
  lua_setmetatable(L, -2);
  lua_rawset(L, -3);   // 存进返回的 luv 表里的 _loop 键

  ctx->loop = loop;
  ctx->L = ctxL;
  ctx->mode = -1;
  ret = uv_loop_init(loop);
  ...
}
```

每个 state 第一次 require 时新建一个 `uv_loop_t` userdata，挂在 `ctx->loop`，同时也放进返回的 luv 表的 `_loop` 字段。**不需要手动 new loop**。

> 补充：`luv.new_loop` 这个函数**不存在**。第一次写测试就被它坑了，报错 `attempt to call field 'new_loop' (a nil value)`。每个 state 的 loop 直接就是它自己的。

---

## 二、实测结果

### 通过：16 / 32 线程各自独立 loop

每个线程 `luaL_newstate()` + `luaL_openlibs()` + `require('luv')`，跑一个 1ms 重复 timer 累计 200 tick（`tick==200` 用 assert 校验），再 `luv.run()`：

```
stress threads=16
main: all joined, exit 0
exit=0
```

```
stress threads=32
main: all joined, exit 0
exit=0
```

8 / 16 / 32 线程反复跑都干净退出，tick 计数正确。

### 通过：多线程并发首次 require

让所有线程 busy-wait 到同一时刻再一起 `require('luv')`（模拟"同时首次加载"），高并发下没有加载竞态。

### 通过：多 loop 同时用 libuv 线程池

每个线程在自己的 loop 上用 `luv.new_work`（底层 `uv_queue_work`，走 libuv 进程级线程池），没有崩。但注意下面的第 3 点。

---

## 三、真正会炸的几种

### 1. 多个线程驱动同一个 loop —— 实测直接死锁

`uv_run` **不是线程安全的**。两个线程同时 `uv_run()` 同一个 `uv_loop_t`，会互相抢 epoll 触发和内部队列。

构造场景：两个线程、两个 `lua_State`，但都用 `luv_set_loop()` 把 `ctx->loop` 指向同一个 `uv_loop_t`，然后同时 `luv.run()`：

```
shared loop buffer 0x101079a50, threads=2
(note: ...)
>>> STILL HUNG after 12s -> deadlock confirmed
```

进程**直接吊死**，12 秒 watchdog 都拉不回来，只能 `kill -9`。

**一个 loop 只能由一个线程驱动。**

### 2. `lua_close` 会触发 `loop_gc`，强制 close 该 loop 上所有 handle

```c
static int loop_gc(lua_State *L) {
  luv_ctx_t *ctx = luv_context(L);
  uv_loop_t* loop = ctx->loop;
  if (loop==NULL) return 0;
  // Call uv_close on every active handle
  uv_walk(loop, walk_cb, NULL);
  // Run the event loop until all handles are successfully closed
  while (uv_loop_close(loop)) {
    uv_run(loop, UV_RUN_DEFAULT);
  }
  ctx->loop = NULL;
  return 0;
}
```

关一个 VM 会把**它自己 loop 上所有 handle 强制 close 掉**。所以绝对不能跨线程共享 handle 或 loop userdata——否则你 close 这个 VM，另一个线程正在用的 handle 就被 close 了，是 use-after-free。

源码注释也说明了 `ctx->loop = NULL` 的用意：允许同一个 state 反复 require（把 `package.loaded['luv']` 置 nil 再 require 一次），ctx 生命周期跟 state 一样长。

### 3. libuv 的进程级设施是共享的

loop 虽然独立，但底下这几样是**进程级**的：

| 设施 | 说明 |
|---|---|
| threadpool | `uv_queue_work` / `fs` / `getaddrinfo` 都走它，默认 4 个线程 |
| signal 处理 | `uv_signal_t` 挂进程级信号设施 |
| DNS resolver | 进程级 |

多 loop 同时抢这些不会崩，但会比单线程卡；**别假设一个 loop 停了另一个也干净**。

---

## 四、结论 / 铁律

要用的模式：

```lua
-- 每个 pthread 里，各建一个 lua_State，然后：
local luv = require('luv')
local t = luv.new_timer()
t:start(0, 1, function() ... end)
luv.run()   -- 不带参数 = 用这个 state 自己的 loop
```

- ✅ 每个 pthread 一个 `lua_State`，各自 `require "luv"`，各自 `luv.run()`——**安全，推荐**
- ❌ **一个 loop 只能由一个线程驱动**，绝不两个线程跑同一个 loop
- ❌ **handle 和 loop userdata 绝不跨线程传**
- ⚠️ 跨线程通信走 `uv_async_send`，或者自己的 MPSC 队列（见 [[MPSC Mailbox 方案对比（Spinlock vs Mutex vs 无锁队列）]]）
- ⚠️ libuv 的 threadpool / signal / DNS 是进程级，多 loop 共享

相关笔记：[[LuaJIT 回调式 HTTP 客户端设计]] · [[LuaJIT期货K线内存库设计与踩坑总结]] · [[luajit-messagepack-serialization-research]]

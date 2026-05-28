# Swoole 源码学习计划

面向 **C++ 熟练、PHP extension 略懂** 的开发者，以 **读/改 swoole-src 为主**，顺带掌握 Swoole 用法。

- **时间投入**：每天 1 小时
- **总周期**：12 周（约 84 小时，D1–D84 连续编号，不含第 0 天）
- **环境**：macOS 优先（reactor 看 `kqueue.cc`；Linux 将 epoll 文件替换即可）
- **PHP 版本**：见 [docs/SUPPORTED.md](SUPPORTED.md)（master 分支支持 PHP 8.1–8.4）

## 初学者：请先领读，再看日程表

若觉得本计划「只有文件名、太抽象」，**不要从下面 Day 1 表格开始**。

1. 打开 **[docs/LEARNING-GUIDED.md](LEARNING-GUIDED.md)**，按 **领读 Day 1 → 2 → 3** 跟做（约 3 小时，有逐行解释、自检题、调用链）。
2. 领读完成后再回到本文 **D4**，继续第 1 周剩余内容。
3. 第 2 周起仍用下文日程表；遇阻可回头查领读里的调用链图。

| 阶段 | 文档 | 内容 |
|------|------|------|
| 先修 | [LEARNING-GUIDED.md](LEARNING-GUIDED.md) | 领读 Day 1–3：跑示例、两层地图、`go()` 六步链 |
| 主线 | 本文 D4–D84 | 按周推进的阅读清单与检查点 |

### 日程编号说明

| 范围 | 含义 |
|------|------|
| 第 0 天 | 一次性环境搭建 |
| D1–D3 | 第 1 周前段：跟 [领读 Day 1–3](LEARNING-GUIDED.md) |
| D4–D7 | 第 1 周后段：Scheduler / 状态机 / Channel / context |
| D8–D14 | 第 2 周：Reactor + 协程 Socket |
| D15–D21 | 第 3 周：Runtime Hook |
| D22–D28 | 第 4 周：Server 进程模型 |
| D29–D35 | 第 5 周：HTTP/1.1 |
| D36–D70 | 第 6–10 周：专题深化 |
| D71–D77 | 第 11 周：C++ 动手 |
| D78–D84 | 第 12 周：ext 动手 + 收尾 |

> 锚点优先用**函数名**；行号仅供 IDE 跳转，合并后可能偏移 ±数行。

## 每天固定节奏（60 分钟）

| 时段 | 时长 | 内容 |
|------|------|------|
| 跑 | 15 min | 1 个 phpt 或 1 个 example |
| 读 | 35 min | 当天列出的文件/函数，不泛读 |
| 记 | 10 min | 笔记模板（见下） |

### 笔记模板

```
① 今天学的入口函数：
② 调用链（箭头串起来）：
③ 还没懂的 1 个点：
④ 明天优先追什么：
```

### 学习方法（每个模块固定四步）

```
① 跑 examples/ 或 tests/*.phpt
② 读 ext-src/swoole_*.cc
③ 读 src/ + include/
④ 跑 core-tests 或 phpt 验证
```

---

## 第 0 天（一次性，约 1 小时）

```bash
cd /path/to/swoole-src

# PHP 扩展
phpize && ./configure --enable-swoole-dev --enable-debug-log --enable-sockets --enable-mysqlnd --enable-swoole-curl
make -j$(sysctl -n hw.ncpu) && make install

# C++ 核心
mkdir -p build && cd build && cmake .. && make -j$(sysctl -n hw.ncpu)
./core-tests --gtest_filter="base.*"
```

验证：

```bash
php --ri swoole | head -5
# 快速协程 smoke test（不依赖 tests/bootstrap）
php examples/coroutine/scheduler.php
php -r 'Co\run(function () { go(function () { Co::sleep(0.001); }); }); echo "OK\n";'
```

完整 PHP 测试套件见 [docs/TESTS.md](TESTS.md)（`./scripts/route.sh`）。本仓库根目录**无** `run-tests.php`。

---

## 架构 mental model

```
┌─────────────────────────────────────────┐
│  ext-src/          PHP 扩展层           │
│  php_swoole.cc     扩展入口              │
│  swoole_*.cc       各 PHP 类实现         │
├─────────────────────────────────────────┤
│  src/ + include/   C++ 核心 (libswoole) │
│  reactor/ coroutine/ server/ protocol/  │
└─────────────────────────────────────────┘
```

**前置知识**：

- 必会：C++、socket、epoll/kqueue 概念、协程 vs 线程
- ext-src 需要：PHP 扩展 API（`zval`、`PHP_METHOD`、`zend_call_function`），**不需要**通读 PHP 解释器源码

---

# 第 1 周：协程引擎（D1–D7）

## Day 1–3 — 请跟领读（不在此重复）

**全部内容在 [docs/LEARNING-GUIDED.md](LEARNING-GUIDED.md)**：

| 领读日 | 主题 | 完成标准 |
|--------|------|----------|
| 领读 Day 1 | 跑 `scheduler.php` / 嵌套 `go`；认两层目录；跟读 3 段 C 代码 | 能画「PHP → ext → src → reactor」图；通过 Day 1 自检 5 题 |
| 领读 Day 2 | `php_swoole.cc` 模块加载、`go` 别名、类注册顺序 | 能说出 `MINIT` 与 `go` 的关系；通过 Day 2 自检 |
| 领读 Day 3 | `go()` 六步调用链；裸 `go` vs `Co\Run` | 能背诵六步链；通过 Day 3 自检 |

> 下文 **D4** 起假设已完成领读 Day 1–3。若跳过领读直接看 D4，容易再次觉得抽象。

---

## Day 4 — Scheduler 与事件循环

| 时间 | 文件 | 锚点 |
|------|------|------|
| 0:00–0:15 | `tests/swoole_coroutine/scheduler.phpt`；`examples/coroutine/scheduler.php` | |
| 0:15–0:30 | `ext-src/swoole_coroutine_scheduler.cc` | `PHP_METHOD(..., add)`；`PHP_METHOD(..., start)` → `php_swoole_event_wait` |
| 0:30–0:40 | `src/coroutine/base.cc` | `swoole_coroutine_create`（无 loop 时自动 init）；`coroutine::run` |
| 0:40–0:45 | `src/coroutine/base.cc` | `Coroutine::yield`；`Coroutine::resume`（只看 swap_in/out） |
| 0:45–0:50 | `ext-src/swoole_event.cc` | `php_swoole_reactor_init` / `php_swoole_event_wait` 与 Scheduler 的关系 |
| 0:50–1:00 | 笔记 | `Co\run` vs 裸 `go()` 在 event loop 上的区别 |

**完成标准**：知道 Scheduler.start 与裸 go 自动 init event 两条路径。

---

## Day 5 — 协程状态机

| 时间 | 文件 | 锚点 |
|------|------|------|
| 0:00–0:10 | 复盘 Day 3；重跑 `cid.phpt` | |
| 0:10–0:25 | `src/coroutine/base.cc` | 静态成员 `current` / `coroutines`；`Coroutine::print_list` 四态 INIT/WAITING/RUNNING/END |
| 0:25–0:40 | `src/coroutine/base.cc` | `check_end`；`close`；`activate` / `deactivate` |
| 0:40–0:50 | `ext-src/swoole_coroutine.cc` | `PHPCoroutine::create_context`（独立 VM stack） |
| 0:50–1:00 | 笔记 | 为何每个协程需要独立 PHP VM stack |

**完成标准**：理解 cid、`coroutines` map、四态。

---

## Day 6 — Channel

| 时间 | 文件 | 锚点 |
|------|------|------|
| 0:00–0:15 | `tests/swoole_channel_coro/basic.phpt` | |
| 0:15–0:32 | `src/coroutine/channel.cc` | `Channel::pop`（yield CONSUMER）；`Channel::push`（yield + resume） |
| 0:32–0:45 | `channel.cc` 开头 + `include/swoole_coroutine_channel.h` | producer/consumer 队列 |
| 0:45–0:50 | `ext-src/swoole_channel_coro.cc` | 浏览 push/pop 的 `PHP_METHOD` |
| 0:50–1:00 | 笔记 | pop 空 → yield → push → resume 流程图 |

**完成标准**：能解释 Channel 是协程同步原语。

---

## Day 7 — 栈切换 + 第 1 周复盘

| 时间 | 文件 | 锚点 |
|------|------|------|
| 0:00–0:10 | 重跑 `nested1.phpt`、`basic.phpt` | |
| 0:10–0:25 | `src/coroutine/context.cc` | `Context::Context`；`swap_in`；`swap_out` |
| 0:25–0:32 | `src/coroutine/context.cc` | `context_func` |
| 0:32–0:38 | `thirdparty/boost/asm/`（浏览） | 与 `context.cc` 中 fcontext 切换对照（按 CPU 架构选子目录） |
| 0:38–0:50 | 合并本周调用链笔记 | |
| 0:50–1:00 | 自测 5 题（见下） | |

**自测**：

1. `go()` 的 C 入口函数名？
2. PHP 闭包在哪个函数里执行？（`PHPCoroutine::main_func`）
3. `Coroutine::create` 定义在哪个头文件？
4. Channel 空时 pop 做什么？
5. `Co\run` 在 C 层最终调什么等待？

**第 1 周检查点** ✓

- [ ] 口述 `go()` → `Coroutine::create` 链路
- [ ] 解释 Channel push/pop 与调度
- [ ] 跑通 ≥5 个 phpt

---

# 第 2 周：Reactor + 协程 Socket（D8–D14）

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D8 | `./core-tests --gtest_filter="coroutine_base.*"` | `include/swoole_reactor.h` |
| D9 | `tests/swoole_socket_coro/` 任 1 个 | `src/reactor/base.cc` |
| D10 | `examples/coroutine/tcp_echo.php` | `src/reactor/kqueue.cc`（Linux: `epoll.cc`） |
| D11 | socket phpt 再 1 个 | `src/coroutine/socket.cc` |
| D12 | `examples/timer/tick.php` | `src/core/timer.cc` |
| D13 | 对照 timer test | `ext-src/swoole_timer.cc`；`ext-src/php_swoole_cxx.h` |
| D14 | 复盘 | `PHPCoroutine::main_func` 里 `zend_call_function` |

**检查点**：协程 read 阻塞时，线程在 wait kqueue。

---

# 第 3 周：Runtime Hook（D15–D21）

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D15 | `examples/runtime/stream.php` | `ext-src/swoole_runtime.cc` |
| D16 | `tests/swoole_runtime/` ×2 | `src/coroutine/hook.cc` |
| D17 | `examples/runtime/curl.php`（若启用） | `include/swoole_socket_hook.h` |
| D18 | file/sleep phpt | `include/swoole_file_hook.h` |
| D19 | runtime phpt ×2 | `hook.cc` 跟一条具体 hook |
| D20 | `tests/swoole_coroutine_lock/` 任 1 个 | `ext-src/swoole_coroutine_lock.cc`；`src/lock/` |
| D21 | 复盘 | 整理 hook 列表；Hook 何时生效 |

**检查点**：能说明 `SWOOLE_HOOK_ALL` 替换哪一层。

---

# 第 4 周：Server 进程模型（D22–D28）

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D22 | `examples/server/echo.php` | `src/server/master.cc` → `Server::start()` |
| D23 | `tests/swoole_server/start_twice.phpt` | `manager.cc` + `worker.cc` |
| D24 | — | `src/server/base.cc`（BASE 模式） |
| D25 | — | `ext-src/swoole_server.cc` |
| D26 | — | `src/server/port.cc` |
| D27 | `examples/task/task.php` | `src/server/task_worker.cc` |
| D28 | 复盘 | Master → Manager → Worker 时序 |

**检查点**：区分 `SWOOLE_PROCESS` 与 `SWOOLE_BASE`。

---

# 第 5 周：HTTP/1.1 全链路（D29–D35）

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D29 | `examples/coroutine/http_server.php` | `ext-src/swoole_http_server.cc` |
| D30 | `tests/swoole_http_server/` 任 1 个 | `src/protocol/http.cc` |
| D31 | http phpt ×1 | `ext-src/swoole_http_request.cc` |
| D32 | 改 example 试跑 | `ext-src/swoole_http_response.cc` |
| D33 | phpt ×2 | `src/server/worker.cc` |
| D34 | `ext-src/swoole_http_server_coro.cc`（浏览） | 协程风格 HTTP Server 与经典 Server 差异 |
| D35 | 复盘 | accept → parse → callback → end 调用栈 |

**检查点**：HTTP 调用栈笔记 ≥10 步。

---

# 第 6 周：WebSocket（D36–D42）

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D36 | `examples/websocket/server.php` | `ext-src/swoole_websocket_server.cc` 类注册与回调 |
| D37 | `examples/websocket/client.php` | `src/protocol/websocket.cc` 帧解析 |
| D38 | `tests/swoole_websocket_server/` 任 1 个 | handshake 与 upgrade 路径 |
| D39 | phpt ×1 | `ext-src/swoole_http_server.cc` 中 WebSocket 与 HTTP 共用部分 |
| D40 | 改 example 试跑 | push / ping / close 在 ext 层的实现 |
| D41 | 复盘 | 画出 HTTP upgrade → WS 帧处理调用栈 |
| D42 | 笔记 | 与第 5 周 HTTP 笔记合并对比 |

**检查点**：能说明 WebSocket 与 `Swoole\Http\Server` 的继承关系。

---

# 第 7 周：Table + Process（D43–D49）

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D43 | `examples/table/set.php` | `src/memory/table.cc` |
| D44 | `examples/table/server.php` | `ext-src/swoole_table.cc` |
| D45 | `tests/swoole_table/` 任 1 个 | 共享内存行锁与迭代 |
| D46 | `examples/process/worker.php` | `ext-src/swoole_process.cc`（单进程封装） |
| D47 | `examples/process_pool/send.php` 或 `examples/task/task.php` | `src/os/process_pool.cc`；`ext-src/swoole_process_pool.cc` |
| D48 | phpt ×1 | Process 与 ProcessPool 使用场景对比 |
| D49 | 复盘 | Table 跨进程共享 vs Channel 协程内通信 |

**检查点**：区分 `Swoole\Process` 与 `Swoole\Process\Pool`。

---

# 第 8 周：HTTP/2（D50–D56）

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D50 | `examples/http2/server.php`（若有）或 `tests/swoole_http2_server/` 任 1 个 | `src/protocol/http2.cc` |
| D51 | phpt ×1 | `ext-src/swoole_http2_server.cc` |
| D52 | `examples/coroutine/http2_client.php` | `ext-src/swoole_http2_client_coro.cc` |
| D53 | phpt ×1 | HTTP/1 与 HTTP/2 在 Server 层的分支 |
| D54 | `thirdparty/nghttp2`（浏览目录） | 与 Swoole 封装边界 |
| D55 | 复盘 | HTTP/2 多路复用与 Worker 模型 |
| D56 | 笔记 | 记录与第 5 周 HTTP/1 的差异点 |

**检查点**：知道 HTTP/2 入口文件与依赖的 thirdparty。

---

# 第 9 周：协程 Client + 网络（D57–D63）

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D57 | `examples/coroutine/http_client.php` | `ext-src/swoole_http_client_coro.cc` |
| D58 | `examples/coroutine/redis_pool.php`（可选 Redis） | `src/protocol/redis.cc` |
| D59 | `tests/swoole_client_coro/` 任 1 个 | `ext-src/swoole_client_coro.cc` |
| D60 | `tests/swoole_coroutine/dnslookup_1.phpt` | `src/network/dns.cc`；`ext-src/swoole_name_resolver.cc` |
| D61 | `src/network/socket.cc`（浏览） | 与 `src/coroutine/socket.cc` 分层关系 |
| D62 | Hook 回顾 | `examples/runtime/curl.php` + `ext-src/swoole_curl.cc` |
| D63 | 复盘 | Client 侧调用栈（connect → send → recv） |

**检查点**：能对比 Server 与 Client 在 ext-src 中的对称文件。

---

# 第 10 周：SSL + 深化复盘（D64–D70）

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D64 | `examples/ssl/server.php`（configure 需 OpenSSL） | `src/protocol/ssl.cc` |
| D65 | `tests/swoole_ssl/` 任 1 个 | SSL 握手在 Server/Client 中的挂接点 |
| D66 | `src/lock/` + `ext-src/swoole_lock.cc` | 进程锁 vs `swoole_coroutine_lock` |
| D67 | **Linux 可选** / macOS 跳过 | `src/coroutine/iouring.cc`（`--enable-iouring`） |
| D68 | **选修** | `docs/windows-native-support.md`；`ext-src/swoole_windows.cc` |
| D69 | **选修** | `ext-src/swoole_thread.cc` + `examples/thread/`（需 ZTS + `--enable-swoole-thread`） |
| D70 | 总复盘 D1–D69 | 更新「源码地图」草稿 |

**检查点**：列出本机可编译特性（OpenSSL / io_uring / thread）与跳过项。

---

# 第 11 周：C++ 动手（D71–D77）

| 天 | 内容 |
|----|------|
| D71 | `src/coroutine/base.cc` 加 trace/log |
| D72 | `make && ./core-tests --gtest_filter="coroutine_base.*"` |
| D73 | 读 `core-tests/src/coroutine/base.cpp` |
| D74 | 跑通修改 |
| D75 | 读 `docs/CODE-STYLE.md` |
| D76 | 整理 patch |
| D77 | 复盘 |

---

# 第 12 周：ext 动手 + 收尾（D78–D84）

| 天 | 内容 |
|----|------|
| D78 | 小 ext 改动或修 phpt |
| D79 | `make install` + phpt |
| D80 | `docs/TESTS.md`、`docs/ISSUE.md` |
| D81 | 写 1 页「源码地图」（可结合下文附录） |
| D82 | 浏览 GitHub issues |
| D83 | 总复习关键 test |
| D84 | 选定后续贡献方向 |

**12 周交付** ✓

- [ ] 1 个 core-tests 改动
- [ ] 1 个 phpt 验证过的改动
- [ ] HTTP 全链路笔记 + 架构地图

---

## 压缩版（8 周保底）

| 周 | 对应日 | 内容 |
|----|--------|------|
| 1–2 | 领读 Day 1–3 + D4–D14 | 协程 + Reactor |
| 3–4 | D15–D28 | Hook + Server |
| 5–6 | D29–D42 | HTTP/1 + WebSocket |
| 7 | D50–D56 或 D57–D63 | HTTP/2 **或** Client（二选一） |
| 8 | D71–D84 | 动手 patch + 收尾 |

> 8 周路径跳过 D43–D49（Table/Process）与 D64–D70（SSL 等），可在贡献前补读。

---

## 扩展补课（碎片 15 min × 3）

1. `zend_parse_parameters` → 对照 `ext-src/swoole_timer.cc`
2. `zend_call_function` → 对照 `PHPCoroutine::create`
3. `ext-src/php_swoole_cxx.h` + `php_swoole_cxx.cc` → 对象封装

参考：<https://www.php.net/manual/en/internals2.php>

---

## 进度原则

- 35 min 读不完 → 顺延，不叠加
- Day 3 没懂 → 重复 Day 3，不进入第 2 周
- 改 C++ 用 `core-tests`；改 ext 用 phpt
- gtest 过滤器区分大小写：常用 `base.*`、`coroutine_base.*`、`coroutine_*`

---

## 参考资源

- 官方文档：<https://wiki.swoole.com/>
- API 补全：<https://github.com/swoole/ide-helper>
- **初学者领读**：本仓库 `docs/LEARNING-GUIDED.md`
- C++ API 说明：本仓库 `docs/API.md`
- 架构说明：本仓库 `CLAUDE.md`
- PHP 版本支持：本仓库 `docs/SUPPORTED.md`

---

## 附录 A：子系统与学习计划对照

| 子系统 | 核心路径 | 计划覆盖 |
|--------|----------|----------|
| 协程引擎 | `src/coroutine/`、`ext-src/swoole_coroutine*.cc` | 领读 Day 1–3 + D4–D7 |
| Reactor / Timer | `src/reactor/`、`src/core/timer.cc` | D8–D14 |
| Runtime Hook | `src/coroutine/hook.cc`、`ext-src/swoole_runtime.cc` | D15–D21 |
| 协程锁 | `src/lock/`、`ext-src/swoole_coroutine_lock.cc` | D20 |
| Server 进程模型 | `src/server/`、`ext-src/swoole_server.cc` | D22–D28 |
| HTTP/1.1 | `src/protocol/http.cc`、`ext-src/swoole_http_*.cc` | D29–D35 |
| WebSocket | `src/protocol/websocket.cc`、`ext-src/swoole_websocket_server.cc` | D36–D42 |
| Table / 共享内存 | `src/memory/table.cc`、`ext-src/swoole_table.cc` | D43–D45 |
| Process / Pool | `src/os/process_pool.cc`、`ext-src/swoole_process*.cc` | D46–D49 |
| HTTP/2 | `src/protocol/http2.cc`、`ext-src/swoole_http2_*.cc` | D50–D56 |
| 协程 Client / DNS | `src/network/`、`ext-src/swoole_*_client_coro.cc` | D57–D63 |
| SSL/TLS | `src/protocol/ssl.cc`、`examples/ssl/` | D64–D65 |
| io_uring | `src/coroutine/iouring.cc` | D67（Linux） |
| Thread 模式 | `ext-src/swoole_thread*.cc` | D69（选修） |
| Windows / IOCP | `ext-src/swoole_windows.cc`、`docs/windows-*.md` | D68（选修） |
| thirdparty | `thirdparty/boost/asm/`、`nghttp2/`、`hiredis/` 等 | D7、D54 |

---

## 附录 B：`ext-src/swoole_*.cc` 索引

| 文件 | 主要职责 | 建议阅读日 |
|------|----------|------------|
| `php_swoole.cc` | 扩展入口、MINIT、函数表 | 领读 Day 2 |
| `swoole_coroutine.cc` | `go()`、PHPCoroutine | 领读 Day 1、3；D5 |
| `swoole_coroutine_scheduler.cc` | `Co\run` / Scheduler | 领读 Day 1；D4 |
| `swoole_coroutine_system.cc` | 协程版系统 API（sleep、文件等） | D15+ |
| `swoole_coroutine_lock.cc` | 协程锁 | D20 |
| `swoole_channel_coro.cc` | Channel | D6 |
| `swoole_event.cc` | Event 循环 PHP 封装 | D4 |
| `swoole_runtime.cc` | Hook 开关 | D15 |
| `swoole_timer.cc` | 定时器 | D12–D13 |
| `swoole_socket_coro.cc` | 协程 Socket | D9–D11 |
| `swoole_server.cc` | Server 主类 | D25 |
| `swoole_server_port.cc` | 多端口 | D26 |
| `swoole_http_server.cc` | HTTP Server | D29–D31 |
| `swoole_http_server_coro.cc` | 协程 HTTP Server | D34 |
| `swoole_http_request.cc` / `swoole_http_response.cc` | 请求/响应对象 | D31–D32 |
| `swoole_websocket_server.cc` | WebSocket | D36–D40 |
| `swoole_http2_server.cc` / `swoole_http2_client_coro.cc` | HTTP/2 | D50–D52 |
| `swoole_http_client_coro.cc` | 协程 HTTP 客户端 | D57 |
| `swoole_client_coro.cc` / `swoole_client.cc` | TCP 客户端 | D59 |
| `swoole_curl.cc` | 协程 curl | D62 |
| `swoole_table.cc` | 共享内存表 | D44 |
| `swoole_process.cc` / `swoole_process_pool.cc` | 进程与进程池 | D46–D47 |
| `swoole_lock.cc` | 锁 | D66 |
| `swoole_atomic.cc` | 原子计数 | D43+ |
| `swoole_name_resolver.cc` | 自定义 DNS | D60 |
| `swoole_redis_server.cc` | Redis 协议 Server | 选修 |
| `swoole_admin_server.cc` | 管理端口 | 选修 |
| `swoole_tracer.cc` | 调试追踪 | 选修 |
| `swoole_thread*.cc` | 线程模式 | D69 |
| `swoole_windows.cc` | Windows 适配 | D68 |
| `swoole_stdext.cc` | Std 扩展 | 选修 |
| DB 驱动（`swoole_pgsql.cc` 等） | 可选编译 | 按 configure 选修 |

---

## 附录 C：`src/` 子目录索引

| 目录 | 职责 | 建议阅读日 |
|------|------|------------|
| `core/` | 定时器、字符串、日志、Channel 底层 | D12–D13 |
| `reactor/` | epoll/kqueue/poll 事件循环 | D8–D10 |
| `coroutine/` | 协程、Hook、Socket、Channel、io_uring | D1–D21 |
| `server/` | Master/Manager/Worker、端口、Task | D22–D28、D33 |
| `protocol/` | HTTP、HTTP/2、WebSocket、SSL、Redis | D29–D40、D50–D65 |
| `network/` | Socket、DNS、地址 | D57–D61 |
| `memory/` | Table、RingBuffer、LRU | D43–D45 |
| `os/` | 信号、管道、进程池、消息队列 | D46–D47 |
| `lock/` | 互斥、读写锁、协程锁底层 | D20、D66 |
| `wrapper/` | PHP stream hook 包装 | D15+ |

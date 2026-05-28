# Swoole 源码学习计划

面向 **C++ 熟练、PHP extension 略懂** 的开发者，以 **读/改 swoole-src 为主**，顺带掌握 Swoole 用法。

- **时间投入**：每天 1 小时
- **总周期**：12 周（约 84 小时）
- **环境**：macOS 优先（reactor 看 `kqueue.cc`；Linux 将 epoll 文件替换即可）

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
./core-tests --gtest_filter="Base.*"
```

验证：

```bash
php --ri swoole | head -5
php run-tests.php tests/swoole_coroutine/array_walk.phpt
```

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

# 第 1 周：协程引擎（分钟级清单）

> 路径基准：仓库根目录。行号以 master 分支为参考，合并后可能略有偏移。

## Day 1 — 项目地图

| 时间 | 任务 |
|------|------|
| 0:00–0:15 | `php run-tests.php tests/swoole_coroutine/array_walk.phpt` |
| 0:15–0:25 | 读 `CLAUDE.md` Two-Layer + Key Subsystems |
| 0:25–0:40 | 读 `docs/API.md` 前半（Basic / Coroutine / Core header） |
| 0:40–0:50 | IDE 浏览 `include/`、`src/coroutine/`、`ext-src/swoole_coroutine*.cc` |
| 0:50–1:00 | 笔记：写出 ext-src ↔ src 两层关系 |

**完成标准**：能画两层架构图。

---

## Day 2 — 扩展加载

| 时间 | 文件 | 行号/内容 |
|------|------|-----------|
| 0:00–0:15 | 跑 `tests/swoole_coroutine/cid.phpt` | |
| 0:15–0:22 | `ext-src/php_swoole.cc` | L334 模块入口；L151 `PHP_FE(swoole_coroutine_create)` |
| 0:22–0:35 | `ext-src/php_swoole.cc` | L626 MINIT 开头；L978–987 `go` alias；**跳过**中间常量 |
| 0:35–0:50 | `ext-src/php_swoole.cc` | L989–1023 类注册顺序（coroutine → scheduler → channel） |
| 0:50–1:00 | 笔记 | `php_swoole_coroutine_minit` 注册了哪些模块 |

**完成标准**：知道协程模块在 `PHP_MINIT` 中注册。

---

## Day 3 — `go()` 完整调用链（最重要）

| 时间 | 文件 | 函数/行号 |
|------|------|-----------|
| 0:00–0:15 | 跑 `tests/swoole_coroutine/nested1.phpt` | |
| 0:15–0:25 | `ext-src/swoole_coroutine.cc` | L1106 `PHP_FUNCTION(swoole_coroutine_create)` |
| 0:25–0:38 | `ext-src/swoole_coroutine.cc` | L823 `PHPCoroutine::create`；L756–783 `main_func` + `zend_call_function` |
| 0:38–0:50 | `include/swoole_coroutine.h` L147；`src/coroutine/base.cc` L76–85、L96–104 | `create` → `run` → `swap_in` |
| 0:50–1:00 | 笔记 | 背出 6 步调用链 |

```
go(closure)
→ swoole_coroutine_create()          [ext, L1106]
→ PHPCoroutine::create()             [ext, L823]
→ Coroutine::create(main_func)       [header, L147]
→ Coroutine::run() → swap_in()       [base.cc, L96]
→ main_func → zend_call_function()   [ext, L783]
→ 你的 PHP 闭包
```

**完成标准**：能不看代码口述上述链路。

---

## Day 4 — Scheduler 与事件循环

| 时间 | 文件 | 内容 |
|------|------|------|
| 0:00–0:15 | `tests/swoole_coroutine/scheduler.phpt`；`examples/coroutine/scheduler.php` | |
| 0:15–0:30 | `ext-src/swoole_coroutine_scheduler.cc` | L236 `add`；L276 `start`（含 L305 `php_swoole_event_wait`） |
| 0:30–0:45 | `src/coroutine/base.cc` | L291 `swoole_coroutine_create`（无 loop 时自动 init）；L273 `coroutine::run` |
| 0:45–0:50 | `src/coroutine/base.cc` | L107 `yield`；L158 `resume`（只看 swap_in/out） |
| 0:50–1:00 | 笔记 | `Co\run` vs 裸 `go()` 在 event loop 上的区别 |

**完成标准**：知道 Scheduler.start 与裸 go 自动 init event 两条路径。

---

## Day 5 — 协程状态机

| 时间 | 文件 | 内容 |
|------|------|------|
| 0:00–0:10 | 复盘 Day 3；重跑 `cid.phpt` | |
| 0:10–0:25 | `src/coroutine/base.cc` | L22–27 静态成员；L201–223 四态 INIT/WAITING/RUNNING/END |
| 0:25–0:40 | `src/coroutine/base.cc` | L87 `check_end`；L186 `close`；L42 `activate/deactivate` |
| 0:40–0:50 | `ext-src/swoole_coroutine.cc` | L314–342 `create_context`（独立 VM stack） |
| 0:50–1:00 | 笔记 | 为何每个协程需要独立 PHP VM stack |

**完成标准**：理解 cid、`coroutines` map、四态。

---

## Day 6 — Channel

| 时间 | 文件 | 内容 |
|------|------|------|
| 0:00–0:15 | `tests/swoole_channel_coro/basic.phpt` | |
| 0:15–0:32 | `src/coroutine/channel.cc` | L55 `pop`（yield CONSUMER）；L105 `push`（yield + resume） |
| 0:32–0:45 | `channel.cc` 开头 + `include/swoole_coroutine_channel.h` | producer/consumer 队列 |
| 0:45–0:50 | `ext-src/swoole_channel_coro.cc` | 浏览 push/pop 的 PHP_METHOD |
| 0:50–1:00 | 笔记 | pop 空 → yield → push → resume 流程图 |

**完成标准**：能解释 Channel 是协程同步原语。

---

## Day 7 — 栈切换 + 第 1 周复盘

| 时间 | 文件 | 内容 |
|------|------|------|
| 0:00–0:10 | 重跑 `nested1.phpt`、`basic.phpt` | |
| 0:10–0:28 | `src/coroutine/context.cc` | L36 构造；L120 `swap_in`；L134 `swap_out` |
| 0:28–0:38 | `src/coroutine/context.cc` | L144 `context_func` |
| 0:38–0:50 | 合并本周调用链笔记 | |
| 0:50–1:00 | 自测 5 题（见下） | |

**自测**：

1. `go()` 的 C 入口函数名？
2. PHP 闭包在哪一行执行？（`swoole_coroutine.cc` ~L783）
3. `Coroutine::create` 在 header 哪一行？（~L147）
4. Channel 空时 pop 做什么？
5. `Co\run` 在 C 层最终调什么等待？

**第 1 周检查点** ✓

- [ ] 口述 `go()` → `Coroutine::create` 链路
- [ ] 解释 Channel push/pop 与调度
- [ ] 跑通 ≥5 个 phpt

---

# 第 2 周：Reactor + 协程 Socket

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D8 | `./core-tests --gtest_filter="Coroutine.Base*"` | `include/swoole_reactor.h` |
| D9 | `tests/swoole_socket_coro/` 任 1 个 | `src/reactor/base.cc` |
| D10 | `examples/coroutine/tcp_echo.php` | `src/reactor/kqueue.cc`（Linux: `epoll.cc`） |
| D11 | socket phpt 再 1 个 | `src/coroutine/socket.cc` |
| D12 | `examples/timer/tick.php` | `src/core/timer.cc` |
| D13 | 对照 timer test | `ext-src/swoole_timer.cc`；`php_swoole_cxx.h` |
| D14 | 复盘 | `PHPCoroutine::create` 里 `zend_call_function` |

**检查点**：协程 read 阻塞时，线程在 wait kqueue。

---

# 第 3～4 周：Runtime Hook

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D15 | `examples/runtime/stream.php` | `ext-src/swoole_runtime.cc` |
| D16 | `tests/swoole_runtime/` ×2 | `src/coroutine/hook.cc` |
| D17 | curl example（若启用） | `include/swoole_socket_hook.h` |
| D18 | file/sleep phpt | `include/swoole_file_hook.h` |
| D19 | runtime phpt ×2 | hook.cc 跟一条具体 hook |
| D20–21 | 复盘 | 整理 hook 列表；Hook 何时生效 |

**检查点**：能说明 `SWOOLE_HOOK_ALL` 替换哪一层。

---

# 第 5～6 周：Server 进程模型

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D22 | `examples/server/echo.php` | `src/server/master.cc` → `Server::start()` |
| D23 | `tests/swoole_server/start.phpt` | `manager.cc` + `worker.cc` |
| D24 | — | `src/server/base.cc`（BASE 模式） |
| D25 | — | `ext-src/swoole_server.cc` |
| D26 | — | `src/server/port.cc` |
| D27 | `examples/task/task.php` | `src/server/task_worker.cc` |
| D28 | 复盘 | Master → Manager → Worker 时序 |

**检查点**：区分 `SWOOLE_PROCESS` 与 `SWOOLE_BASE`。

---

# 第 7～8 周：HTTP 全链路

| 天 | 15 min 跑 | 35 min 读 |
|----|-----------|-----------|
| D29 | `examples/coroutine/http_server.php` | `ext-src/swoole_http_server.cc` |
| D30 | http phpt ×1 | `src/protocol/http.cc` |
| D31 | phpt ×1 | `swoole_http_request.cc` |
| D32 | 改 example 试跑 | `swoole_http_response.cc` |
| D33 | phpt ×2 | `src/server/worker.cc` |
| D34–35 | 复盘 | accept → parse → callback → end 调用栈 |

**检查点**：HTTP 调用栈笔记 ≥10 步。

---

# 第 9～10 周：专题（各 7 天 × 1h）

**专题 A — WebSocket**：`examples/websocket/` → `src/protocol/websocket.cc` → `ext-src/swoole_websocket_server.cc`

**专题 B — Table + Process**：`examples/table/` → `src/memory/table.cc` → `src/os/process_pool.cc`

---

# 第 11 周：C++ 动手

| 天 | 内容 |
|----|------|
| D71 | `src/coroutine/base.cc` 加 trace/log |
| D72 | `make && ./core-tests --gtest_filter="Coroutine.*"` |
| D73 | 读 `core-tests/src/coroutine/base.cpp` |
| D74 | 跑通修改 |
| D75 | 读 `docs/CODE-STYLE.md` |
| D76 | 整理 patch |
| D77 | 复盘 |

---

# 第 12 周：ext 动手 + 收尾

| 天 | 内容 |
|----|------|
| D78 | 小 ext 改动或修 phpt |
| D79 | `make install` + phpt |
| D80 | `docs/TESTS.md`、`docs/ISSUE.md` |
| D81 | 写 1 页「源码地图」 |
| D82 | 浏览 GitHub issues |
| D83 | 总复习关键 test |
| D84 | 选定后续贡献方向 |

**12 周交付** ✓

- [ ] 1 个 core-tests 改动
- [ ] 1 个 phpt 验证过的改动
- [ ] HTTP 全链路笔记 + 架构地图

---

## 压缩版（8 周保底）

| 周 | 内容 |
|----|------|
| 1–2 | 协程 + Reactor |
| 3–4 | Hook + Server |
| 5–6 | HTTP |
| 7 | 专题二选一 |
| 8 | 动手 patch |

---

## 扩展补课（碎片 15 min × 3）

1. `zend_parse_parameters` → 对照 `ext-src/swoole_timer.cc`
2. `zend_call_function` → 对照 `PHPCoroutine::create`
3. `php_swoole_cxx.cc` → 对象封装

参考：<https://www.php.net/manual/en/internals2.php>

---

## 进度原则

- 35 min 读不完 → 顺延，不叠加
- Day 3 没懂 → 重复 Day 3，不进入第 2 周
- 改 C++ 用 `core-tests`；改 ext 用 phpt

---

## 参考资源

- 官方文档：<https://wiki.swoole.com/>
- API 补全：<https://github.com/swoole/ide-helper>
- C++ API 说明：本仓库 `docs/API.md`
- 架构说明：本仓库 `CLAUDE.md`

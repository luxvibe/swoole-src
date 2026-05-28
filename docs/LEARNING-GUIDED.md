# Swoole 源码领读（初学者先走这里）

> **用法**：不要先扫 12 周日程表。按本文 **Day 1 → Day 2 → Day 3** 顺序，每天约 1 小时；每节有「你要做什么」「逐行在说什么」「自检」。  
> 三天走完再打开 [LEARNING-PLAN.md](LEARNING-PLAN.md) 从 **D4** 继续。

---

## 读源码前，先建立 3 个直觉

| 直觉 | 一句话 |
|------|--------|
| **协程** | 用户写的多个 `go()` 看起来像并发，实际在**一个线程**里轮流执行 |
| **两层代码** | PHP 能调到的在 `ext-src/`；真正调度、切栈在 `src/` + `include/` |
| **事件循环** | `Co::sleep`、网络读写会「让出 CPU」；底层用 kqueue/epoll **等事件到了再唤醒** |

后面所有文件都围绕这三点展开。

---

# 领读 Day 1：先跑起来，再认地图（约 60 分钟）

## 1. 跑第一个程序（15 分钟）

### 1.1 最小示例

在仓库根目录执行：

```bash
php examples/coroutine/scheduler.php
```

应输出 `hello world`。打开该文件，只有 5 行：

```php
Co\Run(function () {
    Co::sleep(0.2);
    echo "hello world\n";
});
```

**逐行理解（今天必须懂）：**

| 行 | 在干什么 |
|----|----------|
| `Co\Run(...)` | 启动**协程调度器**：内部会初始化事件循环，跑完闭包后退出 |
| `Co::sleep(0.2)` | 当前协程**挂起** 0.2 秒，把执行权交还给调度器（不是阻塞整个进程） |
| `echo ...` | sleep 结束后继续执行，然后整个 `Co\Run` 结束 |

> `Co\Run` 和 `go()` 区别：今天只记——**`Co\Run` 负责「开机+关机」**；`go()` 是在已开机后**再开一条协程**（Day 3 细讲）。

### 1.2 稍复杂一点的测试

不依赖测试框架，直接跑（与 `tests/swoole_coroutine/array_walk.phpt` 逻辑相同）：

```bash
php -r '
Co\run(function () {
    for ($n = 2; $n--;) {
        go(function () {
            $array = range(0, 1);
            array_walk($array, function ($item) {
                Co::sleep([0.01, 0.001][$item]);
            });
        });
    }
});
echo "DONE\n";
'
```

**带着问题读这段 PHP：**

1. `for ($n = 2; $n--)` 会创建 **2 个** `go()` 协程，它们**交替**推进，而不是开 2 个 OS 线程。  
2. 每个协程里 `array_walk` 的闭包遇到 `Co::sleep` 会 yield——所以两个协程可以「你 sleep 我跑、我 sleep 你跑」。  
3. 最后打印 `DONE` 表示**所有协程都结束**，`Co\run` 才返回。

若这里卡住：先确认 `php --ri swoole` 有输出；读源码建议用**本仓库编译安装的扩展**（见 [LEARNING-PLAN.md 第 0 天](LEARNING-PLAN.md)），避免 PHP 5.x 旧扩展与 master 测试脚本不一致。

---

## 2. 仓库两层：用「接电话」类比（10 分钟）

想象用户 PHP 代码是**客户**，C++ 核心是**后台系统**：

```
客户:  go(function() { ... })
         │
         ▼
前台 ext-src/swoole_coroutine.cc   ← 把 PHP 闭包翻译成 C++ 能执行的回调
         │
         ▼
后台 src/coroutine/base.cc        ← 分配 cid、run、yield、resume
         │
         ▼
机房 src/reactor/kqueue.cc         ← 等 timer/socket 事件（Day 2 再细看）
```

**今天只记 4 个路径（在 IDE 里打开文件名即可，不用通读）：**

| 你写的 PHP | 对应 ext-src | 对应 src/ |
|------------|--------------|-----------|
| `Co\Run` | `swoole_coroutine_scheduler.cc` | 事件循环 init/wait |
| `go()` / `go` 函数 | `swoole_coroutine.cc` | `coroutine/base.cc` |
| `Co::sleep` | `swoole_coroutine_system.cc`（先知道存在） | timer + yield |
| `new Channel` | `swoole_channel_coro.cc` | `coroutine/channel.cc` |

打开 `ext-src/swoole_coroutine.cc`，用搜索（Cmd+F）找 `PHP_FUNCTION(swoole_coroutine_create)`——这是 **`go()` 在 C 层的入口**，Day 3 会顺着它往下读。

---

## 3. 跟读 3 段真实代码（25 分钟）

### 3.1 PHP 入口：`go()` 长什么样

文件：`ext-src/swoole_coroutine.cc`，函数 `PHP_FUNCTION(swoole_coroutine_create)`。

核心逻辑可以概括成 3 步（不必懂每一行 Zend API）：

```cpp
// 1. 从 PHP 解析出「要调用的闭包」
ZEND_PARSE_PARAMETERS_START(1, -1)
Z_PARAM_FUNC(fci, fci_cache)
...

// 2. 交给 PHPCoroutine 创建协程
long cid = PHPCoroutine::create(&fci_cache, fci.param_count, fci.params, &fci.function_name);

// 3. 把协程 id 返回给 PHP
RETURN_LONG(cid);
```

**你要记住**：`go($fn)` 在 C 里就是 `swoole_coroutine_create` → `PHPCoroutine::create`。

### 3.2 调度器：`Co\Run` 结束时在等什么

文件：`ext-src/swoole_coroutine_scheduler.cc`，`PHP_METHOD(swoole_coroutine_scheduler, start)`。

关键两行（中间省略了把任务放进队列的循环）：

```cpp
    // ... 对每个任务调用 PHPCoroutine::create ...

    php_swoole_event_wait();   // ← 阻塞在这里，直到所有协程结束、没有待处理事件
```

**你要记住**：`Co\Run` 不是「跑完闭包就退出」，而是 **先跑任务，再 `event_wait` 把整个调度跑干**。

### 3.3 C++ 协程：`run()` 只做一件事

文件：`src/coroutine/base.cc`，`Coroutine::run()`：

```cpp
long Coroutine::run() {
    current = this;           // 标记「当前正在跑的是这个协程」
    state = STATE_RUNNING;
    ctx.swap_in();            // ← 切换到协程栈，开始执行 C++ 回调
    check_end();              // 回调返回后，若结束则 close
    return _cid;
}
```

**你要记住**：`swap_in()` = 切换到协程私有栈；PHP 闭包最终在 `PHPCoroutine::main_func` 里通过 `zend_call_function` 执行（Day 3 领读）。

---

## 4. 动手画一张图（5 分钟）

在笔记里画（或抄写）：

```
PHP: Co\Run → 闭包里的 go() → Co::sleep
          ↓
ext:  scheduler.start → PHPCoroutine::create → (Day3) main_func
          ↓
src:  Coroutine::run → swap_in → ... yield on sleep ...
          ↓
reactor: 定时器到期 → resume 协程
```

---

## 5. Day 1 自检（必须能答）

| 问题 | 参考答案 |
|------|----------|
| `Co\Run` 和 `go()` 分工？ | `Co\Run` 启停调度器+事件循环；`go()` 在循环里新建协程 |
| `go()` 的 C 函数名？ | `swoole_coroutine_create` |
| 协程调度核心在哪个目录？ | `src/coroutine/`（不是 ext-src） |
| `php_swoole_event_wait` 何时调用？ | `Co\Run` / Scheduler 的 `start` 末尾 |
| 今天**不用**搞懂什么？ | Zend 宏、`swap_in` 汇编细节、Channel 实现 |

全部答得上 → Day 1 合格，进入 [领读 Day 2](#领读-day-2扩展是怎么加载进-php-的约-60-分钟)。

---

# 领读 Day 2：扩展是怎么加载进 PHP 的（约 60 分钟）

## 1. 跑：`cid.phpt` 在验证什么（10 分钟）

```bash
php -r '
Co\run(function () {
    $cid = go(function () {
        echo "child cid=", Co::getCid(), "\n";
    });
    echo "parent cid=", Co::getCid(), ", child=", $cid, "\n";
});
'
```

**预期直觉**：主协程和 `go()` 出来的子协程 **cid 不同**；每个协程有独立 id，存在 C++ 的 `coroutines` map 里（Day 5 日程会读）。

---

## 2. PHP 扩展加载流程（15 分钟）

用「插件安装」类比 `php_swoole.cc`：

| 阶段 | C 符号 | 作用 |
|------|--------|------|
| 模块定义 | `zend_module_entry swoole_module_entry` | 告诉 PHP：模块叫 `swoole`，启动时调 `PHP_MINIT(swoole)` |
| 函数表 | `swoole_functions[]` 里的 `PHP_FE(swoole_coroutine_create, ...)` | 注册 `swoole_coroutine_create()`，并可 alias 成 `go` |
| 模块初始化 | `PHP_MINIT_FUNCTION(swoole)` | 注册类、常量、别名 |

**跟读步骤（打开 `ext-src/php_swoole.cc`）：**

1. 搜索 `zend_module_entry swoole_module_entry` — 看 `PHP_MINIT(swoole)` 在哪一栏。  
2. 搜索 `PHP_FE(swoole_coroutine_create` — 确认 `go` 的来源。  
3. 搜索 `SW_FUNCTION_ALIAS` 和 `"go"` — 短名 `go()` 是别名，不是另一个实现。  
4. 搜索 `php_swoole_coroutine_minit` — 看 `Swoole\Coroutine` 等类何时 `REGISTER`。

**你要记住**：没有 `MINIT` 注册，PHP 里就找不到 `go` / `Swoole\Coroutine` 类。

---

## 3. 协程相关类注册顺序（15 分钟）

在 `PHP_MINIT_FUNCTION(swoole)` 末尾附近，顺序大致是：

```
php_swoole_coroutine_minit          → Swoole\Coroutine
php_swoole_coroutine_scheduler_minit → Swoole\Coroutine\Scheduler (Co\Run)
php_swoole_channel_coro_minit       → Swoole\Coroutine\Channel
php_swoole_runtime_minit            → Runtime Hook（第 3 周再学）
```

**练习**：在 IDE 对 `php_swoole_coroutine_minit` 跳转到定义，看注册了哪些 `PHP_METHOD`（今天扫一眼名单即可）。

---

## 4. 与 Day 1 串联（10 分钟）

填空（写进笔记）：

```
php -m 加载 swoole.so
  → PHP_MINIT(swoole)
    → 注册 go / Swoole\Coroutine / Scheduler
      → 用户调用 Co\Run
        → Scheduler::start → php_swoole_event_wait
      → 用户调用 go()
        → swoole_coroutine_create → PHPCoroutine::create
```

---

## 5. Day 2 自检

| 问题 | 参考答案 |
|------|----------|
| `go` 是独立 C 函数吗？ | 否，是 `swoole_coroutine_create` 的别名 |
| 类在哪注册？ | 各 `php_swoole_*_minit`，在 `PHP_MINIT(swoole)` 里调用 |
| `zend_module_entry` 干什么？ | 扩展模块元数据 + 生命周期钩子 |

---

# 领读 Day 3：`go()` 完整调用链（约 60 分钟）

> 这是第一周最重要的一天。按顺序在 IDE 里**跳转**，每段只读标出的函数。

## 1. 再跑一个嵌套协程（5 分钟）

```bash
php -r '
go(function () {
    go(function () {
        echo "inner\n";
    });
    echo "outer\n";
});
'
```

若没有 `Co\Run` 也能跑：说明裸 `go()` 会**自动初始化**事件循环（`src/coroutine/base.cc` 里 `swoole_coroutine_create` 分支）。对比 Day 1 的 `Co\Run`，两条路径都要会画。

---

## 2. 六级调用链（35 分钟）

在纸上写下这 6 步，然后逐步在代码里核对：

```
① PHP: go(function () { ... })
② ext: PHP_FUNCTION(swoole_coroutine_create)
③ ext: PHPCoroutine::create(...)
④ ext: Coroutine::create(main_func, &args)     [C++ 头文件 inline]
⑤ src: Coroutine::run() → ctx.swap_in()
⑥ ext: PHPCoroutine::main_func → zend_call_function(...)  // 真正执行 PHP 闭包
```

### ② → ③ `PHPCoroutine::create`

文件：`ext-src/swoole_coroutine.cc`

```cpp
long PHPCoroutine::create(...) {
    ...
    return Coroutine::create(main_func, (void *) &_args);
}
```

要点：`main_func` 是 C++ 回调，`_args` 里装着 PHP 闭包的 `fci_cache`。

### ⑥ `main_func` 里执行闭包

同文件：

```cpp
void PHPCoroutine::main_func(void *_args) {
    PHPContext *ctx = create_context(args);  // 为协程分配 PHP 虚拟机栈
    ...
    zend_call_function(&ctx->fci, &ctx->fci_cache);  // ← 你的 closure 在这里跑
    ...
    destroy_context(ctx);
}
```

**为什么要 `create_context`**：每个协程要有**自己的 PHP 执行栈**，否则 A 协程跑到一半被 B 协程打断会乱套（Day 5 日程展开）。

### ④⑤ C++ 侧

头文件 `include/swoole_coroutine.h`：

```cpp
static long create(const CoroutineFunc &fn, void *args = nullptr) {
    return (new Coroutine(fn, args))->run();
}
```

`src/coroutine/base.cc` 的 `run()` 见 Day 1 领读 3.3。

---

## 3. 裸 `go()` vs `Co\Run`（10 分钟）

打开 `src/coroutine/base.cc`，看 `swoole_coroutine_create`（C API，给内部用）：

- 若**已有**事件循环 → 直接 `Coroutine::create`  
- 若**没有** → `swoole_event_init` → `create` → `swoole_event_wait` → `deactivate`

**对比表：**

| | `Co\Run` | 裸 `go()` |
|--|----------|-----------|
| 事件循环 | Scheduler 显式 `reactor_init` + `event_wait` | 可能自动 init |
| 适用场景 | 推荐：边界清晰 | 脚本顶层偶尔用 |

---

## 4. Day 3 自检（要能背诵级别）

1. 画出 6 步调用链。  
2. PHP 闭包在哪一行 C 代码执行？→ `zend_call_function` in `main_func`。  
3. `Coroutine::create` 在 `.cc` 还是 `.h`？→ `.h` 内联，内部 `new` + `run()`。  
4. 裸 `go()` 为何有时不需要 `Co\Run`？→ 自动 `event_init` + `event_wait`。

---

## 领读结束后怎么走

| 已完成 | 下一步 |
|--------|--------|
| 领读 Day 1–3 | [LEARNING-PLAN.md](LEARNING-PLAN.md) **D4–D7**（Scheduler 细节、状态机、Channel、context） |
| 仍觉得抽象 | 重复领读 Day 3，不要跳到 D8 |
| 想巩固 PHP 扩展 | 做 [LEARNING-PLAN.md](LEARNING-PLAN.md) 底部「扩展补课」3 条 |

后续周（Reactor、Hook、Server…）仍用日程表；若某周仍太宽，可在 issue 里提议加「领读 Day N」续篇。

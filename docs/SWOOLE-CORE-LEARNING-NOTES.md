# Swoole Core Learning Notes

本文记录一次面向 Swoole 底层源码学习的阶段性总结。目标不是 API 教程，而是帮助学习者在 CLion 中读源码、打断点、跟调用链，并逐步具备定位协程与 Runtime Hook 问题的能力。

当前学习重点：

- `Co\run()` / `Swoole\Coroutine\run()` 如何启动协程程序
- `go()` / `Swoole\Coroutine::create()` 如何进入 C++ 协程层
- `PHPCoroutine::create()` 如何把 Zend callable 转成 Swoole coroutine
- CLion 如何调试由 `php` 进程动态加载的 `swoole.dylib`

## 构建与调试环境

当前建议以 Debug 构建为主，方便在 CLion 中断点调试。

CMake 配置参考：

```bash
/Users/lux/Applications/CLion.app/Contents/bin/cmake/mac/aarch64/bin/cmake \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_MAKE_PROGRAM=/Users/lux/Applications/CLion.app/Contents/bin/ninja/mac/aarch64/ninja \
  -G Ninja \
  -S /Users/lux/Workspace/clang/swoole-src \
  -B /Users/lux/Workspace/clang/swoole-src/cmake-build-debug \
  -Dlibpq_dir=/opt/homebrew/opt/libpq \
  -Dphp_dir=/opt/homebrew/opt/php \
  -Dopenssl_dir=/opt/homebrew/opt/openssl@3 \
  -Dbrotli_dir=/opt/homebrew/opt/brotli \
  -DCMAKE_CXX_FLAGS='-I/opt/homebrew/opt/pcre2/include -I/opt/homebrew/opt/brotli/include' \
  -DCMAKE_C_FLAGS='-I/opt/homebrew/opt/pcre2/include -I/opt/homebrew/opt/brotli/include'
```

构建扩展：

```bash
LIBRARY_PATH=/opt/homebrew/opt/libpq/lib:/opt/homebrew/opt/curl/lib:/opt/homebrew/opt/brotli/lib:/opt/homebrew/opt/openssl@3/lib \
/Users/lux/Applications/CLion.app/Contents/bin/ninja/mac/aarch64/ninja \
  -C /Users/lux/Workspace/clang/swoole-src/cmake-build-debug \
  ext-swoole
```

调试时加载的扩展路径：

```text
/Users/lux/Workspace/clang/swoole-src/lib/swoole.dylib
```

最小验证命令：

```bash
/opt/homebrew/opt/php/bin/php \
  -n \
  -d extension=/Users/lux/Workspace/clang/swoole-src/lib/swoole.dylib \
  -r 'echo swoole_version(), PHP_EOL; Co\run(function(){ go(function(){ echo Co::getCid(), PHP_EOL; }); });'
```

## 源码模块地图

学习协程主线时优先看这些目录：

```text
ext-src/
  PHP 扩展绑定层，包含 PHP 类、函数、MINIT/RINIT、Zend 参数解析、PHPCoroutine。

src/coroutine/
  Swoole 底层 C++ 协程实现，包含 Coroutine、Context、yield/resume、栈切换。

src/reactor/
  事件循环实现，macOS 主要看 kqueue，Linux 主要看 epoll。

include/
  核心类型声明，例如 Coroutine、coroutine::Context、Socket、Reactor 等。

tests/
  PHPT 扩展测试。

core-tests/
  C++ GTest 核心测试。
```

本阶段的核心文件：

```text
ext-src/php_swoole_library.h
ext-src/swoole_coroutine.cc
ext-src/php_swoole_coroutine.h
ext-src/swoole_coroutine_scheduler.cc
ext-src/swoole_event.cc
include/swoole_coroutine.h
src/coroutine/base.cc
src/coroutine/context.cc
```

## Co\run 与 go 的使用边界

项目入口建议：

```php
Co\run(function () {
    go(function () {
        // child coroutine
    });
});
```

原则：

- CLI 协程程序入口使用 `Co\run()`。
- 已经在协程、Scheduler 或 Swoole Server 回调里时，用 `go()` 创建子协程。
- Swoole Server 项目的生命周期入口是 `$server->start()`，不是 `Co\run()`。
- `Co\run()` 默认设置 `hook_flags = SWOOLE_HOOK_ALL`，直接顶层 `go()` 不经过这个 wrapper。

所以：

```php
Co\run(function () {
    sleep(1);
});
```

默认情况下 `sleep()` 会被 Runtime Hook 协程化。

## Co\run 源码主线

`Co\run()` 只是短名，实际转发到 `Swoole\Coroutine\run()`。

源码位于 `ext-src/php_swoole_library.h`，去掉 C 字符串包装后可读作：

```php
namespace Swoole\Coroutine;

use Swoole\Coroutine;

function run(callable $fn, ...$args)
{
    $s       = new Scheduler();
    $options = Coroutine::getOptions();
    if (!isset($options['hook_flags'])) {
        $s->set(['hook_flags' => SWOOLE_HOOK_ALL]);
    }
    $s->add($fn, ...$args);
    return $s->start();
}
```

逐行理解：

- `namespace Swoole\Coroutine`：函数全名是 `Swoole\Coroutine\run()`。
- `use Swoole\Coroutine`：下面的 `Coroutine::getOptions()` 指向类 `\Swoole\Coroutine`。
- `new Scheduler()`：创建 PHP 层 Scheduler 对象，底层是 `SchedulerObject`。
- `Coroutine::getOptions()`：读取全局协程配置。
- `!isset($options['hook_flags'])`：只有用户没显式设置 hook 时才设置默认值。
- `$s->set(['hook_flags' => SWOOLE_HOOK_ALL])`：默认开启 Runtime Hook。
- `$s->add($fn, ...$args)`：把用户主函数保存到 scheduler 队列，还不执行。
- `$s->start()`：真正初始化 Reactor、创建协程并进入事件循环。

短名 `Co\run()` 的源码逻辑等价于：

```php
namespace Co;

function run(callable $fn, ...$args)
{
    return \Swoole\Coroutine\run($fn, ...$args);
}
```

## Scheduler::start 调用链

`Scheduler::start()` 位于 `ext-src/swoole_coroutine_scheduler.cc`。

核心流程：

```cpp
static PHP_METHOD(swoole_coroutine_scheduler, start) {
    SchedulerObject *s = scheduler_get_object(Z_OBJ_P(ZEND_THIS));

    if (s->started) {
        RETURN_FALSE;
    }
    if (php_swoole_reactor_init() < 0) {
        RETURN_FALSE;
    }

    s->started = true;

    if (!s->list) {
        RETURN_FALSE;
    }

    while (!s->list->empty()) {
        SchedulerTask *task = s->list->front();
        s->list->pop();
        for (zend_long i = 0; i < task->count; i++) {
            PHPCoroutine::create(&task->fci_cache, task->fci.param_count, task->fci.params, &task->fci.function_name);
        }
        sw_zend_fci_cache_discard(&task->fci_cache);
        sw_zend_fci_params_discard(&task->fci);
        efree(task);
    }
    php_swoole_event_wait();
    delete s->list;
    s->list = nullptr;
    s->started = false;
    RETURN_TRUE;
}
```

关键点：

- `scheduler_get_object()`：从 PHP `$this` 取到底层 `SchedulerObject`。
- `php_swoole_reactor_init()`：初始化事件循环 Reactor。
- `s->started = true`：防止 scheduler 运行期间继续 add/start。
- `PHPCoroutine::create(...)`：把 Zend callable 创建为 Swoole PHP 协程。
- `php_swoole_event_wait()`：进入 `sw_reactor()->wait()`，等待协程、timer、socket I/O 等结束。

简化调用链：

```text
Co\run($fn)
  -> Swoole\Coroutine\run($fn)
    -> new Scheduler
    -> Scheduler::set(['hook_flags' => SWOOLE_HOOK_ALL])
    -> Scheduler::add($fn)
    -> Scheduler::start()
      -> php_swoole_reactor_init()
      -> PHPCoroutine::create(...)
      -> php_swoole_event_wait()
```

## CLion 调试方式

调试 Swoole 扩展时，CLion 启动的 executable 应该是 PHP，不是 PHP 脚本本身。

建议准备一个最小脚本：

```php
<?php

Co\run(function () {
    echo "main cid=", Co::getCid(), PHP_EOL;

    go(function () {
        echo "child cid=", Co::getCid(), " pcid=", Co::getPcid(), PHP_EOL;
        sleep(1);
        echo "child done", PHP_EOL;
    });

    echo "main done", PHP_EOL;
});
```

CLion Run/Debug Configuration：

```text
Executable:
/opt/homebrew/opt/php/bin/php

Program arguments:
-n -d extension=/Users/lux/Workspace/clang/swoole-src/lib/swoole.dylib /Users/lux/Workspace/clang/swoole-src/debug/co_run.php

Working directory:
/Users/lux/Workspace/clang/swoole-src
```

不要第一轮直接 Debug `tests/start.sh`。PHPT runner 会再派生 PHP 子进程，CLion 断点容易跟不到真正加载 `swoole.dylib` 的进程。学习源码时更适合先用最小 PHP 脚本。

第一组断点：

```text
ext-src/swoole_coroutine_scheduler.cc:179   Scheduler::set
ext-src/swoole_coroutine_scheduler.cc:236   Scheduler::add
ext-src/swoole_coroutine_scheduler.cc:276   Scheduler::start
ext-src/swoole_event.cc:219                 php_swoole_reactor_init
ext-src/swoole_coroutine.cc:813             PHPCoroutine::create
ext-src/swoole_coroutine.cc:403             PHPCoroutine::activate
include/swoole_coroutine.h:147              Coroutine::create
src/coroutine/base.cc:76                    Coroutine::Coroutine
src/coroutine/base.cc:96                    Coroutine::run
src/coroutine/context.cc:122                Context::swap_in
ext-src/swoole_event.cc:255                 php_swoole_event_wait
```

观察顺序：

```text
Scheduler::set()
  -> 确认 Co\run 默认设置 SWOOLE_HOOK_ALL

Scheduler::add()
  -> 观察 task->fci、task->fci_cache、task->count

Scheduler::start()
  -> 观察 s->list、s->started

php_swoole_reactor_init()
  -> 观察 sw_reactor() 是否从 null 变成有效 Reactor

PHPCoroutine::create()
  -> 观察 fci_cache、argc、argv、activated、config.hook_flags

Coroutine::run()
  -> 观察 cid、origin、current、state

Context::swap_in()
  -> 观察 C 栈切换进入协程入口
```

## PHPCoroutine::create 深入总结

`PHPCoroutine::create()` 是 Zend callable 进入 Swoole 底层协程的桥。

源码位于 `ext-src/swoole_coroutine.cc`：

```cpp
long PHPCoroutine::create(zend_fcall_info_cache *fci_cache, uint32_t argc, zval *argv, zval *callable) {
    if (sw_unlikely(Coroutine::count() >= config.max_num)) {
        php_swoole_fatal_error(E_WARNING, "exceed max number of coroutine %zu", (uintmax_t) Coroutine::count());
        return Coroutine::ERR_LIMIT;
    }
    if (sw_unlikely(!fci_cache || !fci_cache->function_handler)) {
        php_swoole_fatal_error(E_ERROR, "invalid function call info cache");
        return Coroutine::ERR_INVALID;
    }
    zend_uchar type = fci_cache->function_handler->type;
    if (sw_unlikely(type != ZEND_USER_FUNCTION && type != ZEND_INTERNAL_FUNCTION)) {
        php_swoole_fatal_error(E_ERROR, "invalid function type %u", fci_cache->function_handler->type);
        return Coroutine::ERR_INVALID;
    }

    if (sw_unlikely(!activated)) {
        activate();
    }

    Args _args;
    _args.fci_cache = fci_cache;
    _args.argv = argv;
    _args.argc = argc;
    _args.callable = callable;
    save_context(get_context());

    return Coroutine::create(main_func, (void *) &_args);
}
```

入口来源：

```text
go($fn, ...$args)
  -> PHP_FUNCTION(swoole_coroutine_create)
    -> PHPCoroutine::create(&fci_cache, fci.param_count, fci.params, &fci.function_name)

Co\run($fn, ...$args)
  -> Scheduler::add($fn)
  -> Scheduler::start()
    -> PHPCoroutine::create(&task->fci_cache, task->fci.param_count, task->fci.params, &task->fci.function_name)
```

参数含义：

- `zend_fcall_info_cache *fci_cache`：Zend callable cache，核心是 `function_handler`。
- `uint32_t argc`：参数个数。
- `zval *argv`：参数数组。
- `zval *callable`：callable 的 PHP zval 表示。

`Args` 定义在 `ext-src/php_swoole_coroutine.h`：

```cpp
struct Args {
    zend_fcall_info_cache *fci_cache;
    zval *argv;
    uint32_t argc;
    zval *callable;
};
```

`PHPCoroutine::create()` 的关键步骤：

1. 检查协程数量是否超过 `config.max_num`。
2. 检查 `fci_cache` 和 `function_handler` 是否有效。
3. 检查 function handler 类型，只允许 `ZEND_USER_FUNCTION` 和 `ZEND_INTERNAL_FUNCTION`。
4. 首次创建协程时调用 `activate()`。
5. 用 `Args _args` 打包 callable 和参数。
6. `save_context(get_context())` 保存当前 PHP VM 上下文。
7. 调用 `Coroutine::create(main_func, &_args)` 进入底层协程层。

`activate()` 的关键作用：

```text
1. 加载 Swoole 内置 PHP library
2. 初始化 / 检查 Reactor
3. 替换 Zend interrupt function
4. 根据 config.hook_flags 启用 Runtime Hook
5. 激活底层 Coroutine
6. 注册 on_yield / on_resume / on_close
```

其中最重要的是：

```cpp
Coroutine::set_on_yield(on_yield);
Coroutine::set_on_resume(on_resume);
Coroutine::set_on_close(on_close);
```

这三行把底层 C++ 协程切换事件和 PHP VM 上下文保存恢复机制连起来。

`save_context(get_context())` 保存：

```text
save_vm_stack(ctx)
  -> EG(bailout)
  -> EG(vm_stack_top)
  -> EG(vm_stack_end)
  -> EG(vm_stack)
  -> EG(vm_stack_page_size)
  -> EG(current_execute_data)
  -> EG(jit_trace_num)
  -> EG(error_handling)
  -> EG(exception_class)
  -> EG(exception)

save_og(ctx)
  -> output globals

save_bg(ctx)
  -> serialize / unserialize 相关 basic globals
```

这说明 Swoole 协程切换不是只切 C 栈，还要保存和恢复 Zend VM 栈。

## main_func 与 PHP callable 执行

`Coroutine::create(main_func, &_args)` 会立即创建并运行底层协程：

```cpp
static long create(const CoroutineFunc &fn, void *args = nullptr) {
    return (new Coroutine(fn, args))->run();
}
```

所以 `_args` 虽然是 `PHPCoroutine::create()` 的局部变量，但它会在当前函数返回前被新协程入口 `main_func()` 消费，不会悬空。

新协程真正入口是：

```cpp
void PHPCoroutine::main_func(void *_args) {
    bool exception_caught = false;
    const auto args = static_cast<Args *>(_args);
    PHPContext *ctx = create_context(args);

    zend_first_try {
        zend_call_function(&ctx->fci, &ctx->fci_cache);
        exception_caught = catch_exception();

        if (ctx->defer_tasks) {
            // run defer tasks
        }
    }
    zend_catch {
        catch_exception();
        exception_caught = true;
    }
    zend_end_try();

    destroy_context(ctx);
    if (exception_caught) {
        bailout();
    }
}
```

重点：

- `create_context(args)` 创建当前协程自己的 `PHPContext`。
- `zend_call_function(&ctx->fci, &ctx->fci_cache)` 才是真正执行用户 PHP callable 的位置。
- `defer_tasks` 在主函数结束或异常后执行。
- `destroy_context(ctx)` 释放当前协程的 PHP VM 栈、callable 引用和上下文。

`create_context()` 会做三件关键事：

```cpp
ctx->co = Coroutine::get_current();
ctx->co->set_task((void *) ctx);
ctx->pcid = ctx->co->get_origin_cid();
```

这把底层 `Coroutine` 与 PHP 层 `PHPContext` 绑定起来。

然后为当前协程创建新的 Zend VM stack：

```cpp
EG(vm_stack) = zend_vm_stack_new_page(SW_DEFAULT_PHP_STACK_PAGE_SIZE, nullptr);
EG(vm_stack_top) = EG(vm_stack)->top + ZEND_CALL_FRAME_SLOT;
EG(vm_stack_end) = EG(vm_stack)->end;
EG(vm_stack_page_size) = SW_DEFAULT_PHP_STACK_PAGE_SIZE;
```

所以 Swoole PHP 协程具备两层隔离：

```text
底层 coroutine::Context
  -> C 栈隔离

PHPContext
  -> Zend VM 栈与执行状态隔离
```

## Yield / Resume 与 PHP VM 上下文

在 `activate()` 中，Swoole 注册了：

```cpp
Coroutine::set_on_yield(on_yield);
Coroutine::set_on_resume(on_resume);
Coroutine::set_on_close(on_close);
```

底层协程 yield 时：

```cpp
void Coroutine::yield() {
    state = STATE_WAITING;
    if (on_yield && task) {
        on_yield(task);
    }
    current = origin;
    ctx.swap_out();
}
```

PHP 层回调：

```cpp
void PHPCoroutine::on_yield(void *arg) {
    auto *ctx = static_cast<PHPContext *>(arg);
    auto *origin_ctx = get_origin_context(ctx);

    save_context(ctx);
    restore_context(origin_ctx);
}
```

resume 时相反：

```cpp
void PHPCoroutine::on_resume(void *arg) {
    auto *ctx = static_cast<PHPContext *>(arg);
    auto *current_ctx = get_context();

    save_context(current_ctx);
    restore_context(ctx);
}
```

因此一次协程切换包含两条线：

```text
C++ coroutine::Context
  -> swap_in / swap_out 切换 C 栈

PHPCoroutine::PHPContext
  -> save_context / restore_context 切换 Zend VM 执行状态
```

## 阶段性调用链总览

```text
Co\run($fn)
  -> Swoole\Coroutine\run($fn)
    -> new Scheduler
    -> Scheduler::set(['hook_flags' => SWOOLE_HOOK_ALL])
    -> Scheduler::add($fn)
      -> 保存 zend_fcall_info / zend_fcall_info_cache / 参数
    -> Scheduler::start()
      -> php_swoole_reactor_init()
      -> PHPCoroutine::create(...)
        -> 检查 max_num
        -> 检查 callable
        -> activate()
          -> enable_hook(config.hook_flags)
          -> Coroutine::set_on_yield / on_resume / on_close
        -> Args _args
        -> save_context(get_context())
        -> Coroutine::create(main_func, &_args)
          -> new Coroutine(main_func, &_args)
          -> Coroutine::run()
            -> origin = current
            -> current = this
            -> ctx.swap_in()
              -> PHPCoroutine::main_func(&_args)
                -> create_context(args)
                -> zend_call_function(&ctx->fci, &ctx->fci_cache)
                  -> 执行用户 PHP callable
      -> php_swoole_event_wait()
        -> sw_reactor()->wait()
```

## 下一阶段学习建议

下一步建议继续沿着这几条线深入：

1. `PHPCoroutine::main_func()` 逐行解读，重点是 `create_context()`、`zend_call_function()`、`defer`、异常与 `destroy_context()`。
2. `Coroutine::run()` / `yield()` / `resume()` 逐行解读，重点是 `origin/current/state` 和 `ctx.swap_in/swap_out`。
3. Runtime Hook 主线，重点看 `PHPCoroutine::enable_hook()`、`sleep()` hook、stream transport factory 替换。
4. Socket 到 Reactor 主线，重点看 `Socket::wait_event()`、`Reactor::add_event()`、kqueue/epoll wait 和 resume。
5. Server 模型，等协程与 Reactor 主线稳定后再进入 `Server::start()`、master/reactor/worker/task worker。

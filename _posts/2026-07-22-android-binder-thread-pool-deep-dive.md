---
title: 深入拆解 Binder 线程池：从用户态线程到内核调度
date: 2026-07-22 10:00:00 +0800
categories: [Android开发]
tags: [Android, Binder, 源码分析, 进程通信]
---

## 零、引言

在 Android 开发中，`bindService` 发起一次跨进程调用，然后在 `onServiceConnected` 里拿到远端对象的代理，接着像调本地方法一样调用它的接口——这是我们最熟悉的 Binder 用法之一。但你是否认真追问过：服务端进程里，到底是谁在执行你的 `onTransact`？那些叫 `Binder:xxx_1`、`Binder:xxx_2` 的线程，是怎么来的？又是什么时候被唤醒的？

Binder 线程池源码容易越读越散，原因通常不是某个函数太难，而是一上来就钻进 `ProcessState`、`IPCThreadState`、`binder_proc`、`binder_thread` 这些对象，看到的是结构体和函数，却没有先回答最根本的问题：线程池在哪一层？谁创建线程？谁调度线程？谁决定扩容？

本文从一个最朴素的问题出发：**Binder 线程池里的线程，没活干的时候在干什么？** 逐步深入到 App 进程启动时 Binder 线程池如何自动启动、`ProcessState` 如何连接 driver、`IPCThreadState` 如何进入主循环、kernel 如何维护等待队列与分发事务，以及 `BR_SPAWN_LOOPER` 如何触发线程池按需扩容。

一句话先给结论：**Binder 线程池不是一个普通的“用户态任务队列 + worker 线程”模型，而是“用户态线程睡在 driver 上，kernel 维护等待队列、分发事务、唤醒线程并请求扩容”的协作模型。**

---

## 一、先看结构，而不是先看源码

一句话结论：Binder 线程池 = 用户态创建和运行 Binder looper 线程 + 内核态 binder driver 维护调度状态，双方通过 `ioctl` 上的 `BC_*` / `BR_*` 命令协作。kernel 不创建 `pthread`，只记录状态、分发事务、唤醒线程和请求扩容。

先回答 4 个最关键的问题：

| 问题 | 答案 |
| --- | --- |
| 谁创建线程？ | 用户态 libbinder。`ProcessState::spawnPooledThread()` 真正调用 `pthread_create` 创建线程。 |
| 线程跑什么？ | 每个 Binder looper 线程进入 `IPCThreadState::joinThreadPool()`，阻塞在 `ioctl(BINDER_WRITE_READ)` 上等待 driver 返回命令。 |
| 谁决定把请求分给哪个线程？ | 内核态 binder driver。它维护空闲线程队列 `waiting_threads`，新事务到来时挑一个空闲线程唤醒或放入进程队列。 |
| 谁决定要扩容？ | kernel driver。当没有空闲线程且未达上限时，driver 返回 `BR_SPAWN_LOOPER`，用户态收到后才创建新线程。 |

6 个核心对象及其职责：

| 层 | 对象 | 是什么 | 核心职责 | 关键字段/方法 |
| --- | --- | --- | --- | --- |
| 用户态 | `ProcessState` | 进程级单例 | 打开 `/dev/binder`、设置最大线程数、创建 `PoolThread` | `mDriverFD`、`mMaxThreads`、`startThreadPool()`、`spawnPooledThread()` |
| 用户态 | `IPCThreadState` | 线程本地单例 | 每个 Binder 线程持有一个，负责和 driver 收发命令、执行 `BR_*` | `mIn/mOut`、`joinThreadPool()`、`talkWithDriver()`、`executeCommand()` |
| 用户态 | `PoolThread` | libbinder 创建的真实 `pthread` | 线程实体，`threadLoop()` 进入 `joinThreadPool()` | 线程名 `binder:<pid>_<seq>` |
| 内核态 | `binder_proc` | driver 为每个进程维护的状态 | 管理该进程所有 `binder_thread`、空闲线程队列、进程级 todo、线程计数 | `threads`、`waiting_threads`、`todo`、`max_threads`、`requested_threads_started` |
| 内核态 | `binder_thread` | driver 为每个 Binder looper 维护的影子状态 | 对应用户态一个真实线程，记录 looper 状态、事务栈、线程私有 todo | `looper`、`transaction_stack`、`todo`、`wait` |
| 内核态 | `binder_transaction` | 一次跨进程调用 | 真正在队列中排队和分发的工作单元 | 经由 `binder_work` 挂入 `thread.todo` 或 `proc.todo` |

整体关系图：

```text
                          ┌─────────────────────────────────────────┐
                          │            一个 Android 进程              │
                          │                                         │
  BC_TRANSACTION ────────►│  ┌─────────────┐                         │
  BC_REPLY         ioctl  │  │ ProcessState│◄── 进程级：连接 driver、 │
  BC_ENTER_LOOPER  ──────►│  │ (用户态单例)│     设置 maxThreads、    │
  BC_REGISTER_LOOPER      │  └──────┬──────┘     创建 PoolThread      │
  BC_EXIT_LOOPER          │         │ spawnPooledThread()             │
                          │         ▼                                 │
                          │  ┌─────────────┐    ┌─────────────┐      │
                          │  │ PoolThread 1│    │ PoolThread 2│ ...  │  ← 用户态真实 pthread
                          │  │ (isMain=true)│   │  (lazy)     │      │
                          │  └──────┬──────┘    └──────┬──────┘      │
                          │         │ joinThreadPool()  │             │
                          │         ▼                   ▼             │
                          │  ┌─────────────────────────────────┐     │
                          │  │        IPCThreadState            │     │  ← 每线程持有一个
                          │  │  talkWithDriver()                │     │
                          │  │  executeCommand() -> onTransact()│     │
                          │  └──────────────┬──────────────────┘     │
                          │                 │ BINDER_WRITE_READ       │
                          └─────────────────┼─────────────────────────┘
                                            │ ioctl
                          ┌─────────────────┼─────────────────────────┐
                          │  binder driver  ▼                         │
                          │  (内核态)                                │
                          │  ┌──────────────────────────────────┐     │
                          │  │  binder_proc (每个进程一个)        │     │
                          │  │  - waiting_threads (空闲线程队列)  │     │
                          │  │  - todo (进程级工作队列)           │     │
                          │  │  - max_threads                    │     │
                          │  │  - requested_threads_started      │     │
                          │  └──────┬───────────────┬───────────┘     │
                          │         │               │                 │
                          │         ▼               ▼                 │
                          │  ┌────────────┐  ┌────────────┐           │
                          │  │binder_thr 1│  │binder_thr 2│  ...      │  ← 用户态线程在内核的影子
                          │  │ - looper   │  │ - looper   │           │
                          │  │ - txn_stack│  │ - txn_stack│           │
                          │  │ - todo     │  │ - todo     │           │
                          │  └────────────┘  └────────────┘           │
                          │                                         │
                          │  BR_TRANSACTION ──► 唤醒线程执行          │
                          │  BR_REPLY       ──► 唤醒等待同步返回的线程 │
                          │  BR_SPAWN_LOOPER ──► 请求用户态创建新线程  │
                          └─────────────────────────────────────────┘
```

记住这张图，后续所有代码只是在把上述关系落地。`ProcessState` 解决“进程怎么连 driver、怎么创建线程”，`IPCThreadState` 解决“线程怎么成为 looper、怎么收发命令”，`binder_proc` / `binder_thread` 解决“driver 怎么选线程、怎么排队、怎么唤醒”，`BR_SPAWN_LOOPER` 解决“什么时候扩容”。

---

### 这套机制为什么会这样设计？

一句话结论：Binder 把“对象调用语义”放在用户态，把“事务分发与并发调度”放在 kernel，线程池正是这个分工的产物。

| 关注点 | 谁负责 | 为什么放在这一层 |
| --- | --- | --- |
| Binder 对象语义、方法调用、Parcel 序列化 | 用户态 libbinder / 框架层 | 贴近业务逻辑，易于扩展 |
| 真实线程的创建和执行 | 用户态 libbinder | `pthread` 是进程资源，kernel 不应替进程创建线程 |
| 事务缓冲、引用计数、死亡通知 | kernel driver | 需要跨进程一致性保证 |
| 等待队列、线程唤醒、分发决策 | kernel driver | 涉及调度，放在内核可避免用户态忙轮询和竞争 |
| 何时需要更多线程 | kernel driver | driver 全局掌握“有没有空闲线程、队列是否为空” |

普通线程池是“用户态队列 + 用户态 worker 取任务”；Binder 线程池是“用户态 worker 睡在内核上 + kernel 直接唤醒/分发”。这决定了扩容信号必须从 driver 来，而不是用户态自己轮询判断。

---

## 二、从进程启动到第一个 Binder 线程

一句话结论：普通 App 进程在 zygote fork 后的 native 初始化阶段自动启动 Binder 线程池——先打开 `/dev/binder` 建立连接、设置最大线程数、`mmap` 事务缓冲区，然后主动创建 1 个主 `PoolThread`。

两条典型启动路径：

| 进程类型 | 启动方式 | 说明 |
| --- | --- | --- |
| App 进程 | zygote child 自动触发 | `ZygoteInit → nativeZygoteInit → onZygoteInit → ProcessState::self()->startThreadPool()` |
| Native service | 进程自己显式调用 | `ProcessState::self()->startThreadPool(); IPCThreadState::self()->joinThreadPool();` 通常让主线程也成为 Binder looper |

App 进程链路起点：

```java
// frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public static Runnable zygoteInit(...) {
    ...
    ZygoteInit.nativeZygoteInit();   // 进入 native 初始化
    return RuntimeInit.applicationInit(...);
}
```

`nativeZygoteInit()` 经过 JNI 进入 `app_main.cpp` 中的 `AppRuntime::onZygoteInit()`：

```cpp
// frameworks/base/cmds/app_process/app_main.cpp
virtual void onZygoteInit() {
    sp<ProcessState> proc = ProcessState::self();
    proc->startThreadPool();
}
```

这两行做了两件事：`ProcessState::self()` 触发打开 driver；`startThreadPool()` 创建第一个 Binder 线程。

### 2.1 连接 driver

`ProcessState::self()` 首次调用时创建单例，构造函数做 3 件事：

源码位置：`frameworks/native/libs/binder/ProcessState.cpp`

```cpp
#define DEFAULT_MAX_BINDER_THREADS 15                        // lazy 线程上限默认值
#define BINDER_VM_SIZE ((1*1024*1024) - sysconf(_SC_PAGE_SIZE)*2)  // 事务缓冲区 ~1MB - 2页

static unique_fd open_driver(const char* driver, String8* error) {
    auto fd = unique_fd(open(driver, O_RDWR | O_CLOEXEC));
    ioctl(fd.get(), BINDER_VERSION, &vers);                 // 校验协议版本
    size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
    ioctl(fd.get(), BINDER_SET_MAX_THREADS, &maxThreads);   // 告诉 driver lazy 上限 15
    uint32_t enable = DEFAULT_ENABLE_ONEWAY_SPAM_DETECTION;
    ioctl(fd.get(), BINDER_ENABLE_ONEWAY_SPAM_DETECTION, &enable);
    return fd;
}
```

构造函数流程：

```text
ProcessState::ProcessState("/dev/binder")
  -> open_driver()
      -> open("/dev/binder")                        // 获取 fd
      -> ioctl(BINDER_VERSION)                      // 版本校验
      -> ioctl(BINDER_SET_MAX_THREADS, 15)          // 设置 lazy 线程上限
      -> ioctl(BINDER_ENABLE_ONEWAY_SPAM_DETECTION) // 开启 oneway 限流检测
  -> mmap(BINDER_VM_SIZE)                           // 映射事务接收缓冲区
  -> 保存 mDriverFD / mVMStart
```

这一步只建立连接、设置进程级参数、映射缓冲区，还没有创建任何 Binder 线程。也就是说，`ProcessState::self()` 更像是“把当前进程接入 Binder driver”，真正的服务线程要等 `startThreadPool()` 才会出现。

### 2.2 创建第一个线程

`startThreadPool()` 只做一件事：创建 1 个 `isMain=true` 的主 `PoolThread`。

```cpp
void ProcessState::startThreadPool() {
    std::unique_lock<std::mutex> _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);    // isMain=true
    }
}
```

`spawnPooledThread()` 真正创建 `pthread`：

```cpp
void ProcessState::spawnPooledThread(bool isMain) {
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();   // 形如 binder:<pid>_<seq>
        sp<Thread> t = sp<PoolThread>::make(isMain);
        t->run(name.c_str());                    // 启动新 pthread
        mKernelStartedThreads++;
    }
}
```

`PoolThread` 的线程体只有一行：

```cpp
class PoolThread : public Thread {
    virtual bool threadLoop() {
        IPCThreadState::self()->joinThreadPool(mIsMain);  // 进入 Binder looper 主循环
        return false;
    }
};
```

从进程启动到第一个 Binder 线程就绪的完整链路：

```text
ZygoteInit.zygoteInit()
  -> nativeZygoteInit()
  -> AppRuntime::onZygoteInit()
  -> ProcessState::self()
       -> open("/dev/binder")
       -> ioctl(BINDER_SET_MAX_THREADS, 15)
       -> mmap() 事务缓冲区
  -> startThreadPool()
       -> spawnPooledThread(true)         // 创建主 PoolThread（pthread）
       -> PoolThread.threadLoop()
       -> IPCThreadState.joinThreadPool(true)   // -> 进入下一章讲的 looper 主循环
```

后续线程不是预先创建，而是 driver 在需要时返回 `BR_SPAWN_LOOPER` 按需扩容。

---

## 三、Binder looper 主循环：joinThreadPool

一句话结论：每个 Binder 线程进入 `joinThreadPool()` 后，先向 driver 注册自己是 looper，然后循环调用 `talkWithDriver()` 阻塞等待命令、`executeCommand()` 执行命令，直到连接断开才退出。

源码位置：`frameworks/native/libs/binder/IPCThreadState.cpp`

```cpp
void IPCThreadState::joinThreadPool(bool isMain) {
    mProcess->mCurrentThreads++;
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);  // 注册 looper
    mIsLooper = true;

    status_t result;
    do {
        processPendingDerefs();
        result = getAndExecuteCommand();       // 取命令并执行
        if (result == TIMED_OUT && !isMain) {
            break;                             // 非主线程超时可退出
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    mOut.writeInt32(BC_EXIT_LOOPER);           // 退出前通知 driver
    mIsLooper = false;
    talkWithDriver(false);
    mProcess->mCurrentThreads.fetch_sub(1);
}
```

两种 looper 注册命令的区别：

| 命令 | 谁发的 | 含义 |
| --- | --- | --- |
| `BC_ENTER_LOOPER` | `startThreadPool()` 主动创建的主线程（`isMain=true`），或主动调用 `joinThreadPool()` 的线程 | “我主动成为 Binder looper” |
| `BC_REGISTER_LOOPER` | `BR_SPAWN_LOOPER` 触发创建的 lazy 线程（`isMain=false`） | “我是应 driver 请求创建的新 looper，来报到” |

主循环的核心是 `getAndExecuteCommand()`，它内部是：

```text
getAndExecuteCommand()
  -> talkWithDriver()     // 阻塞在 ioctl 上，等待 driver 返回 BR_* 命令
  -> executeCommand()     // 执行返回的命令
```

Binder 服务线程的运行状态可以简化为：先注册成 looper，然后不断从 driver 读取 `BR_*` 命令并执行。

---

## 四、用户态和 driver 的通信通道：BINDER_WRITE_READ

一句话结论：用户态和 kernel 之间通过单一 ioctl `BINDER_WRITE_READ` 双向交换命令——`write_buffer` 放用户态发给 driver 的 `BC_*`，`read_buffer` 放 driver 返回给用户态的 `BR_*`，`talkWithDriver()` 就是这个通道的封装。

`talkWithDriver()` 构造 `binder_write_read` 结构体并发起 ioctl：

```cpp
status_t IPCThreadState::talkWithDriver(bool doReceive) {
    binder_write_read bwr;

    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();     // 用户态 -> driver 的 BC_* 命令

    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();   // driver -> 用户态的 BR_* 命令
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr);
}
```

这里有三个细节值得停一下：

1. **一次 ioctl 可以既写又读。** `mOut` 里的 `BC_*` 命令会通过 `write_buffer` 提交给 driver，driver 返回的 `BR_*` 命令会被写入 `read_buffer`。
2. **`joinThreadPool()` 的阻塞点就在这里。** 当 `doReceive=true` 且当前没有可读命令时，线程会睡在 `BINDER_WRITE_READ` 对应的内核等待路径上。
3. **`BR_SPAWN_LOOPER` 也是从这里回来的。** 扩容不是通过某个单独通知通道发生的，而是 driver 在 read buffer 里返回一条普通的 `BR_*` 命令。

关键命令一览：

| 命令 | 方向 | 含义 |
| --- | --- | --- |
| `BC_ENTER_LOOPER` | 用户态 -> driver | 当前线程主动成为 Binder looper |
| `BC_REGISTER_LOOPER` | 用户态 -> driver | 新 lazy thread 响应扩容请求并注册 |
| `BC_TRANSACTION` | 用户态 -> driver | 发起一次 Binder 调用 |
| `BC_REPLY` | 用户态 -> driver | 返回同步调用结果 |
| `BC_EXIT_LOOPER` | 用户态 -> driver | 当前线程退出 Binder looper |
| `BR_TRANSACTION` | driver -> 用户态 | 有 Binder 调用需要当前线程执行 |
| `BR_REPLY` | driver -> 用户态 | 当前线程等待的同步调用结果返回 |
| `BR_SPAWN_LOOPER` | driver -> 用户态 | 请求用户态创建新 Binder 线程 |

Binder looper 本质是睡在 ioctl 上的服务线程——没有请求时阻塞在内核等待队列中，不消耗 CPU；有请求时 driver 直接把 `BR_TRANSACTION` 写回 read buffer 并唤醒线程。

---

## 五、driver 侧如何管理进程和线程

一句话结论：driver 为每个打开 `/dev/binder` 的进程维护一个 `binder_proc`，为每个调用过 binder ioctl 的线程维护一个 `binder_thread`。`binder_thread` 不是内核线程，而是用户态真实 `pthread` 在 driver 中的影子状态。

源码位置：`kernel/common/drivers/android/binder_internal.h`

`binder_proc`——进程维度的调度台账：

```c
struct binder_proc {
    struct rb_root threads;              // 该进程所有 binder_thread（红黑树）
    struct rb_root nodes;                // 本地 Binder 实体
    struct rb_root refs_by_desc;         // 远端 Binder 引用（按 handle 查找）
    struct list_head waiting_threads;    // 空闲可接活的 looper 线程链表
    struct list_head todo;               // 进程级工作队列
    u32 max_threads;                     // lazy 线程上限（对应 BINDER_SET_MAX_THREADS）
    int requested_threads;               // 已请求但未注册的线程数
    int requested_threads_started;       // 已按请求启动并注册的 lazy 线程数
    struct binder_alloc alloc;           // 事务缓冲区分配器
    spinlock_t inner_lock, outer_lock;
};
```

`binder_thread`——线程维度的状态卡：

```c
struct binder_thread {
    struct binder_proc *proc;            // 所属进程
    struct list_head waiting_thread_node;// 挂入 proc->waiting_threads 的节点
    int pid;                             // 线程 tid
    int looper;                          // looper 状态：ENTERED/REGISTERED/WAITING/EXITED
    struct binder_transaction *transaction_stack;  // 同步调用栈（处理嵌套和 reply 投递）
    struct list_head todo;               // 线程私有工作队列
    bool process_todo;
    wait_queue_head_t wait;              // 线程睡眠等待 work 的等待队列
};
```

关键字段速查：

| 字段 | 所在结构 | 作用 |
| --- | --- | --- |
| `waiting_threads` | `binder_proc` | 当前空闲、可接新事务的 looper 列表，也是 driver 选择服务线程的依据 |
| `todo`（proc） | `binder_proc` | 进程级工作队列，没有选中具体线程时 work 先放这里 |
| `todo`（thread） | `binder_thread` | 线程私有工作队列，必须由该线程处理 |
| `max_threads` | `binder_proc` | lazy 线程上限 |
| `requested_threads_started` | `binder_proc` | 已创建的 lazy 线程数 |
| `transaction_stack` | `binder_thread` | 同步调用栈，嵌套 Binder 调用和 reply 回投都靠它 |
| `looper` | `binder_thread` | 当前线程的 looper 状态位 |
| `wait` | `binder_thread` | 线程阻塞等待 work 的 wait queue |

---

## 六、线程没任务时如何等待

一句话结论：Binder looper 没有 work 时进入 `binder_wait_for_work()`，可处理进程级 work 的线程先把自己挂入 `waiting_threads`，然后 `schedule()` 睡眠；有 work 时被唤醒，从链表中摘除，继续取任务。

源码位置：`kernel/common/drivers/android/binder.c`

```c
static int binder_wait_for_work(struct binder_thread *thread, bool do_proc_work) {
    DEFINE_WAIT(wait);
    struct binder_proc *proc = thread->proc;
    int ret = 0;

    binder_inner_proc_lock(proc);
    for (;;) {
        prepare_to_wait(&thread->wait, &wait, TASK_INTERRUPTIBLE|TASK_FREEZABLE);
        if (binder_has_work_ilocked(thread, do_proc_work))
            break;
        if (do_proc_work)
            list_add(&thread->waiting_thread_node, &proc->waiting_threads);  // 入空闲队列
        binder_inner_proc_unlock(proc);
        schedule();                                                          // 睡眠
        binder_inner_proc_lock(proc);
        list_del_init(&thread->waiting_thread_node);                         // 被唤醒后出队
        if (signal_pending(current)) {
            ret = -EINTR;
            break;
        }
    }
    finish_wait(&thread->wait, &wait);
    binder_inner_proc_unlock(proc);
    return ret;
}
```

等待/唤醒流程：

```text
looper 无 work
  -> 判断是否可处理 proc work（transaction_stack 为空 + thread.todo 为空）
  -> 是：把自己挂入 proc->waiting_threads
  -> prepare_to_wait() + schedule() 进入睡眠

被唤醒后
  -> 从 waiting_threads 摘除自己
  -> 检查 signal
  -> 继续 binder_thread_read() 取 work
```

`waiting_threads` 就是 driver 眼里的“当前可用服务线程列表”。新 transaction 到来时，driver 优先从这里取线程。

小结一下：Binder 线程空闲时不是在用户态轮询，也不是 libbinder 自己维护一个阻塞队列，而是把“我现在可以接活”这个状态登记到 `proc->waiting_threads`，然后真正睡进 kernel。这个设计让 driver 可以在事务到来时直接选择并唤醒目标线程。

---

## 七、事务如何分发到线程

一句话结论：新事务到来时，driver 先从 `waiting_threads` 取一个空闲线程，work 入该线程的 `thread.todo` 并唤醒；没有空闲线程时，work 入 `proc.todo` 等待后续 looper 取走。

driver 处理 `BC_TRANSACTION` 的链路：

```text
binder_thread_write()
  -> binder_transaction()              // 找到 target node/proc，分配 buffer，拷贝数据
  -> binder_proc_transaction()         // 选择目标线程并投递 work
```

`binder_proc_transaction()` 中的关键分发逻辑：

```c
if (!thread && !pending_async)
    thread = binder_select_thread_ilocked(proc);    // 从 waiting_threads 取一个空闲线程

if (thread) {
    binder_transaction_priority(thread, t, node);
    binder_enqueue_thread_work_ilocked(thread, &t->work);   // 入 thread.todo，唤醒
} else if (!pending_async) {
    binder_enqueue_work_ilocked(&t->work, &proc->todo);     // 无空闲线程，入 proc.todo
}
```

`binder_select_thread_ilocked()` 很简单——取链表头部：

```c
thread = list_first_entry_or_null(&proc->waiting_threads,
                                  struct binder_thread, waiting_thread_node);
if (thread)
    list_del_init(&thread->waiting_thread_node);
```

分发规则：

```text
waiting_threads 有空闲线程？
  ├─ 是 -> 取出一个线程 -> work 入 thread.todo -> 唤醒该线程
  └─ 否 -> work 入 proc.todo -> 等某个可处理 proc work 的 looper 来取
```

两个队列的区别：

| 队列 | 位置 | 谁能处理 | 什么情况入队 |
| --- | --- | --- | --- |
| `thread.todo` | `binder_thread` | 只能由该线程处理 | 选中空闲线程后的普通同步事务；必须投递给特定线程的 reply/错误通知 |
| `proc.todo` | `binder_proc` | 任意可处理 proc work 的 looper | 没有空闲线程时新事务暂存；异步事务等 |

注意：`thread.todo` 不只是承载“必须投递给特定线程”的 work。driver 从 `waiting_threads` 选中某个空闲线程后，普通同步 transaction 也会入该线程的 `thread.todo`。

---

## 八、服务端如何执行 onTransact

一句话结论：被唤醒的 Binder looper 从 `binder_thread_read()` 读到 `BR_TRANSACTION`，`executeCommand()` 将 driver buffer 包装成 `Parcel`，设置调用方 pid/uid，然后进入 `BBinder::transact()` / Java Stub 的 `onTransact()`。

用户态收到 `BR_TRANSACTION` 后的处理：

```cpp
// IPCThreadState::executeCommand()
case BR_TRANSACTION_SEC_CTX:
case BR_TRANSACTION: {
    binder_transaction_data_secctx tr_secctx;
    binder_transaction_data& tr = tr_secctx.transaction_data;
    result = mIn.read(&tr, sizeof(tr));

    Parcel buffer;
    buffer.ipcSetDataReference(
        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
        tr.data_size,
        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
        tr.offsets_size / sizeof(binder_size_t), freeBuffer);

    mCallingPid = tr.sender_pid;
    mCallingUid = tr.sender_euid;
    mLastTransactionBinderFlags = tr.flags;

    // 后续进入 BBinder::transact() -> Java Stub.onTransact()
    status_t error = mCallingContext.call(...);
}
```

这一步做了 4 件事：

1. 从 read buffer 读出 `binder_transaction_data`。
2. 把 driver 分配的事务 buffer 包装成 `Parcel` 对象。
3. 设置调用方身份（`mCallingPid` / `mCallingUid`）和 binder flags。
4. 进入本地 Binder 对象的 `transact()`，Java 层最终调到 Stub 的 `onTransact()`。

执行完成后，服务端写入 `BC_REPLY`，driver 将 reply 投递给正在等待的 client 线程，client 读到 `BR_REPLY` 后同步调用返回。

一次同步 Binder 调用的完整闭环：

```text
Client 线程                     Driver                      Server Binder 线程
─────────────                   ──────                      ─────────────────
BC_TRANSACTION ──►
                                binder_transaction()
                                binder_proc_transaction()
                                选线程 / 入 proc.todo
                                唤醒线程 ──────────────────►
                                                            BR_TRANSACTION
                                                            onTransact()
                                                            BC_REPLY ──►
                                投递 reply 到等待线程
◄── BR_REPLY
transact() 返回
```

---

## 九、线程池如何扩容

一句话结论：扩容不是用户态轮询触发，而是 driver 在 `binder_thread_read()` 末尾发现“没有空闲线程 + 没有已请求未注册的线程 + 未达 lazy 上限 + 当前线程是正常 looper”四个条件同时满足时，返回 `BR_SPAWN_LOOPER` 请求用户态创建新线程。

源码位置：`kernel/common/drivers/android/binder.c`，`binder_thread_read()` 末尾：

```c
if (proc->requested_threads == 0 &&                            // 没有已请求未注册的线程
    list_empty(&thread->proc->waiting_threads) &&              // 没有空闲 looper
    proc->requested_threads_started < proc->max_threads &&     // 未达 lazy 上限
    (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
     BINDER_LOOPER_STATE_ENTERED))) {                          // 当前线程是正常 looper
    proc->requested_threads++;
    binder_inner_proc_unlock(proc);
    if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer)) // 向用户态写入 BR_SPAWN_LOOPER
        return -EFAULT;
    binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
}
```

四个条件缺一不可：

| 条件 | 含义 | 为什么需要 |
| --- | --- | --- |
| `requested_threads == 0` | 没有已经请求但还没来注册的线程 | 避免重复请求，同一时刻最多 1 个待注册线程 |
| `waiting_threads` 为空 | 当前没有空闲 looper | 有空闲线程就不需要扩 |
| `requested_threads_started < max_threads` | lazy 线程数未达上限 | 受 `BINDER_SET_MAX_THREADS` 限制 |
| 当前 looper 状态为 `REGISTERED` / `ENTERED` | 当前线程自己是正常 looper | 非 looper 线程（如正在发起调用的 client 线程）不应触发扩容 |

用户态收到 `BR_SPAWN_LOOPER` 后：

```cpp
// IPCThreadState::executeCommand()
case BR_SPAWN_LOOPER:
    mProcess->spawnPooledThread(false);   // isMain=false，创建 lazy thread
    break;
```

新线程进入 `joinThreadPool(false)`，向 driver 写入 `BC_REGISTER_LOOPER` 完成注册：

```c
// kernel 处理 BC_REGISTER_LOOPER
case BC_REGISTER_LOOPER:
    if (proc->requested_threads == 0) {
        thread->looper |= BINDER_LOOPER_STATE_INVALID;   // 无请求却来注册 -> 异常
    } else {
        proc->requested_threads--;
        proc->requested_threads_started++;               // 计入已启动 lazy 线程
    }
    thread->looper |= BINDER_LOOPER_STATE_REGISTERED;
    break;
```

扩容闭环：

```text
没有空闲 waiting thread
  -> driver 返回 BR_SPAWN_LOOPER 给某 looper
  -> 该 looper 调用 spawnPooledThread(false)
  -> 新 pthread 启动，进入 joinThreadPool(false)
  -> 新线程发送 BC_REGISTER_LOOPER
  -> driver: requested_threads--, requested_threads_started++
  -> 新线程成为正式 Binder looper，可接活
```

kernel 不创建用户态线程，它只发出扩容请求；真正的 `pthread` 创建始终由 libbinder 完成。

---

## 十、最大线程数到底是多少？

一句话结论：15 和 31 是 driver 可请求创建的 lazy 线程上限，不是进程 Binder 线程总数硬上限。

| 进程 | lazy 上限（max_threads） | 常规最大总数口径 |
| --- | --- | --- |
| 普通进程默认 | 15 | 1 + 15 = 16 |
| system_server | 31 | 1 + 31 = 32 |

```cpp
// ProcessState.cpp
#define DEFAULT_MAX_BINDER_THREADS 15
```

```java
// SystemServer.java
private static final int sMaxBinderThreads = 31;
BinderInternal.setMaxThreads(sMaxBinderThreads);
```

总数口径来自 `ProcessState::getThreadPoolMaxTotalThreadCount()` 的逻辑：

```text
1 个 startThreadPool() 主动创建的主 PoolThread
+ max_threads 个 driver 可请求创建的 lazy PoolThread
+ 进程自己手动 joinThreadPool() 加入的线程（额外增加）
= 实际 Binder looper 总数
```

| 线程类型 | 是否计入 max_threads |
| --- | --- |
| `startThreadPool()` 创建的 `isMain=true` 主 `PoolThread` | 不计入（主动创建） |
| `BR_SPAWN_LOOPER` 触发创建的 lazy `PoolThread` | 计入 |
| 手动 `IPCThreadState::joinThreadPool()` 的线程 | 不计入（进程自己加的） |

所以不要简单说“Binder 线程池最大 15 个”或“system_server 最大 31 个”。更准确的说法是：lazy 上限 15/31，常规总数口径 16/32，手动加入线程另算。

---

## 十一、oneway 和同步调用的区别

一句话结论：`oneway`（`TF_ONE_WAY`）不让 client 等 reply，不阻塞调用线程，但 server 仍然需要 Binder 线程执行，且受 async buffer、frozen process、spam detection 等约束，不是“免费调用”。

| 特性 | 同步调用 | oneway 调用 |
| --- | --- | --- |
| client 是否阻塞 | 是，等 `BR_REPLY` | 否，发出即返回 |
| driver 是否维护 `transaction_stack` | 是（reply 要回到正确线程） | 否（无 reply） |
| 是否有 `BC_REPLY` / `BR_REPLY` | 有 | 无 |
| 入队方式 | 选中线程 -> `thread.todo`；无线程 -> `proc.todo` | 走 async 队列，受 target node async buffer 限制 |
| 服务端是否消耗 Binder 线程 | 是 | 是 |
| 特殊约束 | 反向同步调用可能死锁 | frozen process、oneway spam detection、async buffer 满会阻塞或报错 |

`oneway` 只是让 client 不等待结果，并不等于“丢给系统就不管了”——它仍然占用目标进程的 Binder 线程和 buffer 资源。实践里尤其要注意三类问题：

1. **async buffer 是有限资源。** 大量 oneway 会占用目标进程的异步事务缓冲，堆积到一定程度后仍然可能影响调用成功率。
2. **frozen process 有额外约束。** 目标进程被冻结时，异步事务不能简单按“立即执行”理解，driver 会结合冻结状态做拦截、排队或失败处理。
3. **oneway spam detection 会识别异常发送方。** Android 新版本通过 `BINDER_ENABLE_ONEWAY_SPAM_DETECTION` 开启检测，异常 oneway 洪泛可能触发 `BR_ONEWAY_SPAM_SUSPECT` 等诊断信号。

---

## 十二、调试 Binder 线程池问题看什么

遇到 Binder 卡顿、system_server Binder 线程耗尽、服务调用超时，按以下方向排查：

| 排查方向 | 看什么 | 说明 |
| --- | --- | --- |
| 服务端线程在干什么 | Binder 线程堆栈 | 是否都卡在耗时 `onTransact()`、锁等待、IO 或反向同步调用 |
| 是否长期无线程空闲 | `waiting_threads` 是否长期为空 | 为空说明请求在 `proc.todo` 排队或持续触发扩容 |
| lazy 线程是否打满 | `requested_threads_started` vs `max_threads` | 接近上限说明线程池被打满 |
| 同步调用是否过深 | `transaction_stack` | 深层嵌套 Binder 调用容易形成等待环或死锁 |
| oneway 是否堆积 | async buffer 占用、`BR_ONEWAY_SPAM_SUSPECT` | oneway 不阻塞 client，但会耗尽服务端处理能力 |
| 是否反向调用死锁 | `A -> B -> A` 模式 | 同步 Binder 重入可能形成死锁 |
| 线程数配置 | 是否调用了 `BinderInternal.setMaxThreads` | system_server 设为 31，其他进程默认 15 |

libbinder 创建的 pooled thread 线程名格式为 `binder:<pid>_<seq>`，通过线程名和堆栈可以判断它是在 `talkWithDriver()` 等待请求，还是正在执行某个 `onTransact()`。

常用抓栈方式：

```bash
# Java 进程可先抓 ANR trace
kill -3 <pid>

# Native / system 进程可直接抓所有线程 backtrace
debuggerd -b <pid>
```

Binder driver 的调试节点路径和 Android 版本、内核配置、binderfs 挂载方式有关，常见位置包括：

```bash
/proc/binder/proc/<pid>
/sys/kernel/debug/binder/proc/<pid>
/dev/binderfs/binder_logs/proc/<pid>
```

---

## 十三、推荐源码阅读顺序

不要从最复杂的 `binder_transaction()`（事务拷贝、偏移处理、`flat_binder_object` 解析）开始读，建议按线程池生命周期递进：

1. `ProcessState::open_driver()` —— 进程如何打开 `/dev/binder`、设置 `BINDER_SET_MAX_THREADS`
2. `ProcessState::startThreadPool()` / `spawnPooledThread()` —— 第一个 Binder 线程如何创建
3. `IPCThreadState::joinThreadPool()` —— 线程如何注册为 looper、主循环如何运转
4. `IPCThreadState::talkWithDriver()` —— `BINDER_WRITE_READ` 如何双向收发命令
5. `binder_wait_for_work()`（kernel）—— 线程如何睡眠、如何进入 `waiting_threads`
6. `binder_proc_transaction()` / `binder_select_thread_ilocked()`（kernel）—— 事务如何分发到 `thread.todo` 或 `proc.todo`
7. `BR_SPAWN_LOOPER` 处理链路（kernel + userspace）—— 扩容如何从 driver 请求回到用户态创建线程
8. 再读 `binder_transaction()`——理解事务 buffer 分配、`flat_binder_object` 处理、引用计数等细节

这条路径从结构到细节，先建立心智模型，再填具体实现。

---

## 结语：先建立边界，再读源码

理解 Binder 线程池最关键的一步，是先划清用户态和内核态的边界：

```text
用户态负责创建和执行线程；
driver 负责记录状态、排队、唤醒和请求扩容；
BC_* / BR_* 是两者协作的语言。
```

很多复杂系统都是这样——不要一头扎进实现细节，先搞清楚谁拥有状态、谁执行动作、谁发出信号、谁承担边界。一旦这个框架建立起来，源码就不再是迷宫，而是一张可以不断验证和修正的地图。

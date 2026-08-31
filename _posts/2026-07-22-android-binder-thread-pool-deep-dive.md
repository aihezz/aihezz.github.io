---
title: 深入拆解 Binder 线程池：从用户态线程到内核调度
date: 2026-07-22 10:00:00 +0800
categories: [Android]
tags: [Android, Binder, 源码分析, 进程通信]
---

## 零、从一个问题开始

你写下 `mRemoteService.doWork()`，跨进程调用就发生了。但服务端进程里，到底是**谁**在执行 `onTransact`？那些叫 `binder:1234_2`、`binder:1234_3` 的线程从哪来？谁决定它们的数量？没活干的时候它们在做什么？

这些问题的答案，都藏在「Binder 线程池」里。可它的源码偏偏最容易越读越晕——一上来就是 `ProcessState`、`IPCThreadState`、`binder_proc`、`binder_thread` 一堆结构体，函数调函数，读完仍然回答不了最朴素的问题。

原因是：**Binder 线程池横跨用户态和内核态两个世界，孤立地读任何一边都拼不出全貌。** 所以本文换一种读法——先建立一张跨越两界的地图，再用一次真实调用把它跑通，最后沿着「一个 Binder 线程的一生」把源码逐段验证。

先把最终结论放这儿，读完全文你会反复回到它：

> **Binder 线程池不是「用户态队列 + worker 线程」那种普通模型。它是一种协作模型：用户态负责创建、运行线程，线程平时睡在内核里；内核 driver 负责记录状态、排队事务、唤醒线程，并在不够用时反过来「请求」用户态扩容。**

---

## 一、先建立地图：两个半场，一条边界

理解线程池，第一步不是读代码，而是划清一条边界——**谁拥有线程，谁调度线程**。

| 关注点 | 归属 | 为什么在这一层 |
| --- | --- | --- |
| 创建、运行真实线程（`pthread`） | 用户态 libbinder | 线程是进程资源，内核不该替进程创建 |
| Binder 对象语义、方法分发、Parcel 序列化 | 用户态框架层 | 贴近业务，易扩展 |
| 事务缓冲、引用计数、死亡通知 | 内核 driver | 需要跨进程一致性 |
| 等待队列、线程唤醒、事务分发 | 内核 driver | 涉及调度，放内核避免用户态忙轮询 |
| 线程够不够、要不要扩容 | 内核 driver | 只有 driver 全局可见「有没有空闲线程」 |

一句话：**用户态出线程，内核出调度。** 普通线程池是「用户态队列 + worker 主动取任务」；Binder 线程池是「worker 睡在内核上，内核直接把活塞给它并唤醒」。正因如此，扩容信号只能由内核**主动下发**，用户态无从自己判断。

带着这条边界，看这张全景图——后面所有源码，都只是在把它落地。图中把线程池启动、Looper 待命、同步事务与 Reply、按需扩容拆成四个独立阶段；点击图片可以打开 SVG 原图无损放大。

[![Binder 线程池全景原理图](/assets/img/posts/2026-07-22-android-binder-thread-pool-deep-dive/binder-thread-pool-panorama.svg)](/assets/img/posts/2026-07-22-android-binder-thread-pool-deep-dive/binder-thread-pool-panorama.svg)

图里的 6 个角色，各司其职：

| 层 | 角色 | 一句话职责 |
| --- | --- | --- |
| 用户态 | `ProcessState` | 进程级单例：连 driver、设线程上限、创建 `PoolThread` |
| 用户态 | `IPCThreadState` | 线程本地单例：和 driver 收发命令、执行 `BR_*` |
| 用户态 | `PoolThread` | libbinder 创建的真实 `pthread`，线程体就是进 looper 循环 |
| 内核态 | `binder_proc` | 进程的调度台账：线程表、空闲队列、待办队列、线程计数 |
| 内核态 | `binder_thread` | 用户态线程在内核里的「影子」，记录 looper 状态和私有待办 |
| 内核态 | `binder_transaction` | 一次跨进程调用，真正在队列里排队、被分发的工作单元 |

四个总纲问题，先记住答案，后面逐一验证：

| 问题 | 答案 |
| --- | --- |
| 谁创建线程？ | 用户态 libbinder，最终落到 `pthread_create` |
| 线程平时在干嘛？ | 睡在内核 `waiting_threads` 里，阻塞在 `ioctl` 上 |
| 谁把请求分给哪个线程？ | 内核 driver，从空闲队列挑一个唤醒 |
| 谁决定扩容？ | 内核 driver，不够用时返回 `BR_SPAWN_LOOPER` |

---

## 二、先跑通一次调用：把地图变成电影

结构图是静态的，现在让它动起来。假设你的 App（client）已经拿到远端服务的代理，发起一次**同步**调用 `service.doWork()`，服务端进程此刻恰好有空闲的 Binder 线程。整个过程是这样的：

```text
Client 线程                     Binder Driver                Server Binder 线程
─────────────                   ─────────────                ──────────────────
doWork()
  └► BC_TRANSACTION ─────────►
                                ① binder_transaction()
                                   找到目标进程，分配 buffer，拷贝参数
                                ② binder_proc_transaction()
                                   从 waiting_threads 挑一个空闲线程
                                ③ 把 work 塞进该线程的 todo，唤醒它 ──►
                                                            ④ 从睡眠中醒来
                                                               读到 BR_TRANSACTION
                                                            ⑤ 包装成 Parcel
                                                               onTransact() 真正干活
                                                            ⑥ BC_REPLY ──►
                                ⑦ 把 reply 投给等待的 client 线程
  ◄──────── BR_REPLY ───────────
doWork() 返回
```

七步里，藏着这篇文章要讲清的全部机制：

- **① 事务从哪来** → client 线程通过 `BC_TRANSACTION` 把调用交给 driver（第四章：通信通道）；
- **② 挑哪个线程** → driver 从 `waiting_threads` 选空闲线程（第五、七章：调度台账与分发）；
- **③④ 线程怎么醒的** → 它本来就睡在内核等待队列上（第六章：待命）；
- **⑤ 谁执行 onTransact** → 被唤醒的那个 Binder looper（第八章：执行）；
- **如果第②步没有空闲线程呢** → 事务进 `proc.todo` 排队，同时可能触发 `BR_SPAWN_LOOPER` 扩容（第九章）。

而这些线程本身是怎么来的——第一个由进程启动时创建（第三章），后续由内核按需请求扩容（第九章）。下面就沿着「一个 Binder 线程的一生」，把每一步落到源码。

---

## 三、诞生：进程启动时的第一个线程

一句话结论：普通 App 进程在 zygote fork 出来后的 native 初始化阶段，就自动启动了 Binder 线程池——连上 driver、设好上限、映射缓冲区，再主动创建**第一个**线程。

App 进程不需要自己写任何代码，这一切发生在 `ZygoteInit` 的启动链路里：

```java
// frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
public static Runnable zygoteInit(...) {
    ...
    ZygoteInit.nativeZygoteInit();   // 经 JNI 进入 native
    return RuntimeInit.applicationInit(...);
}
```

```cpp
// frameworks/base/cmds/app_process/app_main.cpp
virtual void onZygoteInit() {
    sp<ProcessState> proc = ProcessState::self();  // 第一步：接入 driver
    proc->startThreadPool();                       // 第二步：创建第一个线程
}
```

（Native service 则是自己显式调 `startThreadPool()` + `joinThreadPool()`，通常让主线程也当 looper。原理一致，不再单列。）

**第一步：接入 driver。** `ProcessState::self()` 首次调用时构造单例，做三件事——打开设备、校验版本、设置进程级参数，最后 `mmap` 出事务缓冲区：

```cpp
// frameworks/native/libs/binder/ProcessState.cpp
#define DEFAULT_MAX_BINDER_THREADS 15                             // lazy 线程上限默认值
#define BINDER_VM_SIZE ((1*1024*1024) - sysconf(_SC_PAGE_SIZE)*2) // 事务缓冲区 ~1MB - 2页

static unique_fd open_driver(const char* driver, String8* error) {
    auto fd = unique_fd(open(driver, O_RDWR | O_CLOEXEC));       // 打开 /dev/binder
    ioctl(fd.get(), BINDER_VERSION, &vers);                     // 校验协议版本
    size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
    ioctl(fd.get(), BINDER_SET_MAX_THREADS, &maxThreads);       // 告诉 driver：lazy 上限 15
    uint32_t enable = DEFAULT_ENABLE_ONEWAY_SPAM_DETECTION;
    ioctl(fd.get(), BINDER_ENABLE_ONEWAY_SPAM_DETECTION, &enable);
    return fd;
}
```

注意这一步**还没有任何 Binder 线程**——它只是「把进程接入 driver」。真正的服务线程要等下一步。

**第二步：创建第一个线程。** `startThreadPool()` 只做一件事：创建 1 个 `isMain=true` 的主 `PoolThread`：

```cpp
void ProcessState::startThreadPool() {
    std::unique_lock<std::mutex> _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);    // isMain=true
    }
}

void ProcessState::spawnPooledThread(bool isMain) {
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();   // 形如 binder:<pid>_<seq>
        sp<Thread> t = sp<PoolThread>::make(isMain);
        t->run(name.c_str());                    // 这里才真正 pthread_create
        mKernelStartedThreads++;
    }
}
```

`PoolThread` 的线程体只有一行——一头扎进 looper 主循环，从此这个线程的余生都在里面（下一章）：

```cpp
class PoolThread : public Thread {
    virtual bool threadLoop() {
        IPCThreadState::self()->joinThreadPool(mIsMain);  // 进入 Binder looper 主循环
        return false;
    }
};
```

至此，「第一个 Binder 线程」的完整出生链路：

```text
ZygoteInit.zygoteInit()
  └► nativeZygoteInit() → onZygoteInit()
       └► ProcessState::self()          // open + BINDER_SET_MAX_THREADS + mmap
       └► startThreadPool()
            └► spawnPooledThread(true)         // pthread_create
                 └► joinThreadPool(true)       // 进入主循环，见下一章
```

**记住一个关键点：** 进程启动只创建这**一个**线程。后面即使并发请求暴涨，也不是预先铺一堆线程等着，而是内核发现忙不过来时才逐个「请求」扩容（第九章）。这正是 Binder 线程池「按需伸缩」的核心特征。

---

## 四、报到与通话：成为 looper，接通 BINDER_WRITE_READ

线程出生后进入 `joinThreadPool()`。它做的第一件事是**向 driver 报到**——声明「我是一个 Binder looper，可以接活了」，然后进入「等命令 → 执行命令」的无限循环：

```cpp
// frameworks/native/libs/binder/IPCThreadState.cpp
void IPCThreadState::joinThreadPool(bool isMain) {
    mProcess->mCurrentThreads++;
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);  // 报到
    mIsLooper = true;

    status_t result;
    do {
        processPendingDerefs();
        result = getAndExecuteCommand();       // 阻塞取命令 + 执行
        if (result == TIMED_OUT && !isMain) {
            break;                             // 非主线程超时可退出
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    mOut.writeInt32(BC_EXIT_LOOPER);           // 退场前跟 driver 打招呼
    mIsLooper = false;
    talkWithDriver(false);
    mProcess->mCurrentThreads.fetch_sub(1);
}
```

报到用两种命令，区别只在于这个 looper 是「自愿加入」还是「被内核请求来的」——这个区别在第九章的扩容里很关键：

| 报到命令 | 谁发的 | 潜台词 |
| --- | --- | --- |
| `BC_ENTER_LOOPER` | 主线程（`isMain=true`）或主动 `joinThreadPool()` 的线程 | 「我主动来当 looper」 |
| `BC_REGISTER_LOOPER` | 被 `BR_SPAWN_LOOPER` 请求创建的 lazy 线程（`isMain=false`） | 「我是应召扩容来报到的」 |

循环的核心 `getAndExecuteCommand()` 内部就两步：`talkWithDriver()`（阻塞等 `BR_*`）+ `executeCommand()`（执行）。而 `talkWithDriver()` 就是用户态和内核之间**唯一的通话通道**：

一句话结论：一切收发都走同一个 ioctl —— `BINDER_WRITE_READ`。`write_buffer` 装用户态发给 driver 的 `BC_*`，`read_buffer` 装 driver 返回的 `BR_*`，一次调用可同时读写。

```cpp
status_t IPCThreadState::talkWithDriver(bool doReceive) {
    binder_write_read bwr;

    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();     // 用户态 -> driver：BC_*

    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();   // driver -> 用户态：BR_*
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr);   // ← 线程就阻塞在这
}
```

关于这个通道，三点最容易被忽略：

1. **一次 ioctl 既写又读。** 发出去的 `BC_*` 和收回来的 `BR_*` 在同一次系统调用里完成。
2. **线程的「睡眠点」就在这行 `ioctl`。** 当没有可读命令时，线程不会返回用户态空转，而是直接睡在内核等待路径上（下一章）。
3. **扩容信号 `BR_SPAWN_LOOPER` 没有专用通道**，它就是 driver 在 `read_buffer` 里塞回来的一条普通 `BR_*`。

一张表记住这套「协作语言」：

| 命令 | 方向 | 含义 |
| --- | --- | --- |
| `BC_ENTER_LOOPER` / `BC_REGISTER_LOOPER` | 用户态 → driver | 报到成 looper（主动 / 应召） |
| `BC_TRANSACTION` / `BC_REPLY` | 用户态 → driver | 发起调用 / 返回同步结果 |
| `BC_EXIT_LOOPER` | 用户态 → driver | 退出 looper |
| `BR_TRANSACTION` | driver → 用户态 | 有活，你来执行 |
| `BR_REPLY` | driver → 用户态 | 你等的同步结果回来了 |
| `BR_SPAWN_LOOPER` | driver → 用户态 | 忙不过来了，再造一个线程 |

---

## 五、内核的账本：binder_proc 与 binder_thread

线程报到后，内核给它记了一笔账。要看懂调度，得先认识 driver 的两个核心结构——它们是前面全景图里「内核半场」的实体。

```c
// kernel/common/drivers/android/binder_internal.h

// 进程维度：driver 为每个打开 /dev/binder 的进程建一个
struct binder_proc {
    struct rb_root threads;              // 本进程所有 binder_thread
    struct list_head waiting_threads;    // ★ 空闲、可接活的 looper 队列
    struct list_head todo;               // ★ 进程级待办（没指定线程时的暂存）
    u32 max_threads;                     // lazy 线程上限（= BINDER_SET_MAX_THREADS）
    int requested_threads;               // 已请求、但还没报到的线程数
    int requested_threads_started;       // 已应召启动并报到的 lazy 线程数
    struct binder_alloc alloc;           // 事务缓冲区分配器
    spinlock_t inner_lock, outer_lock;
};

// 线程维度：driver 为每个调过 binder ioctl 的线程建一个——它是用户态 pthread 的「影子」
struct binder_thread {
    struct binder_proc *proc;                      // 所属进程
    struct list_head waiting_thread_node;          // 挂进 proc->waiting_threads 的节点
    int looper;                                    // 状态位：ENTERED/REGISTERED/WAITING/EXITED
    struct binder_transaction *transaction_stack;  // 同步调用栈（处理嵌套与 reply 回投）
    struct list_head todo;                         // ★ 线程私有待办（只有它能处理）
    wait_queue_head_t wait;                        // ★ 线程睡眠等待的队列
};
```

关键一点：**`binder_thread` 不是内核线程**，它只是用户态那个真实 `pthread` 在内核里的状态卡。真正跑代码的永远是用户态线程，内核只是拿这张卡记录「它现在是空闲还是在忙、它的待办里有什么」。

带 ★ 的五个字段，是后面两章调度的全部主角：

| 字段 | 位置 | 作用 |
| --- | --- | --- |
| `waiting_threads` | proc | 空闲 looper 列表，driver 选线程的唯一来源 |
| `todo`（proc / thread） | proc / thread | 进程公共待办 vs 线程私有待办 |
| `max_threads` / `requested_threads_started` | proc | lazy 上限 / 已创建的 lazy 线程数 |
| `transaction_stack` | thread | 同步调用栈，嵌套调用和 reply 回投靠它 |
| `wait` | thread | 线程睡在这个等待队列上 |

---

## 六、待命：没活干时，线程睡在哪

一句话结论：looper 没活时进入 `binder_wait_for_work()`——先把自己挂进 `waiting_threads`（宣告「我空闲可接活」），再 `schedule()` 真正睡进内核；有活时被唤醒、出队、继续干。

```c
// kernel/common/drivers/android/binder.c
static int binder_wait_for_work(struct binder_thread *thread, bool do_proc_work) {
    DEFINE_WAIT(wait);
    struct binder_proc *proc = thread->proc;
    int ret = 0;

    binder_inner_proc_lock(proc);
    for (;;) {
        prepare_to_wait(&thread->wait, &wait, TASK_INTERRUPTIBLE|TASK_FREEZABLE);
        if (binder_has_work_ilocked(thread, do_proc_work))
            break;                                                           // 有活，不睡
        if (do_proc_work)
            list_add(&thread->waiting_thread_node, &proc->waiting_threads);  // 宣告空闲
        binder_inner_proc_unlock(proc);
        schedule();                                                          // 睡
        binder_inner_proc_lock(proc);
        list_del_init(&thread->waiting_thread_node);                         // 醒来出队
        if (signal_pending(current)) { ret = -EINTR; break; }
    }
    finish_wait(&thread->wait, &wait);
    binder_inner_proc_unlock(proc);
    return ret;
}
```

这段是「协作模型」最精彩的一笔：Binder 线程空闲时**不在用户态轮询**，也不靠 libbinder 自己维护什么阻塞队列，而是把「我可以接活」这个状态**登记到内核的 `waiting_threads`**，然后彻底睡下。于是 `waiting_threads` 就成了 driver 眼里那份「当前可用服务线程」名单——事务一来，直接从名单里挑人唤醒，零轮询、零竞争。

---

## 七、干活（一）：事务如何被分到某个线程

回到第二章那次调用的 ②③ 步。client 发来 `BC_TRANSACTION` 后，driver 的分发路径是：

```text
binder_thread_write()
  └► binder_transaction()        // 找目标 node/proc、分配 buffer、拷贝数据
       └► binder_proc_transaction()   // 选线程、投递 work
```

核心决策就在 `binder_proc_transaction()` 里——**有空闲线程就点名派给它，没有就丢进进程公共待办**：

```c
if (!thread && !pending_async)
    thread = binder_select_thread_ilocked(proc);    // 从 waiting_threads 挑一个

if (thread) {
    binder_transaction_priority(thread, t, node);
    binder_enqueue_thread_work_ilocked(thread, &t->work);   // 进 thread.todo，唤醒它
} else if (!pending_async) {
    binder_enqueue_work_ilocked(&t->work, &proc->todo);     // 没空闲线程 → 进 proc.todo 排队
}
```

「挑一个」的逻辑简单到就是取链表头：

```c
thread = list_first_entry_or_null(&proc->waiting_threads,
                                  struct binder_thread, waiting_thread_node);
if (thread)
    list_del_init(&thread->waiting_thread_node);   // 选中即出队，不再算空闲
```

于是有了两个待办队列，分工明确：

| 队列 | 谁能处理 | 什么时候用它 |
| --- | --- | --- |
| `thread.todo` | 只有该线程 | 已点名给某空闲线程的普通同步事务；必须回到特定线程的 reply/错误 |
| `proc.todo` | 任意空闲 looper | 没有空闲线程时的暂存；异步事务等 |

一个常见误解要澄清：`thread.todo` **不只**装「必须给特定线程」的 work。只要 driver 从 `waiting_threads` 点了某个空闲线程，普通同步事务也会进它的 `thread.todo`——这就是第二章 ②③ 步发生的事。

---

## 八、干活（二）：执行 onTransact，闭合 reply

被唤醒的线程从内核读回一条 `BR_TRANSACTION`，接下来在用户态做四件事，然后调进业务代码：

```cpp
// IPCThreadState::executeCommand()
case BR_TRANSACTION_SEC_CTX:
case BR_TRANSACTION: {
    binder_transaction_data_secctx tr_secctx;
    binder_transaction_data& tr = tr_secctx.transaction_data;
    result = mIn.read(&tr, sizeof(tr));            // ① 读出事务描述

    Parcel buffer;
    buffer.ipcSetDataReference(                    // ② driver 的 buffer 包成 Parcel（零拷贝）
        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
        tr.data_size,
        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
        tr.offsets_size / sizeof(binder_size_t), freeBuffer);

    mCallingPid = tr.sender_pid;                   // ③ 记下调用方身份
    mCallingUid = tr.sender_euid;                  //    （getCallingPid/Uid 就靠它）
    mLastTransactionBinderFlags = tr.flags;

    // ④ 进入 BBinder::transact() → Java 层 Stub.onTransact()
    status_t error = mCallingContext.call(...);
}
```

`onTransact()` 执行完，服务端写 `BC_REPLY`，driver 沿着当初记下的 `transaction_stack` 把结果**精准投回**当初等待的那个 client 线程，client 读到 `BR_REPLY`，`doWork()` 返回。第二章那张时序图，到这里就完整闭合了。

> 顺带回答一个高频问题：`getCallingPid()`/`getCallingUid()` 为什么可信？因为调用方身份（③）是 driver 在内核里填进 `binder_transaction_data` 的，用户态伪造不了——这是 Binder 权限校验的基石。

---

## 九、扩容：线程池如何按需增援

现在回到第二章那个「如果没有空闲线程」的分支。事务进了 `proc.todo` 排队，但如果请求持续涌来、线程始终不够，光排队不是办法——这时**扩容**登场。

一句话结论：扩容不是用户态轮询判断的，而是 driver 在 `binder_thread_read()` 末尾，发现**四个条件同时成立**时，主动返回 `BR_SPAWN_LOOPER` 请求用户态造线程。

```c
// kernel/common/drivers/android/binder.c，binder_thread_read() 末尾
if (proc->requested_threads == 0 &&                            // ① 没有在途的扩容请求
    list_empty(&thread->proc->waiting_threads) &&              // ② 当前没有空闲 looper
    proc->requested_threads_started < proc->max_threads &&     // ③ 没到 lazy 上限
    (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
     BINDER_LOOPER_STATE_ENTERED))) {                          // ④ 当前线程本身是正常 looper
    proc->requested_threads++;
    binder_inner_proc_unlock(proc);
    if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))  // 把扩容请求写回用户态
        return -EFAULT;
    binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
}
```

四个条件，缺一不可：

| 条件 | 为什么需要 |
| --- | --- |
| ① `requested_threads == 0` | 同一时刻最多请求 1 个，避免一次性造出一堆 |
| ② `waiting_threads` 为空 | 还有空闲线程就没必要扩 |
| ③ `requested_threads_started < max_threads` | 受 `BINDER_SET_MAX_THREADS` 上限约束 |
| ④ 当前线程是 `REGISTERED`/`ENTERED` | 正在发起调用的 client 线程等「非 looper」不该触发扩容 |

用户态收到后造线程，新线程进 `joinThreadPool(false)` 并用 `BC_REGISTER_LOOPER` 报到（对应第四章的「应召」）：

```cpp
// IPCThreadState::executeCommand()
case BR_SPAWN_LOOPER:
    mProcess->spawnPooledThread(false);   // isMain=false，造一个 lazy 线程
    break;
```

内核收到 `BC_REGISTER_LOOPER` 时的处理，正好和 `BC_ENTER_LOOPER` 形成对照——它要**核对并更新计数**，因为这是一次「应召报到」：

```c
// kernel 处理 BC_REGISTER_LOOPER
case BC_REGISTER_LOOPER:
    if (proc->requested_threads == 0) {
        thread->looper |= BINDER_LOOPER_STATE_INVALID;   // 没请求你却来了 → 标记异常
    } else {
        proc->requested_threads--;                       // 在途请求减一
        proc->requested_threads_started++;               // 已启动 lazy 计数加一
    }
    thread->looper |= BINDER_LOOPER_STATE_REGISTERED;
    break;
```

（相比之下，`BC_ENTER_LOOPER` 是主线程主动加入，内核不碰这两个计数——这就是两种报到命令的本质差异。）

整个扩容闭环：

```text
proc.todo 有活但 waiting_threads 空
  └► driver 给「当前正在读的这个 looper」返回 BR_SPAWN_LOOPER
       └► 该 looper 调 spawnPooledThread(false)     // 用户态造 pthread
            └► 新线程 joinThreadPool(false) → BC_REGISTER_LOOPER 报到
                 └► driver: requested_threads--, requested_threads_started++
                      └► 新线程成为正式 looper，投入接活
```

**再强调一遍协作模型：内核从不亲手创建用户态线程，它只发「请求」；真正的 `pthread` 永远由 libbinder 创建。** 这就是为什么线程池能随负载弹性伸缩，又不会把创建线程的权力交给内核。

---

## 十、到底能有多少个 Binder 线程？

一句话结论：`15` 和 `31` 是 driver **可请求创建的 lazy 线程上限**，不是进程 Binder 线程的总数硬顶。

```cpp
// ProcessState.cpp —— 普通进程默认
#define DEFAULT_MAX_BINDER_THREADS 15
```

```java
// SystemServer.java —— system_server 抬高上限
private static final int sMaxBinderThreads = 31;
BinderInternal.setMaxThreads(sMaxBinderThreads);
```

「总数」要把主线程和手动加入的线程也算进去：

```text
   1 个主 PoolThread（startThreadPool 主动建，不计入 max_threads）
 + 最多 max_threads 个 lazy 线程（BR_SPAWN_LOOPER 触发，计入）
 + 进程手动 joinThreadPool() 加入的线程（不计入，另算）
 = 实际 Binder looper 总数
```

| 进程 | lazy 上限 `max_threads` | 常规总数口径 |
| --- | --- | --- |
| 普通进程默认 | 15 | 1 + 15 = **16** |
| system_server | 31 | 1 + 31 = **32** |

所以别再笼统说「Binder 线程池最大 15 个」。**准确的说法是：lazy 上限 15/31，常规总数口径 16/32，手动加入的线程另算。**

---

## 十一、oneway 不是「免费调用」

一句话结论：`oneway`（`TF_ONE_WAY`）只是让 client **不等 reply、不阻塞**，但服务端**照样要占一个 Binder 线程**去执行，而且还额外受 async buffer、frozen process、spam detection 三重约束。

| 维度 | 同步调用 | oneway 调用 |
| --- | --- | --- |
| client 是否阻塞 | 是，等 `BR_REPLY` | 否，发出即返回 |
| 维护 `transaction_stack` | 是（reply 要回原线程） | 否（没有 reply） |
| `BC_REPLY` / `BR_REPLY` | 有 | 无 |
| 入队方式 | 选中线程→`thread.todo`；否则→`proc.todo` | 走 async 队列，受目标 node 的 async buffer 限制 |
| 是否消耗服务端 Binder 线程 | 是 | **是** |
| 特殊风险 | 反向同步调用可能死锁 | async buffer 满、目标被冻结、被判定 spam |

三个实战中最容易踩的坑：

1. **async buffer 是有限资源。** 大量 oneway 会占满目标进程的异步事务缓冲，堆积到阈值后，后续 oneway 可能直接失败——「不阻塞」不等于「无限量」。
2. **frozen process 有额外拦截。** 目标进程被冻结（App 进入后台被 cgroup freeze）时，异步事务不能按「立即执行」理解，driver 会结合冻结状态做拦截、排队或失败处理。
3. **oneway spam detection 会盯异常发送方。** 新版本通过 `BINDER_ENABLE_ONEWAY_SPAM_DETECTION`（还记得第三章 `open_driver` 里那个 ioctl 吗）开启检测，oneway 洪泛会触发 `BR_ONEWAY_SPAM_SUSPECT` 等诊断信号。

---

## 十二、实战：线程池出问题时看什么

理解了机制，排查就有了抓手。遇到 Binder 卡顿、system_server 线程耗尽、服务调用超时，按下面的方向对号入座——每一条都能对应到前面某一章：

| 现象/方向 | 看什么 | 对应机制 |
| --- | --- | --- |
| 服务端线程都在忙什么 | Binder 线程堆栈 | 是否卡在耗时 `onTransact()`、锁、IO、反向调用（第八章） |
| 是否长期没有空闲线程 | `waiting_threads` 是否总是空 | 空 = 请求都在 `proc.todo` 排队或不停扩容（第六、七章） |
| lazy 线程是否打满 | `requested_threads_started` vs `max_threads` | 逼近上限 = 线程池被打爆（第九、十章） |
| 同步调用是否太深 | `transaction_stack` | 深层嵌套易成等待环/死锁（第五、八章） |
| oneway 是否堆积 | async buffer 占用、`BR_ONEWAY_SPAM_SUSPECT` | 不阻塞 client，但耗尽服务端线程（第十一章） |
| 线程上限配置 | 是否调过 `setMaxThreads` | system_server 31，其余默认 15（第十章） |

线程名 `binder:<pid>_<seq>` 配合堆栈，一眼能看出它是**睡在 `talkWithDriver()` 等活**，还是**正在跑某个 `onTransact()`**。常用抓栈：

```bash
kill -3 <pid>        # Java 进程：抓 ANR trace
debuggerd -b <pid>   # Native / system 进程：抓所有线程 backtrace
```

driver 侧的调试节点随 Android 版本、内核配置、binderfs 挂载方式而变，常见位置：

```bash
/proc/binder/proc/<pid>
/sys/kernel/debug/binder/proc/<pid>
/dev/binderfs/binder_logs/proc/<pid>
```

里面能直接看到该进程的 `waiting_threads`、`requested_threads`、每个线程的 todo 和事务栈——正是前面几章那些字段的运行时快照。

---

## 十三、想读源码，按这个顺序

别一上来就啃最复杂的 `binder_transaction()`（buffer 分配、偏移处理、`flat_binder_object` 解析全堆在里面）。按「线程池的一生」递进，先有心智模型再抠细节：

1. `ProcessState::open_driver()` —— 进程怎么连上 driver、设上限（第三章）
2. `startThreadPool()` / `spawnPooledThread()` —— 第一个线程怎么造（第三章）
3. `IPCThreadState::joinThreadPool()` —— 怎么报到、主循环怎么转（第四章）
4. `talkWithDriver()` —— `BINDER_WRITE_READ` 怎么双向收发（第四章）
5. `binder_wait_for_work()` —— 线程怎么睡、怎么进 `waiting_threads`（第六章）
6. `binder_proc_transaction()` / `binder_select_thread_ilocked()` —— 事务怎么分发（第七章）
7. `BR_SPAWN_LOOPER` 处理链路 —— 扩容怎么从内核回到用户态（第九章）
8. 最后再读 `binder_transaction()` —— buffer、`flat_binder_object`、引用计数等细节

---

## 结语：先划边界，源码就不再是迷宫

如果读完只能记住一句话，请记住这条边界：

```text
用户态：创建线程、执行线程；
内核态：记录状态、排队事务、唤醒线程、请求扩容；
BC_* / BR_*：两个世界之间唯一的通话语言。
```

Binder 线程池的所有「复杂」，本质都是这条边界的展开：线程在用户态出生、在内核里睡去、被内核唤醒、由内核请求增援。很多大型系统都是这个套路——**先搞清楚谁拥有状态、谁执行动作、谁发出信号，源码就从迷宫变成了一张能反复验证的地图。**

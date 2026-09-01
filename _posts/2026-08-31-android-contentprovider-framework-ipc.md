---
title: 深入理解 ContentProvider：从 call 调用到进程拉起与 Provider 生命周期
date: 2026-08-31 21:00:00 +0800
categories: [Android]
tags: [Android, ContentProvider, Binder, IPC, Framework, 源码分析]
lang: zh-CN
---

## 零、从一次 `ContentResolver.call()` 开始

下面这行代码看起来只是在调用一个由 Provider 自定义的方法：

```kotlin
val result = context.contentResolver.call(
    Uri.parse("content://com.example.account.provider"),
    "account.getLoginState.v1",
    null,
    Bundle().apply {
        putString("scene", "cold_start")
    }
)
```

但它实际可能走过完全不同的路径：

- Provider 已经安装在当前进程中：直接执行一次本地 Java 调用。
- Provider 与调用方同进程，但尚未安装：先向 AMS 查询，再由当前进程现场创建 Provider。
- Provider 位于其他进程，且 Binder 代理已经缓存：直接发起跨进程调用。
- Provider 位于其他进程，进程已经存在但 Provider 尚未发布：等待目标进程完成安装与发布。
- Provider 所在进程尚未启动：触发目标进程冷启动、Application 创建、Provider 安装与发布，最后才真正执行 `call()`。

这也是 ContentProvider 容易被低估的原因。它不只是一个 CRUD 接口，而是一套由 Android Framework 托管的、以 `authority` 为发现键的数据型 IPC 机制。

下面这张图把调用方进程、`system_server` 和目标 Provider 进程放在同一张时序图里。点击图片可以打开 SVG 原图无损放大。

[![ContentProvider call 全景流程](/assets/img/posts/2026-08-31-android-contentprovider-framework-ipc/contentprovider-call-flow.svg)](/assets/img/posts/2026-08-31-android-contentprovider-framework-ipc/contentprovider-call-flow.svg)

本文以 Android 30 的 AOSP 源码为基准。不同 Android 版本的方法签名、类拆分和 attribution 参数会有变化，但 Provider 获取、安装、发布与调用的主链路基本稳定。

---

## 一、先建立三方模型

理解 ContentProvider，第一步不是背 API，而是划清三个执行主体。

| 执行主体 | 核心对象 | 职责 |
| --- | --- | --- |
| 当前进程 | `ContentResolver`、`ActivityThread`、`ProviderClientRecord` | 发起调用、查询本地缓存、持有本地 Transport 或远端 Binder proxy |
| `system_server` | AMS、`ContentProviderRecord`、`ContentProviderConnection` | 按 authority 查找 Provider、检查调用资格、管理引用、拉起进程、等待发布 |
| 目标 Provider 进程 | `ActivityThread`、`ContentProvider.Transport`、业务 `ContentProvider` | 创建并安装 Provider、发布 Binder、执行真正的业务方法 |

它们之间由几个关键抽象串起来：

```text
content://authority
    └── Provider 的注册与发现键

ContentResolver
    └── 客户端统一入口

IContentProvider
    └── 同进程与跨进程共用的接口抽象

ContentProvider.Transport
    └── Provider 内部的 IContentProvider 服务端实现

AMS / ContentProviderHelper
    └── Provider 的发现、进程调度、发布等待和连接管理中心
```

调用方最终拿到的都是 `IContentProvider`。如果对象位于本进程，它实际指向 `ContentProvider.Transport`；如果对象位于其他进程，它实际是 `ContentProviderProxy`。因此上层 API 不需要为本地调用和远端调用分别设计两套接口。

---

## 二、`ContentResolver.call()` 做了什么

Android 30 中，`ContentResolver.call()` 的主干逻辑可以压缩为：

```java
public final Bundle call(
        String authority,
        String method,
        String arg,
        Bundle extras
) {
    IContentProvider provider = acquireProvider(authority);
    if (provider == null) {
        throw new IllegalArgumentException("Unknown authority " + authority);
    }

    try {
        return provider.call(
            mPackageName,
            mAttributionTag,
            authority,
            method,
            arg,
            extras
        );
    } finally {
        releaseProvider(provider);
    }
}
```

源码位置：

```text
frameworks/base/core/java/android/content/ContentResolver.java
```

这段代码揭示了三个事实：

1. `call()` 的常规实现通过 `acquireProvider()` 获取 stable 引用。
2. `provider.call()` 是否穿过 Binder，由 `provider` 的实际对象决定。
3. `finally` 中的 `releaseProvider()` 释放本次引用，但不等于立即销毁 Provider。

真正复杂的部分不是 `call()`，而是 `acquireProvider()` 如何拿到那个 `IContentProvider`。

---

## 三、`ActivityThread.acquireProvider()`：先查本地，再问 AMS

`ActivityThread.acquireProvider()` 首先调用 `acquireExistingProvider()`：

```java
public final IContentProvider acquireProvider(
        Context context,
        String authority,
        int userId,
        boolean stable
) {
    IContentProvider provider =
        acquireExistingProvider(context, authority, userId, stable);

    if (provider != null) {
        return provider;
    }

    ContentProviderHolder holder =
        ActivityManager.getService().getContentProvider(...);

    holder = installProvider(
        context, holder, holder.info,
        true, holder.noReleaseNeeded, stable
    );
    return holder.provider;
}
```

表面看，`acquireExistingProvider()` 只是一次 Map 查询。实际上它处在三套状态之间：

```text
authority + userId
  → ProviderClientRecord
  → IContentProvider / Binder identity
  → ProviderRefCount
  → ContentProviderConnection（system_server）
```

理解这组映射，才能看懂为什么缓存命中后还要增加引用、为什么一个 Provider 的多个 authority 共享同一份计数，以及为什么最后一份引用释放时不能立刻删缓存。

### 3.1 `ContentResolver` 如何把 stable/unstable 传给 `ActivityThread`

应用拿到的 `ContentResolver`，具体实现是 `ContextImpl.ApplicationContentResolver`。它本身不维护 Provider，而是把操作转交给当前进程的 `ActivityThread`：

```java
protected IContentProvider acquireProvider(Context context, String auth) {
    return mMainThread.acquireProvider(
        context,
        ContentProvider.getAuthorityWithoutUserId(auth),
        resolveUserIdFromAuthority(auth),
        true  // stable
    );
}

protected IContentProvider acquireUnstableProvider(Context context, String auth) {
    return mMainThread.acquireProvider(
        context,
        ContentProvider.getAuthorityWithoutUserId(auth),
        resolveUserIdFromAuthority(auth),
        false // unstable
    );
}
```

对应的释放操作也只是把同一个布尔值传回去：

```java
releaseProvider(provider)
    → ActivityThread.releaseProvider(provider, true)

releaseUnstableProvider(provider)
    → ActivityThread.releaseProvider(provider, false)
```

这说明 stable/unstable 不是两套 Provider，也不是两种 Binder 对象。它是一次 acquire/release 对同一连接施加的引用语义。

### 3.2 客户端为什么需要四张表

`ActivityThread` 中和 Provider 缓存直接相关的结构主要有四个：

```java
// authority + userId → ProviderClientRecord
ArrayMap<ProviderKey, ProviderClientRecord> mProviderMap;

// Binder identity → stable/unstable 引用状态
ArrayMap<IBinder, ProviderRefCount> mProviderRefCountMap;

// 仅本进程 Provider：Binder → ProviderClientRecord
ArrayMap<IBinder, ProviderClientRecord> mLocalProviders;

// 仅本进程 Provider：组件名 → ProviderClientRecord
ArrayMap<ComponentName, ProviderClientRecord> mLocalProvidersByName;
```

它们解决的是不同问题：

| 结构 | Key | 回答的问题 |
| --- | --- | --- |
| `mProviderMap` | `authority + userId` | 这个 URI 应该落到哪个 `IContentProvider`？ |
| `mProviderRefCountMap` | `IBinder` | 这条实际 Provider 连接还有多少 stable/unstable 引用？ |
| `mLocalProviders` | 本地 Binder | 这个 Binder 对应哪个本地 Provider 记录？ |
| `mLocalProvidersByName` | `ComponentName` | 这个 Provider 类是否已经在本进程实例化？ |

这里特意把“寻址”和“生命周期”分开了。一个 Provider 可以声明多个以分号分隔的 authority。`installProviderAuthoritiesLocked()` 会把这些 authority 分别写入 `mProviderMap`，但它们指向同一个 `ProviderClientRecord`；引用计数则按 `provider.asBinder()` 存在 `mProviderRefCountMap` 中。

所以：

> authority 是查找维度，Binder identity 才是连接与计数维度。

`ProviderClientRecord` 本身把四类对象绑在一起：

```java
final String[] mNames;
final IContentProvider mProvider;
final ContentProvider mLocalProvider;
final ContentProviderHolder mHolder;
```

- `mProvider`：统一调用入口，本地时是 `Transport`，远端时是 proxy。
- `mLocalProvider`：只有本进程 Provider 才有真实 Java 实例。
- `mHolder`：携带 `ProviderInfo`、Binder 和 AMS 连接等系统元数据。

### 3.3 `acquireExistingProvider()` 的完整逻辑

核心代码可以简化为：

```java
public IContentProvider acquireExistingProvider(
        Context context, String auth, int userId, boolean stable) {
    synchronized (mProviderMap) {
        ProviderKey key = new ProviderKey(auth, userId);
        ProviderClientRecord record = mProviderMap.get(key);
        if (record == null) {
            return null;
        }

        IContentProvider provider = record.mProvider;
        IBinder binder = provider.asBinder();
        if (!binder.isBinderAlive()) {
            handleUnstableProviderDiedLocked(binder, true);
            return null;
        }

        ProviderRefCount ref = mProviderRefCountMap.get(binder);
        if (ref != null) {
            incProviderRefLocked(ref, stable);
        }
        return provider;
    }
}
```

按执行顺序拆开：

1. 用 `authority + userId` 构造 `ProviderKey`。同一个 authority 在不同 Android User 下不是同一个 Provider 实例。
2. 在 `mProviderMap` 中查 `ProviderClientRecord`。未命中直接返回 `null`，让外层进入 AMS 获取流程。
3. 取出 `IContentProvider`，再取它的 Binder identity。
4. 调用 `isBinderAlive()`。缓存项存在不代表远端进程仍存活。
5. Binder 已死时，立即清理这条 Binder 对应的引用状态和所有 authority 映射，然后返回 `null`。下一次获取会重新进入 AMS。
6. Binder 存活时，按 stable/unstable 增加本进程引用，再返回同一个 Provider。

整个方法持有 `mProviderMap` 锁，因此 authority 缓存、Binder 计数和死亡清理对本进程其他线程是一个一致状态。

### 3.4 为什么缓存命中不等于每次都通知 AMS

`ProviderRefCount` 保存的是本进程聚合后的状态：

```java
final ContentProviderHolder holder;
final ProviderClientRecord client;
int stableCount;
int unstableCount;
boolean removePending;
```

这一套普通引用计数主要针对需要释放的远端 Provider。本进程 Provider 会被当作进程内组件长期保存；`holder.noReleaseNeeded=true` 的特殊 Provider 也不会走普通的 `0 → remove` 生命周期。源码中有的版本不给本地 Provider 建 `ProviderRefCount`，有的特殊远端记录会使用哨兵计数。业务代码不应依赖这些哨兵值，只需要理解：`releaseProvider()` 只有在 `mProviderRefCountMap` 找到正常计数记录时才真正减计数。

应用进程里可能有很多调用方同时使用同一个 Provider。如果每一次 acquire/release 都跨 Binder 更新 AMS，成本会很高。因此 `incProviderRefLocked()` 的策略是：

- 本地计数每次都变。
- 只有某类计数从 `0 → 1` 时，才需要告诉 AMS“本进程开始持有这类引用”。
- 从 `1 → 2 → 3` 只在本进程累加，AMS 不需要知道进程内部有几个 Java 调用方。

稳定引用第一次出现时：

```java
stableCount++;
if (stableCount == 1) {
    ActivityManager.getService().refContentProvider(
        connection,
        +1, // stable delta
        0   // unstable delta
    );
}
```

unstable 同理，第一次出现时发送 `(0, +1)`。

这揭示了客户端计数与 AMS 计数的关系：

```text
ActivityThread 的 count
    = 本进程内部精确使用次数

AMS ContentProviderConnection 的 count
    = 这个客户端进程是否持有 stable/unstable 关系
      以及系统侧需要维护的连接状态
```

两边都叫 count，但粒度不同。前者聚合 Java 调用方，后者管理进程间依赖。

### 3.5 缓存未命中时，为什么还要按 authority 加一把锁

`acquireProvider()` 在本地未命中后，不会持有 `mProviderMap` 锁直接调用 AMS，而是为同一个 `authority + userId` 取得专用锁，再执行慢路径：

```java
ProviderKey key = getGetProviderKey(auth, userId);
synchronized (key) {
    holder = ActivityManager.getService().getContentProvider(...);
}
```

原因有两个：

1. AMS 查询、目标进程启动和 Provider 安装可能很慢，不能长期占用全局 `mProviderMap` 锁。
2. 同进程 Provider 的获取可能发生重入；持有全局锁跨进程调用会放大死锁风险。

专用锁只串行化“同一个 authority 的首次获取”，不同 authority 仍可并行。新版 AOSP 还让 `ProviderKey` 暂存异步发布返回的 `ContentProviderHolder`，在本地等待超时后继续安装。

即使两个线程发生竞争，`installProvider()` 仍有最后一道去重：进入 `mProviderMap` 锁后检查 Binder 或组件名是否已经安装。败者复用胜出的记录，并把自己刚从 AMS 获取的多余连接引用归还。

### 3.6 首次慢路径中的“连接转移”

缓存未命中时，AMS 在返回远端 `ContentProviderHolder` 前，已经为这次请求建立或增加了一份 `ContentProviderConnection` 引用。客户端拿到 holder 后再调用 `installProvider()`：

- 本地还没有该 Binder：写入 authority 映射，按本次 stable/unstable 类型创建初始 `ProviderRefCount(1, 0)` 或 `(0, 1)`。
- 本地已经有同一 Binder：说明另一个线程抢先安装成功。当前线程不再创建第二份缓存，而是增加已有本地计数，并调用 AMS `removeContentProvider(holder.connection, stable)` 归还自己这次慢路径多拿到的系统侧引用。

可以把这段理解成：

```text
AMS 先为每个慢路径请求预留一份连接引用
  → 客户端安装竞争
      ├── 胜者：把引用纳入新的 ProviderRefCount
      └── 败者：复用胜者缓存，归还多余的 AMS 引用
```

这也是 `installProvider()` 不只是“把 Binder 放进 Map”的原因。它还是客户端缓存与 system_server 连接账本之间的对账点。

因此第一层分流仍然是：

```text
本地缓存命中
    ├── 本地 Transport：同进程快速路径
    └── 远端 Proxy：跨进程热路径

本地缓存未命中
    └── 请求 AMS.getContentProvider()
```

源码位置：

```text
frameworks/base/core/java/android/app/ActivityThread.java
  - acquireProvider()
  - acquireExistingProvider()
  - incProviderRefLocked()
  - releaseProvider()
  - completeRemoveProvider()
  - handleUnstableProviderDiedLocked()
  - installProvider()
```

---

## 四、路径 A：同进程，Provider 已安装

当 `mProviderMap` 命中本地 Provider 时，调用链最短：

```text
ContentResolver.call()
  → ActivityThread.acquireExistingProvider()
  → 本地 ContentProvider.Transport
  → Transport.call()
  → ContentProvider.call()
  → Bundle 返回
```

`ContentProviderNative.asInterface()` 会先调用 `queryLocalInterface()`。如果 Binder 对象就在当前进程，客户端获得的是本地接口，而不是 `ContentProviderProxy`。

这条路径有两个重要特征：

- 不需要再次查询 AMS。
- 业务 `call()` 不经过 Binder 驱动，可能直接运行在调用方当前线程。

因此，Provider 代码不能假定自己总在 Binder 线程池中执行。如果同进程调用发生在主线程，Provider 中的磁盘 IO、锁等待或复杂计算会直接阻塞主线程。

---

## 五、路径 B：跨进程，远端 Proxy 已缓存

如果缓存命中的是远端 `IContentProvider` proxy，客户端同样不需要查询 AMS，但业务调用会穿过 Binder：

```text
ContentResolver.call()
  → acquireExistingProvider()
  → ContentProviderProxy.call()
  → Binder transact(CALL_TRANSACTION)
  → ContentProviderNative.onTransact()
  → ContentProvider.Transport.call()
  → ContentProvider.call()
```

`ContentProviderProxy.call()` 会把 calling package、attribution、authority、method、arg 和 extras 写入 `Parcel`，然后调用：

```java
mRemote.transact(
    IContentProvider.CALL_TRANSACTION,
    data,
    reply,
    0
);
```

服务端的 `ContentProviderNative.onTransact()` 解包参数，再分发到 `Transport.call()`。这条路径没有目标进程冷启动，但仍包含 Parcel 编解码、Binder 线程调度和远端进程故障等成本。

源码位置：

```text
frameworks/base/core/java/android/content/ContentProviderNative.java
frameworks/base/core/java/android/content/ContentProvider.java
```

---

## 六、路径 C：同进程，但 Provider 尚未安装

这条路径最容易被忽略。典型场景是在 `Application.attachBaseContext()` 中提前访问本进程 Provider。此时 Application 对象已经创建，但 `handleBindApplication()` 还没有执行正常的 Provider 批量安装。

本地缓存未命中后，调用仍会先进入 AMS：

```text
ContentResolver.call()
  → acquireExistingProvider() 未命中
  → AMS.getContentProvider(authority)
  → ContentProviderRecord.canRunHere(callerProcess) == true
  → 返回 ContentProviderHolder(provider = null)
  → 调用进程执行 installProvider(holder.info)
  → attachInfo() / Provider.onCreate()
  → 写入本地 Provider map
  → 本地 Transport.call()
```

`ContentProviderRecord.canRunHere()` 的源码条件是：

```java
public boolean canRunHere(ProcessRecord app) {
    return (info.multiprocess || info.processName.equals(app.processName))
            && uid == app.info.uid;
}
```

AMS 返回 `provider = null` 的 holder 并不是失败。它表达的是：Provider 可以在调用方进程内运行，AMS 只返回 `ProviderInfo`，让客户端自行实例化。

### 只安装一个，还是安装全部

在这次提前调用中，`ActivityThread` 执行的是单个：

```java
installProvider(context, holder, holder.info, ...);
```

因此，当下只会安装被请求 authority 对应的 Provider，不会立即安装本进程全部 Provider。

等 `attachBaseContext()` 返回后，正常启动主线继续：

```text
设置 mInitialApplication
  → installContentProviders(app, data.providers)
  → 遍历本进程全部 Provider
  → Application.onCreate()
```

这会带来一个微妙风险：提前安装的 Provider 仍然存在于 `data.providers` 中。常见 AOSP 实现可能在后续批量安装时再次创建一个候选实例，并调用 `attachInfo()/onCreate()`，直到写入本地 map 时才发现同名 Provider 已存在，保留先安装的实例。

所以不应依赖这类提前访问行为。确实无法避免时，至少要保证：

- 初始化副作用幂等。
- 不在 `Provider.onCreate()` 中递归访问自身。
- 避免 Provider A 与 Provider B 形成环形初始化依赖。
- 不依赖尚未完成的 `Application.onCreate()` 初始化。

---

## 七、路径 D：跨进程，Provider 已发布但客户端未缓存

当本地缓存未命中，且 Provider 不能在当前进程运行时，AMS 会检查目标 Provider 是否已经发布。

如果 `ContentProviderRecord` 已关联存活进程和 `IContentProvider`，AMS 会：

1. 检查调用方与 Provider 的关联关系和权限。
2. 通过 `incProviderCountLocked()` 建立或更新 `ContentProviderConnection`。
3. 调用 `updateOomAdjLocked()` 更新目标进程优先级。
4. 返回包含 Binder 的 `ContentProviderHolder`。
5. 客户端将远端 proxy 安装进本地 `ProviderClientRecord`。
6. 执行后续 Binder `call()`。

这条路径比 B 多了一次 system_server 寻址和连接建立，但没有进程启动成本。

---

## 八、目标进程存活，但 Provider 尚未发布

目标进程已经存在，并不代表目标 Provider 已经安装完成。AMS 在发现 `proc.thread` 可用、但 `cpr.provider` 仍为空时，会请求现有进程安装 Provider：

```text
AMS scheduleInstallProvider(cpi)
  → 目标进程 handleInstallProvider()
  → installContentProviders(单个 Provider)
  → publishContentProviders()
  → AMS 写入 cpr.provider 并 notifyAll()
  → 等待中的客户端取得 proxy
```

客户端会在 `ContentProviderRecord` 上等待发布，但等待并非无限期。源码使用 `CONTENT_PROVIDER_READY_TIMEOUT_MILLIS` 计算截止时间；超时仍未发布则返回失败。

这说明“目标进程存活”和“Provider 可调用”是两个不同状态。排查 Provider 首次调用卡顿时，必须把进程启动、Provider 安装、Provider 发布分别看待。

---

## 九、路径 E：跨进程，宿主进程尚未启动

当目标进程不存在时，AMS 会通过 `startProcessLocked()` 拉起宿主进程。完整流程是：

```text
AMS startProcessLocked / Zygote fork
  → ActivityThread.main() / attachApplication()
  → AMS 查询该进程应安装的 ProviderInfo 列表
  → bindApplication(AppBindData.providers)
  → ActivityThread.handleBindApplication()
  → LoadedApk.makeApplication()
  → Application 构造 / attach / attachBaseContext
  → 设置 mInitialApplication
  → installContentProviders()
  → installProvider() / instantiateProvider()
  → ContentProvider.attachInfo() / Provider.onCreate()
  → publishContentProviders()
  → AMS 保存 IContentProvider 并 notifyAll()
  → Instrumentation.callApplicationOnCreate()
  → Application.onCreate()
  → 客户端取得 proxy，发起 Binder call
```

### Provider 与 Application 的准确生命周期关系

常见说法是“Provider 比 Application 更早创建”，但这句话不够准确。Android 30 的 `handleBindApplication()` 顺序是：

```text
Application 对象构造
  → Application.attach() / attachBaseContext()
  → Provider.attachInfo() / Provider.onCreate()
  → Application.onCreate()
```

因此，准确结论是：

> `Provider.onCreate()` 早于 `Application.onCreate()`，但晚于 Application 对象构造和 `attachBaseContext()`。

这也解释了为什么很多 SDK 使用 Provider 做自动初始化，以及为什么 Provider 中不能默认依赖 `Application.onCreate()` 已经完成。

源码位置：

```text
frameworks/base/core/java/android/app/ActivityThread.java
  - handleBindApplication()
  - installContentProviders()
  - installProvider()

frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
  - generateApplicationProvidersLocked()
  - getContentProviderImpl()
  - publishContentProviders()
```

---

## 十、`installProvider()` 与 `attachInfo()` 的职责

对于本地 Provider，`ActivityThread.installProvider()` 主要完成：

1. 创建合适的 Context。
2. 通过 `AppComponentFactory.instantiateProvider()` 实例化 Provider。
3. 调用 `getIContentProvider()` 取得内部 `Transport`。
4. 调用 `attachInfo(context, info)`。
5. 将 Provider 注册进 `mProviderMap`、`mLocalProviders` 等本地结构。

`ContentProvider.attachInfo()` 会设置：

- `mContext` 和当前 UID。
- authority。
- exported 状态。
- read/write permission。
- path permission。
- Transport 使用的 `AppOpsManager`。

这些元数据就绪后，`attachInfo()` 最后调用业务 Provider 的 `onCreate()`。

对于远端 Provider，客户端不会创建业务 Provider 实例，而是把 AMS 返回的 Binder proxy 记录到本进程缓存，并维护它的引用计数。

---

## 十一、`ContentProvider.Transport` 是本地与远端的共同落点

`ContentProvider.Transport` 继承 `ContentProviderNative`，后者实现 `IContentProvider` Binder 接口。

```java
class Transport extends ContentProviderNative {
    volatile ContentInterface mInterface = ContentProvider.this;
}
```

同进程调用时，`IContentProvider` 指向本地 Transport，方法调用停留在当前 Java 调用栈。跨进程调用时，客户端 proxy 经 Binder driver 到达服务端 `ContentProviderNative.onTransact()`，最终仍落到同一个 Transport。

Android 30 中，`Transport.call()` 的核心逻辑是：

```java
validateIncomingAuthority(authority);
Bundle.setDefusable(extras, true);
Pair<String, String> original =
    setCallingPackage(new Pair<>(callingPkg, attributionTag));
try {
    return mInterface.call(authority, method, arg, extras);
} finally {
    setCallingPackage(original);
}
```

需要避免一种过度简化：不能笼统地说每个 `call()` 都在 Transport 内执行完整的 read/write permission、URI grant 和 AppOps 校验。Provider 获取阶段已经由 AMS 做了 exported、关联关系和权限检查；不同 CRUD/file API 又会在 Transport 中做各自的 URI 权限与 AppOps 处理。`call()` 是自定义 RPC 入口，敏感 method 仍应在业务 Provider 中按 UID、包名、签名或自定义权限进行显式校验。

---

## 十二、stable 与 unstable：两类引用，不是两个 Provider

前面已经看到，`ApplicationContentResolver` 最终只是把一个 `stable` 布尔值传给 `ActivityThread`。真正的状态机由客户端 `ProviderRefCount` 和服务端 `ContentProviderConnection` 共同完成。

### 12.1 第一次安装时，计数从哪里来

远端 Provider 第一次被安装到客户端缓存时，`installProvider()` 会按本次 acquire 类型创建 `ProviderRefCount`：

```java
ProviderRefCount ref = stable
        ? new ProviderRefCount(holder, client, 1, 0)
        : new ProviderRefCount(holder, client, 0, 1);

mProviderRefCountMap.put(provider.asBinder(), ref);
```

对应地，AMS 在 `getContentProviderImpl()` 中创建或复用 `ContentProviderConnection`：

```text
Provider 侧 ContentProviderRecord
        ↕
ContentProviderConnection
        ↕
Client 侧 ProcessRecord
```

这条 connection 同时挂在 Provider 的连接列表和客户端进程的 Provider 连接列表中。它既是计数容器，也是 OOM 调整、进程关联统计、等待发布和死亡清理的凭据。

首次请求 stable 时，系统侧初始化为 `stable=1, unstable=0`；首次请求 unstable 时反过来。后续同一客户端进程再次获取同一 Provider，会复用这条 connection。

### 12.2 Stable 引用意味着什么

stable 表达调用方对 Provider 的稳定依赖。它参与目标进程优先级计算和 Provider 死亡后的依赖清理，但不保证远端进程永远不死。

在 Android 11 的 AMS 获取路径中，连接建立后会调用 `updateOomAdjLocked(...GET_PROVIDER)`。新版本则把 `ContentProviderConnection` 纳入统一 OOM Adjuster 连接模型。实现形式在演进，但核心语义一致：系统知道“客户端进程正在依赖这个 Provider 进程”，并据此重新评估宿主进程的重要性。

stable 不是“杀不死”。目标进程仍可能因为崩溃、强杀或系统策略消失；stable 只表示更强的依赖关系和更严格的死亡处理。

Android 11 的 Provider 进程死亡清理逻辑更能体现“严格”在哪里：如果某个 `ContentProviderConnection.stableCount > 0`，非 persistent 的依赖客户端可能被系统以 `DEPENDENCY_DIED` 原因终止；只有不存在 stable 引用时，系统才走 `unstableProviderDied()` 通知客户端自行清缓存和恢复。

所以 stable/unstable 的差异不只是 OOM 权重：

```text
stable 依赖：Provider 意外死亡可能连带终止依赖方
unstable 依赖：允许客户端观察死亡并尝试重新获取
```

具体退出策略会随 Android 版本演进，但“stable 是强依赖、unstable 是可恢复弱依赖”是更准确的工程语义。

### 12.3 Unstable 引用与故障恢复

unstable 表达较弱依赖。远端死亡时，调用方应处理 `DeadObjectException`，调用 `unstableProviderDied()` 清理旧关系，再决定是否重新获取 Provider。

客户端发现 Binder 已死时，`handleUnstableProviderDiedLocked()` 会：

1. 从 `mProviderRefCountMap` 删除该 Binder 的计数。
2. 遍历 `mProviderMap`，删除所有指向同一 Binder 的 authority 映射。
3. 如果死亡是客户端在调用时主动发现的，通知 AMS `unstableProviderDied(connection)`。

AMS 不会盲信客户端。它会先 `pingBinder()`；确认 Binder 确实死亡且 connection 仍指向同一 Provider 后，才按进程死亡路径继续清理。这样可以避免客户端误报或 Provider 已重启换代造成竞态。

### 12.4 `releaseProvider()` 不是简单的 `count--`

普通 release 分三种情况。

#### 情况一：本地还有同类引用

例如 stable 从 3 降到 2：

```text
客户端 stableCount: 3 → 2
AMS：不通知
连接：继续保留
```

AMS 只关心客户端进程是否仍持有 stable 关系，不需要知道进程内部有 2 个还是 3 个使用者。

#### 情况二：某类引用归零，但另一类还存在

例如 `stable:1→0`，同时 `unstableCount>0`：

```java
refContentProvider(connection, -1, 0);
```

客户端通知 AMS 去掉 stable 关系，但连接仍由 unstable 引用维持。反过来，unstable 归零而 stable 仍存在时，发送 `(0, -1)`。

#### 情况三：stable 和 unstable 都归零

最后一份 stable 被释放时，Android 11 的客户端不会立即把系统侧总计数降到零，而是先做一次转换：

```java
refContentProvider(
    connection,
    -1, // 去掉最后一份 stable
    +1  // 临时保留一份 unstable
);
```

随后设置 `removePending=true`，向 `ActivityThread.H` 发送一个延迟 `CONTENT_PROVIDER_RETAIN_TIME` 的 `REMOVE_PROVIDER` 消息。Android 11/当前客户端源码中该保留时间为 1 秒。

这 1 秒不是业务超时，而是防抖窗口：很多代码会在很短时间内释放后又重新查询同一个 Provider。立即删除缓存、拆连接，再马上走 AMS 重建，会造成明显抖动。

### 12.5 移除窗口内重新 acquire 会发生什么

假设本地状态是：

```text
stable=0, unstable=0, removePending=true
AMS 侧仍保留一份临时 unstable
```

此时再次 stable acquire：

1. `acquireExistingProvider()` 命中原缓存。
2. `incProviderRefLocked()` 将 stable 从 0 加到 1。
3. 发现 `removePending=true`，取消延迟移除消息。
4. 调用 `refContentProvider(connection, +1, -1)`：把 AMS 侧临时 unstable 原子转换为 stable。

如果重新获取的是 unstable，只需取消 pending remove；AMS 侧那份临时 unstable 正好可以继续代表这条关系，不必重复加一。

源码注释把 stable 重获形容为“从死亡边缘抢回 Provider”。本质上，这是一个连接复用优化，同时避免两次独立 Binder 更新之间出现总引用短暂归零。

### 12.6 延迟到期后如何真正删除

`completeRemoveProvider()` 执行时还会再次检查 `removePending`，因为延迟期间可能已经发生重新 acquire。

- `removePending=false`：说明 Provider 又被使用，放弃本次删除。
- 仍为 `true`：从 `mProviderRefCountMap` 删除 Binder 计数；从 `mProviderMap` 删除所有指向该 Binder 的 authority；最后调用 AMS `removeContentProvider(connection, false)`，释放系统侧那份临时 unstable。

AMS 侧计数变成 0 后，会把 `ContentProviderConnection` 同时从 Provider 和客户端进程的连接列表移除，停止进程关联统计，并重新评估 Provider 宿主进程的 OOM 优先级。

这里还有一个刻意设计的 API 边界：`refContentProvider(connection, stableDelta, unstableDelta)` 只负责在连接仍存在时调整两类计数，它会拒绝让 `stable + unstable` 直接变成 0。真正的最后一次释放必须走 `removeContentProvider()`，再由 `decProviderCountLocked()` 完成连接拆除。

为什么要分两条 API？因为“改一个数字”和“销毁一条系统关系”不是同一件事。归零时还必须同步完成：

- 从 `ContentProviderRecord.connections` 移除连接。
- 从客户端 `ProcessRecord` 的 Provider 连接列表移除。
- 停止 procstats association。
- 记录 Provider 最近使用时间，减少宿主进程抖动。
- 重新计算 Provider 宿主进程的 OOM adj。

把归零限制在专用删除路径，可以避免某个普通 delta 更新绕过这些副作用。

这里要区分两个“延迟”：

- Android 11 客户端的 `CONTENT_PROVIDER_RETAIN_TIME`：用于本进程缓存和临时 unstable 的 1 秒防抖。
- 新版 system_server 中还可能对最后连接移除增加更长的延迟，以减少 Provider 进程反复升降和连接 churn。

具体时间是版本实现细节，不应当被业务依赖；稳定的设计意图是“最后引用释放后不要立刻抖动式拆建”。

### 12.7 `call()` 与 `query()` 为什么采用不同策略

Android 30 的常见实现是：

- `ContentResolver.call()` 直接使用 stable `acquireProvider()`。
- `ContentResolver.query()` 先使用 `acquireUnstableProvider()`。
- `query()` 遇到 `DeadObjectException` 后调用 `unstableProviderDied()`，再获取 stable Provider 重试。
- query 成功后，`CursorWrapperInner` 持有 stable 引用，直到 Cursor 关闭才释放。

因此，不能把 stable/unstable 简化成“所有 API 都先 unstable，失败再 stable”。具体策略取决于 API 路径和 Android 版本。

把一次完整生命周期写成状态表，会更直观：

| 动作 | 客户端 `ProviderRefCount` | AMS `ContentProviderConnection` |
| --- | --- | --- |
| 第一次 stable acquire | `s=1,u=0` | 新建连接，`s=1,u=0` |
| 同一客户端进程再次 stable acquire | `s=2,u=0` | 通常仍是 `s=1,u=0` |
| 第一次 unstable acquire | `s=2,u=1` | `s=1,u=1` |
| 释放一份 stable | `s=1,u=1` | 不变 |
| 最后一份 stable 释放，unstable 仍在 | `s=0,u=1` | `s=0,u=1` |
| 所有引用释放 | `s=0,u=0, pending` | 临时保留 `u=1` |
| 保留期内重新 stable acquire | `s=1,u=0` | 原子转换为 `s=1,u=0` |
| 保留期结束且无人重获 | 删除本地缓存 | 删除 connection，更新 OOM |
| 远端 Binder 死亡 | 立即删 Binder 计数和全部 authority 映射 | 确认死亡并进入进程清理 |

### 12.8 从这套实现可以推导出的通用设计

Provider 的引用管理不是孤立技巧，它体现了几种常见的系统设计方法：

1. **寻址与资源身份分离**：authority 用来查找，Binder identity 用来判定“是不是同一条真实连接”。
2. **进程内聚合、跨进程降频**：先把多个 Java 调用方聚合成边界变化，再通知 system_server。
3. **最后引用释放采用两阶段提交**：先进入 pending 状态并保留临时引用，延迟到期后再真正拆连接。
4. **慢路径必须可并发去重**：authority 级锁减少重复请求，安装阶段再按 Binder/组件名做最终仲裁。
5. **死亡清理按资源身份反向扫索引**：Binder 一死，删除所有引用它的 authority，避免留下可命中但不可用的缓存。
6. **连接对象承载调度语义**：`ContentProviderConnection` 不只是引用计数器，还连接 OOM、关联统计、等待发布和依赖死亡。

用这六点回看 Service 绑定、Binder proxy 缓存或跨进程连接池，会发现相似问题都需要回答：查找键是什么、真实资源身份是什么、局部引用如何聚合、最后释放如何防抖、死亡如何一次清完全部索引。

### 12.9 方法级源码阅读索引

如果要沿本章自己读源码，建议按状态变化而不是按文件顺序：

| 状态变化 | 入口方法 | 重点观察 |
| --- | --- | --- |
| API 选择 stable/unstable | `ContextImpl.ApplicationContentResolver` | `true/false` 如何传入 `ActivityThread` |
| 缓存命中 | `acquireExistingProvider()` | `ProviderKey`、Binder 存活检查、计数增加 |
| 缓存未命中 | `acquireProvider()` | authority 级锁、AMS 慢路径、等待发布 |
| 写入缓存 | `installProviderAuthoritiesLocked()` | 多 authority 指向同一 `ProviderClientRecord` |
| 安装竞争 | `installProvider()` | Binder/组件名去重、归还多余 AMS 引用 |
| 本地引用增加 | `incProviderRefLocked()` | 只在 `0→1` 边界同步 AMS |
| 本地引用释放 | `releaseProvider()` | `1→0`、临时 unstable、pending remove |
| 延迟删除 | `completeRemoveProvider()` | 再检查竞态、清两张表、通知 AMS |
| 远端死亡 | `handleUnstableProviderDiedLocked()` | 按 Binder 删除所有 authority 映射 |
| 系统侧建连接 | `incProviderCountLocked()` | connection 同挂 provider/client、OOM/LRU |
| 系统侧调计数 | `refContentProvider()` | delta 校验，不允许普通调整直接归零 |
| 系统侧拆连接 | `removeContentProvider()` / `decProviderCountLocked()` | 连接、association、OOM 的完整收尾 |

---

## 十三、从图中读出性能层次

从短到长，五种状态对应不同成本：

| 路径 | 状态 | 主要成本 |
| --- | --- | --- |
| A | 同进程，Provider 已安装 | 本地 Java 调用，路径最短 |
| C | 同进程，Provider 尚未安装 | AMS 寻址 + 单个 Provider 本地安装 |
| B | 跨进程，proxy 已缓存 | Binder IPC，无 AMS 寻址与冷启动 |
| D | 跨进程，Provider 已发布但未缓存 | AMS 寻址 + 连接建立 + Binder IPC |
| 启动中 | 进程存活、Provider 未发布 | schedule install + 等待 publish + Binder IPC |
| E | 目标进程未启动 | fork + Application attach + Provider 安装/发布 + Binder IPC |

所以“第一次 Provider 调用慢”不能只归因于 Binder。真正昂贵的部分往往是目标进程冷启动、类加载、Application attach、Provider 初始化和发布等待。

---

## 十四、工程实践：什么时候适合把 Provider 当主 IPC

ContentProvider 更适合这些场景：

- 结构化数据查询和增删改。
- 配置或状态读取。
- 跨 App 文件暴露。
- 低频轻量 RPC。
- 需要通过 authority 按需发现并拉起进程。
- 配合 `ContentObserver` 做“数据可能已变化”的通知。

AIDL 或 Binder Service 更适合：

- 高频、低延迟调用。
- 强类型复杂 RPC。
- 双向回调。
- 长连接。
- 持续状态机或流式交互。

如果项目决定把 `ContentProvider.call()` 作为主 IPC 通道，至少应建立这些约束：

```text
IpcProvider
  → Caller / Permission Checker
  → Method Router
  → Typed Handler
  → Business Service
```

- method 带稳定版本，例如 `account.getLoginState.v1`。
- 入参、返回和错误码有固定协议，不依赖异常或 null 猜测结果。
- 服务端校验调用 UID、包名、签名、权限和 method 白名单。
- Provider 方法保持短、快、可失败、可重试。
- 不在 `Provider.onCreate()` 中执行网络请求、重磁盘 IO 或同步等待。
- 不持有业务锁再发起其他跨进程调用，避免跨进程死锁。
- Bundle 只承载小数据；大文件使用 `openFile()` / `ParcelFileDescriptor`。
- Provider 方法按任意线程、并发访问和可重入设计。

---

## 十五、最终心智模型

把整条链路压缩成一句话：

> ContentProvider 是 Android Framework 托管的一种数据型 IPC 服务。`authority` 负责发现，`ActivityThread` 负责客户端缓存与本地安装，AMS 负责进程和连接管理，`IContentProvider / Transport` 统一同进程与跨进程调用，stable/unstable 引用则把一次 API 调用纳入系统的进程存活和故障恢复模型。

理解 ContentProvider，不能只停在 `query()`、`insert()` 和 `call()`。真正决定它工程价值的，是 Framework 已经替开发者完成的这几件事：

- 服务发现。
- 目标进程拉起。
- Provider 生命周期托管。
- 本地与远端统一抽象。
- Binder 代理缓存与引用管理。
- 权限边界与调用方归因。
- Cursor、Bundle 和文件描述符等不同数据承载方式。

也正因为这些能力都藏在一行 `ContentResolver.call()` 背后，首次调用的时机、Provider 初始化重量、线程安全和协议治理，才值得在大型项目里被当作正式架构问题处理。

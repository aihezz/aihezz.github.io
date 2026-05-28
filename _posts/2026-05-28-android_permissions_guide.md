---
layout: post
title:  "Android 权限体系与自定义权限指南"
date:   2026-05-28 10:00:00 +0800
categories: [Android]
tags: [android, security]
lang: zh-CN
---

本文档旨在全面介绍 Android 权限体系的基础知识，并在此基础上，深度剖析**自定义权限**的原理、实现方式及其在复杂业务场景中的核心应用。

***

## 第一部分：Android 权限体系基础 

Android 权限体系是操作系统的安全架构核心，通过限制应用访问敏感用户数据（如联系人、位置）和系统功能（如摄像头、麦克风、网络），防止恶意软件或设计不当的应用对系统造成破坏。

### 1. 权限的四大核心分类及常见权限

Android 根据权限带来的风险大小，将权限主要分为四个保护级别（`protectionLevel`）：

#### 1.1 普通权限 (Normal Permissions)

**特点**：对用户隐私或设备安全风险极小。只要在 `AndroidManifest.xml` 中声明，系统会在应用安装时自动授予，且用户无法在设置中撤销。
**常见权限**：

- `INTERNET`：允许应用访问网络（几乎所有联网 App 必备）。
- `ACCESS_NETWORK_STATE`：允许应用获取网络连接状态（判断 Wi-Fi/移动数据可用性）。
- `ACCESS_WIFI_STATE`：允许应用获取 Wi-Fi 的连接状态信息。
- `VIBRATE`：允许应用控制设备的震动马达。
- `WAKE_LOCK`：允许应用防止设备屏幕变暗或处理器休眠（常用于播放视频或后台下载）。
- `BLUETOOTH`：允许应用连接到已配对的蓝牙设备。
- `FOREGROUND_SERVICE`：允许应用使用前台服务（Android 9.0 引入）。

#### 1.2 危险权限 (Dangerous Permissions)

**特点**：涉及用户敏感私密数据或可能影响系统及其他应用的操作。除了在清单文件声明，还必须在应用运行时向用户弹出系统对话框**动态申请**。用户随时可以在系统设置中收回这些权限。
**常见权限（按权限组划分，同组权限同意一个即同组授权）**：

- **位置组 (Location)**：`ACCESS_FINE_LOCATION` (精确定位)、`ACCESS_COARSE_LOCATION` (粗略定位)、`ACCESS_BACKGROUND_LOCATION` (后台定位)。
- **相机组 (Camera)**：`CAMERA`。
- **麦克风组 (Microphone)**：`RECORD_AUDIO`。
- **存储与媒体组 (Storage / Media)**：`READ_EXTERNAL_STORAGE` / `WRITE_EXTERNAL_STORAGE` (旧版存储)、`READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO` / `READ_MEDIA_AUDIO` (Android 13 细分)、`READ_MEDIA_VISUAL_USER_SELECTED` (Android 14 部分授权)。
- **通知组 (Notifications)**：`POST_NOTIFICATIONS` (Android 13 引入)。
- **联系人组 (Contacts)**：`READ_CONTACTS` / `WRITE_CONTACTS`。
- **电话组 (Phone)**：`READ_PHONE_STATE` (读取设备信息/IMEI等，常用于风控)、`CALL_PHONE`。

#### 1.3 特殊权限 (Special Permissions)

**特点**：涉及极其敏感的系统级操作。无法通过代码直接弹出常规授权框，必须通过 `Intent` 引导用户跳转到系统设置的**特定权限管理页面**手动开启。
**常见权限**：

- `SYSTEM_ALERT_WINDOW`：悬浮窗权限（如微信语音悬浮球）。
- `WRITE_SETTINGS`：修改系统设置（如修改屏幕亮度、休眠时间）。
- `REQUEST_INSTALL_PACKAGES`：允许安装来自未知来源的应用（Android 8.0 引入）。
- `MANAGE_EXTERNAL_STORAGE`：所有文件访问权限（文件管理器、杀毒软件必备，Android 11 引入）。
- `PACKAGE_USAGE_STATS`：允许获取设备上其他应用的使用统计信息（如屏幕时间管理）。

#### 1.4 签名权限 (Signature Permissions)

**特点**：最高安全级别，系统仅在请求应用与声明应用具有**相同数字签名**时自动授予。主要用于系统应用或自家应用矩阵。
**常见权限**：

- `INSTALL_PACKAGES` / `DELETE_PACKAGES`：静默安装或卸载（仅限系统预装应用）。
- `BIND_ACCESSIBILITY_SERVICE`：绑定无障碍服务。
- `BIND_NOTIFICATION_LISTENER_SERVICE`：绑定通知监听服务（如智能手表消息同步）。

### 2. 常规权限的开发与使用范式

现代 Android 开发中，申请危险权限的标准范式如下：

**1. 清单声明**：所有需要的权限必须在 `AndroidManifest.xml` 中声明。

```xml
<uses-permission android:name="android.permission.CAMERA" />
```

**2. 运行时动态申请 (Kotlin 示例)**：使用最新的 Activity Result API 进行状态检查和申请。

```kotlin
// 1. 注册权限申请回调
val requestPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { isGranted: Boolean ->
    if (isGranted) {
        // 权限被授予，执行需要权限的操作
    } else {
        // 权限被拒绝，提示用户或执行降级处理
    }
}

// 2. 检查并申请权限
if (ContextCompat.checkSelfPermission(context, Manifest.permission.CAMERA) == PackageManager.PERMISSION_GRANTED) {
    // 已经有权限，直接执行
} else {
    // 可选：判断是否需要向用户解释为何需要该权限 (shouldShowRequestPermissionRationale)
    // 发起权限申请
    requestPermissionLauncher.launch(Manifest.permission.CAMERA)
}
```

***

## 第二部分：自定义权限 

在复杂业务和 SDK 开发中，常规的系统权限往往不够用。**自定义权限**主要用于限制其他应用对本应用四大组件（Activity、Service、BroadcastReceiver、ContentProvider）的访问，是应用间通信（IPC）的一道坚固防火墙。

### 1. 为什么需要自定义权限？

如果你的应用暴露了某些组件（`android:exported="true"`）供外部调用，但不希望任何恶意应用都能随意调用，就需要通过自定义权限来进行**调用者鉴权**和**访问隔离**。

### 2. 自定义权限的实现三部曲

#### 步骤一：声明自定义权限（服务端 App）

在提供组件的应用的 `AndroidManifest.xml` 中，使用 `<permission>` 标签定义权限。

```xml
<!-- 定义一个签名级别的自定义权限 -->
<permission 
    android:name="com.yourdomain.app.permission.MY_CUSTOM_PERMISSION"
    android:label="My Custom Permission"
    android:description="@string/perm_desc"
    android:protectionLevel="signature" /> 
```

> **⚠️ 核心点：`protectionLevel`** **的选择**
> 绝大多数情况下，自定义权限应使用 **`signature`**。这意味着只有与你使用相同签名文件打包的应用才能获得此权限。如果错设为 `normal` 或 `dangerous`，恶意应用只需在清单中声明即可突破防御，失去了保护意义。

#### 步骤二：保护组件（服务端 App）

在需要保护的组件标签中，添加 `android:permission` 属性。

```xml
<!-- 保护 Activity 不被外部随意拉起 -->
<activity 
    android:name=".SecureActivity"
    android:exported="true"
    android:permission="com.yourdomain.app.permission.MY_CUSTOM_PERMISSION">
</activity>

<!-- 保护 Service 不被外部随意绑定或启动 -->
<service
    android:name=".BackgroundSyncService"
    android:exported="true"
    android:permission="com.yourdomain.app.permission.MY_CUSTOM_PERMISSION">
</service>
```

#### 步骤三：外部应用申请权限（客户端 App）

调用方如果想访问受保护的组件，必须在其 `AndroidManifest.xml` 中声明使用该权限，且**必须使用与服务端相同的签名打包**。

```xml
<!-- 声明需要使用服务端的自定义权限 -->
<uses-permission android:name="com.yourdomain.app.permission.MY_CUSTOM_PERMISSION" />
```

### 3. 自定义权限的核心业务场景

#### 场景一：应用家族/生态内的数据共享（最常见）

**痛点**：同一家公司的多个 App（例如“主端 App”与“极速版 App”，或矩阵应用群）之间需要深度的 IPC 调用（如通过 AIDL 绑定 Service、ContentProvider 读取共享登录态）。
**方案**：通过 `signature` 级别的自定义权限，构建一个只有“自家兄弟”才能进入的内部通信网络，完美阻挡第三方应用窃取数据。

#### 场景二：防组件劫持攻击（保护核心逻辑）

**痛点**：处理支付逻辑的 Activity 或执行后台敏感操作的 Service，如果因为业务需要暴露给外部，极易被恶意应用强行拉起，导致越权操作。
**方案**：强制要求调用者持有特定的自定义权限，从而只允许受信任的应用唤起它们。

#### 场景三：安全的广播机制 (Broadcast Security)

- **安全发送广播**：使用 `sendBroadcast(intent, "com.yourdomain.app.permission.MY_CUSTOM_PERMISSION")`，确保广播携带的敏感数据只有拥有该权限的 Receiver 才能接收到。
- **安全接收广播**：在 `<receiver>` 配置中添加自定义权限，确保只有受信任的应用才能向你发送广播，防止恶意伪造广播触发业务逻辑（如伪造支付成功广播）。

#### 场景四：对外提供 SDK 或开放平台 API

**痛点**：作为一个平台级应用，需要为外部第三方 App 提供可控的接入能力。
**方案**：定义 `normal` 或 `dangerous` 级别的自定义权限。要求第三方应用接入时必须在清单中声明。服务端在收到请求时，可通过 `Binder.getCallingUid()` 和 `checkCallingPermission()` 进行动态的调用者鉴权和调用审计。

***

## 第三部分：权限体系的底层机制与演进 (拓展阅读)

### 1. 底层机制 (Linux UID & AppOps)

- **基于 Linux 的沙箱隔离**：Android 为每个 App 分配独立 UID。底层权限（如网络、蓝牙）在安装时映射为对应的 GID（用户组 ID）。例如拥有 `inet` GID 的进程才能创建网络套接字。
- **AppOps (Application Operations)**：Framework 层的权限动态管控中心。用户在设置中关闭动态权限，实质是修改了 AppOps 的状态。`checkSelfPermission` 本质上也是在查询 AppOps 的状态。

### 2. 关键版本演进

- **Android 6.0**：引入**动态权限（Runtime Permissions）**，标志着权限管理的现代化。
- **Android 10/11**：引入**分区存储（Scoped Storage）**，单次授权机制（仅限这一次），并严格限制后台定位。
- **Android 13/14**：存储权限进一步细化（拆分图片、视频、音频），新增照片/视频的**部分访问权限**（允许用户只挑选几张照片授权），并要求发送桌面通知必须动态申请 `POST_NOTIFICATIONS` 权限。

***

## 第四部分：权限开发最佳实践

1. **最小特权原则**：只申请核心功能绝对需要的权限。例如，如果只需拍照，优先考虑调用系统相机 Intent (`MediaStore.ACTION_IMAGE_CAPTURE`)，而不是直接申请 `CAMERA` 权限。
2. **场景化按需申请**：切忌在应用刚启动时“查户口”式集中索要一堆权限。应在用户点击某个具体功能时，再发起对应权限的申请，并配以友好的用途解释。
3. **优雅降级体验**：当用户拒绝权限时，应用不应直接崩溃，而是应禁用对应功能模块，并提供清晰的引导提示，告知用户如何在设置中重新开启。
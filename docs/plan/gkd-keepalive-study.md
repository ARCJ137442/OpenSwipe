# GKD 无障碍保活策略研究

> 基于 gkd-kit/gkd (37k+ stars) 源码分析，commit 时间 2026 年初

## 1. GKD 保活策略完整清单

### 策略 A: TYPE_ACCESSIBILITY_OVERLAY 保活窗口

**文件**: `A11yService.kt` -> `useAliveOverlayView()`

GKD 在 `onServiceConnected` 时创建一个 1x1 像素的不可见 `TYPE_ACCESSIBILITY_OVERLAY` 窗口：

```kotlin
val tempView = View(context)
val lp = WindowManager.LayoutParams().apply {
    type = WindowManager.LayoutParams.TYPE_ACCESSIBILITY_OVERLAY
    format = PixelFormat.TRANSLUCENT
    flags = flags or FLAG_NOT_TOUCHABLE or FLAG_NOT_FOCUSABLE
    width = 1; height = 1
}
wm.addView(tempView, lp)
```

**原理**: AccessibilityService 持有 overlay 窗口时，系统对其进程有更高的保护优先级。窗口存在意味着服务"正在使用中"，OOM killer 会优先保留。

**关键细节**: `onDestroyed` 时 `removeView`，失败时 toast 提示用户重启。

### 策略 B: StatusService 前台服务 (foregroundServiceType=specialUse)

**文件**: `StatusService.kt`, `Notif.kt`, `AndroidManifest.xml`

GKD 有独立的 `StatusService` 作为前台服务：
- 类型为 `foregroundServiceType="specialUse"`（Android 14+ 要求声明具体用途）
- 显示常驻通知，内容动态更新（规则匹配状态、触发次数等）
- **在 A11yService.onCreated 中自动启动**: `onCreated { StatusService.autoStart() }`

通知使用 `ServiceCompat.startForeground()` 绑定到 Service，确保前台状态。

**autoStart 机制**: 仅在 `enableStatusService=true` 且通知权限已授予时才启动，有 1 秒防抖。

### 策略 C: WRITE_SECURE_SETTINGS 权限实现无障碍自启/自修复

**文件**: `GkdTileService.kt` -> `fixA11yService()`

这是 GKD 最核心的保活能力。通过 `WRITE_SECURE_SETTINGS` 权限，GKD 可以：

1. **直接操作系统无障碍设置**，无需用户手动去设置页面开关
2. **检测到无障碍异常时自动修复**：
   ```kotlin
   // 无障碍出现故障, 重启服务
   names.remove(A11yService.a11yCn)
   app.putSecureA11yServices(names)
   delay(1000)  // 必须等待，否则概率不触发重启
   names.add(A11yService.a11yCn)
   app.putSecureA11yServices(names)
   ```
3. **支持通过快捷磁贴/TileService 一键开关无障碍**

授权方式：通过 Shizuku 或 ADB 命令 `pm grant <pkg> android.permission.WRITE_SECURE_SETTINGS`

### 策略 D: 快捷磁贴 (Quick Settings Tile) 实现"无感保活"

**文件**: `GkdTileService.kt`, `AuthA11yPage.kt`

GKD 在 `AuthA11yPage` 中专门有"无感保活"功能说明：

> 添加通知栏快捷开关 -> 只要此快捷开关在通知面板可见，无论是系统杀后台还是自身崩溃，简单下拉打开通知即可重启

配合 `WRITE_SECURE_SETTINGS`，点击磁贴即可直接重新注册无障碍服务，无需进入系统设置页。

### 策略 E: 电池优化白名单

**文件**: `PermissionState.kt`, `AndroidManifest.xml`

- 声明了 `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` 权限
- 有完整的 `ignoreBatteryOptimizationsState` 权限检测和引导流程
- 通过 XXPermissions 框架统一管理权限请求

### 策略 F: Shizuku + UiAutomation 双模式

**文件**: `AuthA11yPage.kt`, `GkdTileService.kt`

GKD 支持两种工作模式：
- **无障碍模式 (A11yMode)**: 传统 AccessibilityService
- **自动化模式 (AutomationMode)**: 通过 Shizuku 获取 UiAutomation 实例

自动化模式的优势：不依赖系统无障碍框架，不会被其他应用检测为无障碍。两种模式可以根据应用动态切换。

### 策略 G: 全面的 AppOps 权限管理

**文件**: `PermissionState.kt`

GKD 检测并管理以下 AppOps 权限：
- `OP_ACCESS_ACCESSIBILITY` — 访问无障碍（Android 10+）
- `OP_CREATE_ACCESSIBILITY_OVERLAY` — 创建无障碍悬浮窗
- `OP_FOREGROUND_SERVICE_SPECIAL_USE` — 特殊前台服务（Android 14+）
- `OP_ACCESS_RESTRICTED_SETTINGS` — 访问受限设置（Android 14+）

当任一权限被系统限制时，通知栏显示"权限受限，请解除限制"并引导用户处理。

### GKD 没有使用的策略

- **没有 START_STICKY**: 搜索结果为空，GKD 不依赖 Service 重启机制
- **没有进程守护/双进程互拉**: 无此类代码
- **没有 onDestroy 中发广播自启**: 无此类代码
- **没有 JobScheduler/WorkManager 定时检查**: 无此类代码

## 2. 与 OpenSwipe 当前策略对比

| 策略 | GKD | OpenSwipe | 差距评估 |
|------|-----|-----------|----------|
| Overlay 保活窗口 | 1x1px TYPE_ACCESSIBILITY_OVERLAY | 有（手势触发区 overlay） | OpenSwipe 已有类似效果 |
| 前台服务 | StatusService, specialUse, 动态通知 | KeepAliveService, **仅开机启动** | **严重差距** |
| WRITE_SECURE_SETTINGS | 核心能力，自动修复无障碍 | 无 | **最大差距** |
| 快捷磁贴 | 有，配合 WRITE_SECURE_SETTINGS 实现一键恢复 | 无 | 中等差距 |
| 电池优化白名单 | 有权限 + 引导 UI | 有权限声明，无引导 | 小差距 |
| AppOps 权限检测 | 全面检测 6+ 项，受限时主动提示 | 无 | 中等差距 |
| 双模式(A11y/Shizuku) | 有 | 无 | 架构差距大，非必需 |
| onServiceConnected 启动前台服务 | autoStart() | 已修复（P0） | 已对齐 |

## 3. 可借鉴的具体技术点

### 3.1 TYPE_ACCESSIBILITY_OVERLAY 1x1 保活窗口（低成本，高收益）

OpenSwipe 已有 overlay 窗口用于手势触发区，但可以额外增加一个最小化的保活窗口，确保即使手势区域被移除时进程仍受保护。

### 3.2 WRITE_SECURE_SETTINGS 自修复能力（高成本，最高收益）

这是 GKD 保活的核心武器。实现路径：
1. 引导用户通过 ADB 授予 `WRITE_SECURE_SETTINGS`
2. 获得此权限后，App 可以直接操作 `Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES`
3. 检测到无障碍断开时（通过 Flow 监听），自动重新注册
4. 提供快捷磁贴让用户一键恢复

**注意**: 这需要用户执行一次 ADB 命令或安装 Shizuku，门槛较高，适合作为"高级选项"。

### 3.3 前台服务通知内容动态化

GKD 的 StatusService 通知内容会实时更新（显示规则匹配数、当前状态等），让用户感知服务在运行。OpenSwipe 可以在通知中显示"手势服务运行中 | 已执行 N 次操作"等信息。

### 3.4 AppOps 权限主动检测

Android 14+ 系统可能会在用户不知情的情况下限制前台服务权限。GKD 主动检测 `OP_FOREGROUND_SERVICE_SPECIAL_USE` 等 AppOps 状态，在被限制时立即提示用户。

### 3.5 快捷磁贴（低成本，体验好）

提供 Quick Settings Tile，下拉通知栏即可看到服务状态并一键开关。

## 4. 推荐修改方案

### Phase 1: 低成本高收益（1-2 天）

**P1-1: 增加 1x1 保活 overlay 窗口**

在 `GestureAccessibilityService.onServiceConnected()` 中添加：
```kotlin
private var aliveView: View? = null

// 在 onServiceConnected 中
val aliveParams = WindowManager.LayoutParams().apply {
    type = WindowManager.LayoutParams.TYPE_ACCESSIBILITY_OVERLAY
    format = PixelFormat.TRANSLUCENT
    flags = FLAG_NOT_TOUCHABLE or FLAG_NOT_FOCUSABLE
    gravity = Gravity.START or Gravity.TOP
    width = 1; height = 1
}
aliveView = View(this)
windowManager.addView(aliveView, aliveParams)
```

**P1-2: foregroundServiceType 升级为 specialUse**

当前 KeepAliveService 可能使用的是旧类型。Android 14+ 要求声明 `specialUse` 并说明用途：
```xml
<service android:name=".service.KeepAliveService"
    android:foregroundServiceType="specialUse">
    <property
        android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
        android:value="Keep gesture service alive for edge gesture detection" />
</service>
```

**P1-3: AppOps 前台服务权限检测**

检测 `OP_FOREGROUND_SERVICE_SPECIAL_USE` 是否被限制（Android 14+），受限时提示用户。

### Phase 2: 中等投入（3-5 天）

**P2-1: Quick Settings Tile**

添加 `GestureTileService` 显示服务状态，点击可切换开关。

**P2-2: 电池优化白名单引导 UI**

首次启用服务后弹窗引导关闭电池优化，使用 `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`。

**P2-3: 通知内容动态化**

StatusService 通知显示运行状态信息，如手势执行次数。

### Phase 3: 高级功能（可选，1-2 周）

**P3-1: WRITE_SECURE_SETTINGS 支持**

- 添加设置页面说明如何通过 ADB 授权
- 获得权限后实现无障碍自动修复
- 提供命令行一键授权脚本

**P3-2: Shizuku 集成**

作为 ADB 授权的替代方案，通过 Shizuku 获取 WRITE_SECURE_SETTINGS。

## 5. 核心结论

GKD 的保活策略可以总结为三层防御：

1. **第一层 - 被动防御**: overlay 保活窗口 + 前台服务通知 -> 提高进程优先级，减少被杀概率
2. **第二层 - 主动检测**: AppOps 权限监控 + 电池优化白名单 -> 发现问题主动提示用户
3. **第三层 - 自动恢复**: WRITE_SECURE_SETTINGS + 快捷磁贴 -> 被杀后能自行或一键恢复

OpenSwipe 当前仅部分实现了第一层（已修复 P0 后）。建议按 Phase 1 -> Phase 2 -> Phase 3 逐步对齐。其中 Phase 1 的三项修改投入最小但能覆盖大部分场景。

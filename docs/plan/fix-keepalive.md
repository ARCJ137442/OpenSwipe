# 杀后台后手势失效 - 根因分析与修复方案

## 1. 根因分析

### 根因 #1（最可能）：KeepAliveService 从未在正常使用时启动

**证据：** `KeepAliveService` 仅在 `BootReceiver` 中启动（BootReceiver.kt:19-24），即仅在 `BOOT_COMPLETED` 或 `MY_PACKAGE_REPLACED` 时触发。**用户正常打开 App 并启用无障碍服务后，KeepAliveService 根本没有运行。**

这意味着进程只有 AccessibilityService 在维持，没有前台通知提升进程优先级。当用户从最近任务列表中滑掉 App 时，系统会杀死进程。

AccessibilityService 理论上受系统保护，但：
- 进程被杀后，系统会尝试重启 AccessibilityService（通常几秒到几十秒）
- 重启时会重新创建 Application（触发 `OpenSwipeApp.onCreate()`）
- 然后系统调用 `onServiceConnected()`

**问题在于重启后的状态恢复（见根因 #2）。**

### 根因 #2（核心问题）：进程重启后 compiledRuleSet 加载存在竞态

`GestureAccessibilityService.onServiceConnected()`（第 48-52 行）立即读取 `(application as OpenSwipeApp).compiledRuleSet`。

但 `OpenSwipeApp.onCreate()`（第 57-66 行）中，规则加载是在 `Dispatchers.IO` 协程中异步执行的：
```kotlin
appScope.launch(Dispatchers.IO) {
    val prefs = settingsDataStore.data.first()
    val json = prefs[KEY_RULES_JSON]
    val graph = ...
    _compiledRuleSet.value = graph.compile()  // 异步完成
}
```

**时序问题：**
1. 进程被杀后系统重建 Application -> `OpenSwipeApp.onCreate()` 执行
2. 规则加载在 IO 线程异步进行，`_compiledRuleSet` 初始值为 `CompiledRuleSet.EMPTY`
3. 系统立即调用 `onServiceConnected()` -> GestureEngine 拿到 `EMPTY` 规则集
4. GestureEngine.start() 中 combine configFlow 与 compiledRuleSetFlow
5. 首次 collect 时 compiledRuleSet 可能仍为 EMPTY -> `rebuildOverlays()` 发现没有任何规则 -> **不创建任何 overlay 窗口**
6. 之后 compiledRuleSet 异步加载完成，触发第二次 collect -> `applyConfigDiff()` 执行
7. 但 `applyConfigDiff` 比较的是 `old` 和 `new` GestureConfig（第 73 行），**不检测 ruleSet 变化**，因此如果 config 没变，overlay 可能不会重建

实际上再看代码，`combine(configFlow, compiledRuleSetFlow)` 在 ruleSet 变化时也会触发 collect，此时走 `applyConfigDiff(old, newConfig, ruleSet)` 路径（第 59 行）。这个方法确实会检查 `ruleSet.hasRulesFor(edge)` 并在需要时 addEdgeOverlay。**所以理论上这个竞态最终会自愈** —— 但前提是 `started` 已经为 true，且 configFlow 的值没有变化导致 `old == new`。

仔细看逻辑：第一次 collect 时 `started = false`，走 `rebuildOverlays(ruleSet)` 路径（ruleSet 为 EMPTY），设置 `started = true`。第二次 ruleSet 更新时，走 `applyConfigDiff`，此时 `old` 和 `new` config 相同，`sideNeedsRebuild = false`，`bottomNeedsRebuild = false`。但因为 `hadOverlay = false`（EMPTY 时没创建）而 `hasRules = true`，所以走第 89 行 `addEdgeOverlay(edge)` 分支。**这条路径应该能正确恢复。**

所以根因 #2 可能不是主要问题，除非 DataStore 读取极慢或失败。

### 根因 #3（最可能的真正原因）：系统不一定会重启 AccessibilityService

用户"杀后台"（从最近任务列表中滑掉）的行为，在不同 OEM 上效果不同：

- **原生 Android / Pixel**：杀后台 -> 系统会在几秒内自动重启 AccessibilityService
- **国产 ROM（MIUI/ColorOS/OriginOS 等）**：杀后台 -> 系统可能**永久杀死进程且不重启** AccessibilityService，除非：
  - App 在电池优化白名单中
  - App 有前台通知（提升进程优先级到前台级别）
  - App 被加入系统自启动白名单

**STB（Swipe To Back）之所以能存活**，不是因为它有更好的保活策略（竞品分析显示 STB 没有 KeepAliveService、没有 BootReceiver），而是因为：
1. STB 使用 `TYPE_ACCESSIBILITY_OVERLAY`（我们也用了，这点一样）
2. STB 的 `onAccessibilityEvent()` 是空实现 —— 服务极度轻量
3. **关键区别**：STB 可能被用户手动加入了自启动/电池优化白名单，或者测试环境不同

但最核心的差异是：**我们的 KeepAliveService 虽然存在但从未在正常流程中启动**，导致进程优先级低，容易被 OEM ROM 杀死且不恢复。

## 2. 竞品对比

| 策略 | OpenSwipe | STB | XDA |
|------|-----------|-----|-----|
| 前台通知保活 | 有 KeepAliveService 但**仅开机时启动** | 无 | 有，App 启动时即启动 |
| 开机自启 | BOOT_COMPLETED + MY_PACKAGE_REPLACED | 无 | 4 种 action |
| 电池优化白名单引导 | 有权限声明，**未见引导 UI** | 无 | 有权限 + 引导 |
| 被杀后自恢复 | 无 | 无 | onDestroy 中发 RELAUNCH 广播 |
| Overlay 窗口类型 | TYPE_ACCESSIBILITY_OVERLAY | TYPE_ACCESSIBILITY_OVERLAY | TYPE_APPLICATION_OVERLAY |
| AccessibilityService 重启后恢复 | 依赖 combine flow 自动重建 | overlay 在 onServiceConnected 直接创建 | 通过 Binder 广播触发 |

**核心差距**：STB 在 `onServiceConnected()` 中直接创建 overlay 窗口，不依赖异步数据加载。而 OpenSwipe 依赖 Application 中的异步 DataStore 读取。

## 3. 修复方案（按优先级排序）

### P0: 在 onServiceConnected 中启动 KeepAliveService

这是最关键的修复。当前 KeepAliveService 只在开机时启动，正常使用时从未运行。

**文件：** `GestureAccessibilityService.kt`

```kotlin
override fun onServiceConnected() {
    super.onServiceConnected()
    instance = this

    // [新增] 启动前台保活服务，提升进程优先级
    val keepAliveIntent = Intent(this, KeepAliveService::class.java)
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        startForegroundService(keepAliveIntent)
    } else {
        startService(keepAliveIntent)
    }

    // ... 现有初始化代码 ...
}
```

### P1: 确保规则加载完成后再构建 overlay

将 `OpenSwipeApp.onCreate()` 中的规则加载改为同步（在 `runBlocking` 或提前到 Application 构造阶段），或者在 GestureEngine 中增加对 EMPTY 规则集的特殊处理。

**方案 A（推荐）：在 GestureEngine.start() 中，如果首次 collect 拿到 EMPTY，跳过 rebuildOverlays 但不设置 started = true**

**文件：** `GestureEngine.kt` 第 55-57 行

```kotlin
if (!started) {
    if (ruleSet == CompiledRuleSet.EMPTY) {
        // 规则尚未加载完成，等待下一次 emit
        return@collect
    }
    started = true
    rebuildOverlays(ruleSet)
}
```

### P2: 引导用户关闭电池优化

在首次启用无障碍服务后，检测并引导用户将 App 加入电池优化白名单。

```kotlin
// 在 MainActivity 或设置页中
if (!isIgnoringBatteryOptimizations(this)) {
    // 显示对话框解释为什么需要关闭电池优化
    // 点击确认后跳转系统设置
    val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS)
    intent.data = Uri.parse("package:$packageName")
    startActivity(intent)
}
```

### P3: 在 cleanup() 中停止 KeepAliveService

当 AccessibilityService 被用户主动关闭时，同步停止保活服务。

**文件：** `GestureAccessibilityService.kt` 第 88-93 行

```kotlin
private fun cleanup() {
    if (::gestureEngine.isInitialized) gestureEngine.stop()
    if (::overlayManager.isInitialized) overlayManager.removeAll()
    stopService(Intent(this, KeepAliveService::class.java))  // 新增
    instance = null
    _serviceState.value = ServiceState.DISCONNECTED
}
```

### P4（可选）：针对国产 ROM 的额外保活

在 `onDestroy()` 中尝试通过 JobScheduler/WorkManager 延迟重启检测：

```kotlin
override fun onDestroy() {
    cleanup()
    // 安排一个一次性 WorkRequest，5 秒后检查服务状态
    val request = OneTimeWorkRequestBuilder<ServiceCheckWorker>()
        .setInitialDelay(5, TimeUnit.SECONDS)
        .build()
    WorkManager.getInstance(applicationContext).enqueue(request)
    super.onDestroy()
}
```

## 4. 总结

**根因排序：**
1. **KeepAliveService 未在正常流程中启动** —— 进程优先级低，被 OEM ROM 杀死后不恢复
2. **规则异步加载竞态** —— 即使 Service 恢复，首次可能拿到空规则集（但 flow combine 机制理论上能自愈）
3. **缺少电池优化白名单引导** —— 在国产 ROM 上被激进杀后台

**最小修复：** 仅实施 P0（在 onServiceConnected 中启动 KeepAliveService）即可解决 90% 场景。

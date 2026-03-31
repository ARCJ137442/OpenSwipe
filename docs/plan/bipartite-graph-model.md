# 二部图手势模型：架构设计文档

> 版本：v1.0 | 日期：2026-03-31
> 状态：设计阶段

---

## 1. 核心思想

OpenSwipe 的手势配置本质上是一张**二部图（Bipartite Graph）**：

- **左侧**：触发节点（Trigger Nodes）— 用户在屏幕边缘做了什么
- **右侧**：动作节点（Action Nodes）— 系统执行什么操作
- **边**：规则（Rules）— 触发到动作的映射

```
    触发节点 (T)                    动作节点 (A)
    ─────────                      ─────────

    ┌──────────────┐          ┌──────────────┐
    │ 左·全段·短滑  │─────────→│ 返回         │
    └──────────────┘     ┌───→└──────────────┘
    ┌──────────────┐     │
    │ 右·全段·短滑  │─────┘    ┌──────────────┐
    └──────────────┘     ┌───→│ 主页         │
    ┌──────────────┐     │    └──────────────┘
    │ 左·全段·长滑  │──┐  │
    └──────────────┘  │  │    ┌──────────────┐
    ┌──────────────┐  └──┼───→│ 切换上个App   │
    │ 右·全段·长滑  │─────┘    └──────────────┘
    └──────────────┘
    ┌──────────────┐          ┌──────────────┐
    │ 底·左1/3·上滑 │─────────→│ 返回         │
    └──────────────┘          └──────────────┘
    ┌──────────────┐
    │ 底·中1/3·上滑 │─────────→  主页
    └──────────────┘
    ┌──────────────┐
    │ 底·右1/3·上滑 │─────────→  最近任务
    └──────────────┘
```

这不是「信号网络」（节点间无互连），而是一个纯粹的偏函数 `f: T ⇀ A`。

---

## 2. 拓扑约束

| 约束 | 含义 | 原因 |
|------|------|------|
| **T → A 是函数** | 每个触发节点**至多连一条边** | 否则同一个手势触发两个动作，产生歧义 |
| **多对一允许** | 多个触发可以连到同一个动作 | 左右短滑都→返回，完全合理 |
| **T 可以悬空** | 未配置的触发 = 该位置该手势不响应 | 用户选择性启用 |
| **A 可以悬空** | 没有触发连过来的动作 = 闲置 | 动作池是完整枚举，不必全部启用 |
| **T 不互连** | 触发节点之间无边 | 每个触发独立判定 |
| **A 不互连** | 动作节点之间无边 | 暂不支持宏/链式动作（可作为未来扩展） |

---

## 3. 节点定义

### 3.1 触发节点 (TriggerNode)

三元组 `(Edge, Section, GestureType)` 的**具体实例**：

```kotlin
data class TriggerNode(
    val edge: Edge,
    val section: SectionRange,
    val gestureType: GestureType
)

enum class Edge { LEFT, RIGHT, BOTTOM }

enum class GestureType {
    SHORT_SWIPE,    // 短距离滑动
    LONG_SWIPE,     // 长距离滑动
    // 未来扩展：
    // TAP, DOUBLE_TAP, LONG_PRESS, FLING, L_SHAPE
}
```

**T 节点集合是动态的** — 用户创建规则时才实例化触发节点。初始状态下 T 集合为空，由预设方案或用户手动添加。

### 3.2 区段范围 (SectionRange)

区段用**比例**表示，而非像素。

> **设计决策**：由于横屏/竖屏自动旋转的存在，屏幕像素尺寸在运行时会变化。
> 因此区段范围始终用 `[0.0, 1.0]` 比例存储，像素换算延迟到运行时实际匹配的瞬间完成。
> 编译阶段的产物也是**比例**，不预计算像素。

```kotlin
data class SectionRange(
    val start: Float = 0f,   // 0.0 = 边缘起始端（左边缘的顶部/底部边缘的左端）
    val end: Float = 1f      // 1.0 = 边缘末端
) {
    init {
        require(start in 0f..1f) { "start must be in [0, 1]" }
        require(end in 0f..1f) { "end must be in [0, 1]" }
        require(start < end) { "start must < end" }
    }

    fun contains(position: Float) = position in start..end

    fun overlapsWith(other: SectionRange): Boolean {
        return start < other.end && other.start < end
    }

    companion object {
        val ALL = SectionRange(0f, 1f)
        fun thirds(index: Int) = SectionRange(index / 3f, (index + 1) / 3f)
        fun halves(index: Int) = SectionRange(index / 2f, (index + 1) / 2f)
        fun nths(index: Int, n: Int) = SectionRange(index.toFloat() / n, (index + 1).toFloat() / n)
    }
}
```

**比例语义约定**：

| 边缘 | 0.0 端 | 1.0 端 | 说明 |
|------|--------|--------|------|
| LEFT | 屏幕顶部 | 屏幕底部 | 沿左边缘从上到下 |
| RIGHT | 屏幕顶部 | 屏幕底部 | 沿右边缘从上到下 |
| BOTTOM | 屏幕左侧 | 屏幕右侧 | 沿底部从左到右 |

横屏旋转后，这些方向自动跟随屏幕坐标系，无需额外处理。

### 3.3 动作节点 (ActionNode)

**静态枚举**，App 出厂定义好全量。每个节点自描述，UI 可直接渲染。

```kotlin
sealed interface ActionNode {
    val id: String        // 持久化标识，稳定不变
    val label: String     // 用户可见的名称
    val minApi: Int       // 最低 Android API 要求（低于此版本不可选）

    // ═══ 系统导航 ═══
    data object Back : ActionNode {
        override val id = "back"
        override val label = "返回"
        override val minApi = 16
    }
    data object Home : ActionNode {
        override val id = "home"
        override val label = "主页"
        override val minApi = 16
    }
    data object Recents : ActionNode {
        override val id = "recents"
        override val label = "最近任务"
        override val minApi = 16
    }
    data object SwitchLastApp : ActionNode {
        override val id = "switch_last_app"
        override val label = "切换上个应用"
        override val minApi = 16  // 通过 RECENTS×2 技巧实现
    }

    // ═══ 系统控制 ═══
    data object LockScreen : ActionNode {
        override val id = "lock_screen"
        override val label = "锁屏"
        override val minApi = 28
    }
    data object Screenshot : ActionNode {
        override val id = "screenshot"
        override val label = "截屏"
        override val minApi = 28
    }
    data object SplitScreen : ActionNode {
        override val id = "split_screen"
        override val label = "分屏"
        override val minApi = 24
    }
    data object PowerMenu : ActionNode {
        override val id = "power_menu"
        override val label = "电源菜单"
        override val minApi = 21
    }

    // ═══ 面板 ═══
    data object NotificationPanel : ActionNode {
        override val id = "notification_panel"
        override val label = "通知栏"
        override val minApi = 16
    }
    data object QuickSettings : ActionNode {
        override val id = "quick_settings"
        override val label = "快速设置"
        override val minApi = 17
    }

    // ═══ 媒体控制 ═══
    data object MediaPlayPause : ActionNode {
        override val id = "media_play_pause"
        override val label = "播放/暂停"
        override val minApi = 16
    }
    data object MediaNext : ActionNode {
        override val id = "media_next"
        override val label = "下一曲"
        override val minApi = 16
    }
    data object MediaPrevious : ActionNode {
        override val id = "media_previous"
        override val label = "上一曲"
        override val minApi = 16
    }
    data object VolumeUp : ActionNode {
        override val id = "volume_up"
        override val label = "音量+"
        override val minApi = 16
    }
    data object VolumeDown : ActionNode {
        override val id = "volume_down"
        override val label = "音量-"
        override val minApi = 16
    }

    // ═══ 硬件 ═══
    data object ToggleFlashlight : ActionNode {
        override val id = "toggle_flashlight"
        override val label = "开关闪光灯"
        override val minApi = 23
    }

    // ═══ 应用启动 ═══
    data class LaunchApp(
        val packageName: String,
        val appName: String
    ) : ActionNode {
        override val id = "launch_app:$packageName"
        override val label = appName
        override val minApi = 16
    }

    // ═══ 空操作 ═══
    data object NoAction : ActionNode {
        override val id = "no_action"
        override val label = "无操作"
        override val minApi = 16
    }
}
```

**A 集合特征**：
- 固定的系统动作约 16 种（随 API 版本过滤可选集）
- `LaunchApp` 是唯一的开放子类（可实例化无限多个）
- `NoAction` 用于显式禁用某个触发位置（与"悬空不配置"的语义不同：悬空=该区域触摸事件穿透到下层 App；NoAction=该区域消费事件但不执行动作）

---

## 4. 边 = 规则 (GestureRule)

```kotlin
data class GestureRule(
    val id: String = UUID.randomUUID().toString(),
    val trigger: TriggerNode,
    val action: ActionNode
)
```

---

## 5. 图 = 规则集合 (GestureRuleGraph)

```kotlin
data class GestureRuleGraph(
    val rules: List<GestureRule>
) {
    // ─── 验证 ───

    data class Conflict(
        val ruleA: GestureRule,
        val ruleB: GestureRule,
        val overlapSection: SectionRange
    )

    fun validate(): List<Conflict> {
        val conflicts = mutableListOf<Conflict>()
        for (i in rules.indices) {
            for (j in i + 1 until rules.size) {
                val a = rules[i]
                val b = rules[j]
                if (a.trigger.edge == b.trigger.edge
                    && a.trigger.gestureType == b.trigger.gestureType
                    && a.trigger.section.overlapsWith(b.trigger.section)
                ) {
                    conflicts.add(Conflict(a, b, /* overlap */))
                }
            }
        }
        return conflicts
    }

    // ─── 编译 ───

    fun compile(): CompiledRuleSet {
        // 见下文第 6 节
    }
}
```

---

## 6. 编译：图 → 运行时查找结构

### 6.1 编译产物

```kotlin
class CompiledRuleSet(
    private val table: Map<Edge, Map<GestureType, List<CompiledSection>>>
) {
    /**
     * 核心匹配方法。运行时热路径。
     *
     * @param edge 哪个边缘
     * @param gestureType 什么手势
     * @param sectionRatio 触摸位置在边缘上的比例 [0.0, 1.0]
     *        运行时由 touchPositionPx / edgeLengthPx 实时计算
     * @return 匹配到的动作，或 null（无匹配规则）
     */
    fun match(edge: Edge, gestureType: GestureType, sectionRatio: Float): ActionNode? {
        val sections = table[edge]?.get(gestureType) ?: return null
        // sections 已按 start 排序，二分查找
        for (section in sections) {
            if (sectionRatio < section.start) return null  // 已过范围
            if (sectionRatio <= section.end) return section.action
        }
        return null
    }
}

data class CompiledSection(
    val start: Float,        // 比例起始 [0.0, 1.0]
    val end: Float,          // 比例结束 [0.0, 1.0]
    val action: ActionNode
)
```

### 6.2 为什么编译产物用比例而非像素

| 方案 | 预编译像素 | 预编译比例 |
|------|-----------|-----------|
| 横竖屏旋转 | ❌ 需要重新编译 | ✅ 比例不变 |
| 分屏/小窗模式 | ❌ 需要重新编译 | ✅ 比例不变 |
| 运行时开销 | 查表 O(1) 但需监听配置变化 | 查表 O(log N) + 一次除法 |
| 正确性 | ⚠️ 屏幕尺寸变化时有窗口期 | ✅ 始终正确 |

**结论**：编译产物存比例。运行时做一次 `touchY / edgeHeight` 除法的成本可以忽略不计（相比触摸事件本身的开销），但换来了对旋转和窗口模式变化的天然免疫。

### 6.3 运行时匹配流程

```
触摸事件 (MotionEvent)
    ↓
EdgeGestureDetector 检测手势
    ↓ 输出 (gestureType)
计算 sectionRatio = touchAlongEdgePx / currentEdgeLengthPx
    ↓               ↑ 运行时实时获取，适应旋转
CompiledRuleSet.match(edge, gestureType, sectionRatio)
    ↓ 返回 ActionNode?
ActionDispatcher.dispatch(actionNode)
```

**像素 → 比例的转换只发生在一个点**：手势检测完成的瞬间。这个瞬间屏幕尺寸是确定的。

### 6.4 编译过程

```kotlin
fun GestureRuleGraph.compile(): CompiledRuleSet {
    val table = mutableMapOf<Edge, MutableMap<GestureType, MutableList<CompiledSection>>>()

    for (rule in rules) {
        val edge = rule.trigger.edge
        val gestureType = rule.trigger.gestureType

        table
            .getOrPut(edge) { mutableMapOf() }
            .getOrPut(gestureType) { mutableListOf() }
            .add(CompiledSection(
                start = rule.trigger.section.start,
                end = rule.trigger.section.end,
                action = rule.action
            ))
    }

    // 每组按 start 排序（使匹配时可以提前退出）
    for ((_, byGesture) in table) {
        for ((_, sections) in byGesture) {
            sections.sortBy { it.start }
        }
    }

    return CompiledRuleSet(table)
}
```

编译复杂度：O(N log N)（排序主导），N = 规则数。
运行时匹配复杂度：O(1) edge查找 + O(1) gestureType查找 + O(K) 区段遍历，K = 单边缘单手势的区段数（通常 ≤ 5）。

---

## 7. 与无障碍服务的绑定

二部图的两侧分别绑定到 Android 系统的不同 API 层：

```
┌─────────────────────────────┐        ┌─────────────────────────────┐
│       触发侧 (T)             │        │       动作侧 (A)             │
│                             │        │                             │
│  AccessibilityService       │        │  performGlobalAction()      │
│   → TYPE_ACCESSIBILITY      │ rules  │  dispatchMediaKeyEvent()    │
│     _OVERLAY 窗口权限        │───────→│  CameraManager              │
│                             │ match  │  Intent.startActivity()     │
│  MotionEvent 触摸事件        │        │  AudioManager               │
│   → EdgeGestureDetector     │        │  dispatchGesture()          │
│     手势识别                 │        │                             │
│   → (edge, type, ratio)     │        │                             │
└──────────────┬──────────────┘        └──────────────┬──────────────┘
               │                                      │
               └──── 都绑定到同一个 ────────────────────┘
                     AccessibilityService 实例
```

**解耦点**：触发侧只产出 `(Edge, GestureType, Float)` 三元组，不知道会执行什么动作；动作侧只接收 `ActionNode`，不知道由什么手势触发。二者通过 `CompiledRuleSet.match()` 这一个函数连接。

---

## 8. 图的操作 (CRUD)

### 8.1 添加规则

```
用户操作：选择边缘 → 选择区段 → 选择手势类型 → 选择动作
                                    ↓
                          validate(newTrigger)
                           ├── 无冲突 → 添加到 rules
                           └── 有冲突 → 提示用户
                                        ├── 覆盖已有规则
                                        ├── 调整区段范围
                                        └── 取消
```

### 8.2 删除规则

直接从 `rules` 中移除，无约束。

### 8.3 修改规则

等价于删除旧规则 + 添加新规则，需重新验证冲突。

### 8.4 应用规则

```
用户点击「应用」
    ↓
validate() → 如果有冲突则阻止
    ↓ 通过
compile() → 生成 CompiledRuleSet
    ↓
OverlayManager.rebuild() → 根据规则中涉及的边缘重建 overlay 窗口
    ↓
缓存 CompiledRuleSet 到 DataStore
    ↓ 下次启动
直接加载缓存（附带 rulesHash 校验）
```

---

## 9. 持久化

### 9.1 规则表存储

规则表序列化为 JSON，存入 DataStore 或文件：

```json
{
  "version": 1,
  "rules": [
    {
      "trigger": { "edge": "LEFT", "section": [0.0, 1.0], "gesture": "SHORT_SWIPE" },
      "action": { "id": "back" }
    },
    {
      "trigger": { "edge": "BOTTOM", "section": [0.0, 0.33], "gesture": "SHORT_SWIPE" },
      "action": { "id": "back" }
    },
    {
      "trigger": { "edge": "BOTTOM", "section": [0.33, 0.67], "gesture": "SHORT_SWIPE" },
      "action": { "id": "home" }
    }
  ]
}
```

### 9.2 编译缓存

```kotlin
data class CompiledCache(
    val rulesHash: Int,             // rules.hashCode()
    val compiledRuleSet: CompiledRuleSet,
    val compiledAt: Long
)

// 启动流程：
// 1. 加载规则 JSON → 计算 hash
// 2. 加载缓存 → 比对 hash
//    match  → 直接使用缓存的 CompiledRuleSet
//    mismatch → 重新 compile() → 更新缓存
```

### 9.3 预设方案

```kotlin
object Presets {
    val IOS_STYLE = GestureRuleGraph(listOf(
        GestureRule(TriggerNode(LEFT, ALL, SHORT_SWIPE), Back),
        GestureRule(TriggerNode(RIGHT, ALL, SHORT_SWIPE), Back),
        GestureRule(TriggerNode(BOTTOM, ALL, SHORT_SWIPE), Home),
    ))

    val ANDROID_CLASSIC = GestureRuleGraph(listOf(
        GestureRule(TriggerNode(BOTTOM, thirds(0), SHORT_SWIPE), Back),
        GestureRule(TriggerNode(BOTTOM, thirds(1), SHORT_SWIPE), Home),
        GestureRule(TriggerNode(BOTTOM, thirds(2), SHORT_SWIPE), Recents),
    ))

    val MEDIA_CONTROL = GestureRuleGraph(listOf(
        GestureRule(TriggerNode(BOTTOM, thirds(0), SHORT_SWIPE), MediaPrevious),
        GestureRule(TriggerNode(BOTTOM, thirds(1), SHORT_SWIPE), MediaPlayPause),
        GestureRule(TriggerNode(BOTTOM, thirds(2), SHORT_SWIPE), MediaNext),
    ))
}
```

用户可以选择预设作为起点，然后在此基础上自定义修改。

### 9.4 导入/导出

规则表的 JSON 格式即为导入/导出格式。用户可以：
- 导出当前配置为 JSON 文件
- 从文件或剪贴板导入 JSON
- 未来：在社区分享配置方案

---

## 10. 未来扩展点

本架构为以下功能预留了扩展空间，但 Phase 1 不实现：

| 扩展 | 影响 | 改动范围 |
|------|------|---------|
| 新 Edge（TOP） | TriggerNode.edge 加枚举值 | 小 |
| 新 GestureType（TAP, LONG_PRESS） | TriggerNode.gestureType 加枚举值 + EdgeGestureDetector 加检测逻辑 | 中 |
| 按应用规则 | TriggerNode 增加 `context: AppContext?` 字段，编译时多一层索引 | 中 |
| 横屏独立配置 | TriggerNode 增加 `orientation: Orientation?` 字段 | 小 |
| 复合手势 | TriggerNode 变为 `List<TriggerStep>`（手势序列） | 大 |
| 动作宏 | ActionNode 增加 `Sequence(actions: List<ActionNode>)` 子类 | 中 |
| 左右宽度可调 | GestureRuleGraph 增加 `edgeConfig: Map<Edge, EdgeConfig>` | 小 |

所有扩展都是**在现有二部图模型上增加维度**，不需要改变核心的图拓扑结构和编译/匹配机制。

---

## 11. 实施计划

| 阶段 | 内容 | 产出 |
|------|------|------|
| **Phase 1** | TriggerNode + ActionNode + GestureRule + GestureRuleGraph 数据模型 | 核心类型定义 |
| **Phase 1** | RuleValidator 冲突检测 | 编辑时验证 |
| **Phase 1** | compile() + CompiledRuleSet + match() | 编译和匹配引擎 |
| **Phase 1** | 替换现有 GestureConfig 硬编码映射 | 运行时集成 |
| **Phase 1** | 规则表 JSON 序列化/反序列化 | 持久化 |
| **Phase 2** | 规则编辑 UI（规则列表 + 动作选择器） | 用户可配置 |
| **Phase 2** | 预设方案 + 一键切换 | 小白友好 |
| **Phase 2** | 导入/导出 JSON | 社区分享 |
| **Phase 3** | 可视化手机轮廓配置 | 高级 UI |
| **Phase 3** | 按应用/横屏等上下文扩展 | 高级功能 |

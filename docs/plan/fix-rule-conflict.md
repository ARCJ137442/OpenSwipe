# 规则冲突分析与修复方案

## 问题概述

系统中存在两套并行的配置机制，导致主页开关覆盖规则编辑器的配置。

---

## 分析结果

### Q1: 是否存在两套并行配置机制？

**是。** 明确存在两套：

| 机制 | 数据源 | 控制内容 |
|------|--------|----------|
| 旧：`GestureConfig` | DataStore preferences (`edge_left_enabled` 等 key) | 边缘开关、底部触发高度、触发模式 |
| 新：`CompiledRuleSet` | DataStore (`gesture_rules_json` key) → `GestureRuleGraph` → `compile()` | 每条边、每种手势、每个区段的动作映射 |

两者共存于 `OpenSwipeApp`，分别通过 `gestureConfigFlow` 和 `compiledRuleSet` 暴露。

### Q2: GestureEngine 实际监听的是哪个？

**两个都监听，但职责不同，且旧配置拥有"否决权"：**

- `GestureEngine.start()` 中 `configFlow.collect` 控制 **overlay 窗口的创建与销毁**（第 39-49 行）。当 `GestureConfig.leftEnabled = false` 时，左侧 overlay 直接被移除，**根本不会产生触摸事件**，规则编辑器中对左侧的任何规则都不会生效。
- `handleGestureResult()` 中先查 `compiledRuleSetFlow`（新规则），匹配不到才 fallback 到 `mapResultToAction()`（旧硬编码映射）。

**核心冲突**：旧 `GestureConfig` 的 `leftEnabled/rightEnabled/bottomEnabled` 开关在 overlay 层面就决定了边缘是否存在。即使规则编辑器配置了该边缘的规则，如果旧开关关闭，overlay 不存在，手势根本不会被检测到。

### Q3: HomeScreen 的开关写入是否影响了 GestureEngine 的行为？

**是，而且是根本性的影响。**

数据流路径：
```
HomeScreen Switch
  → HomeViewModel.setLeftEnabled()
    → OpenSwipeApp.updateEdgeEnabled()
      → DataStore edit (KEY_LEFT_ENABLED)
        → gestureConfigFlow 更新
          → GestureEngine.configFlow.collect
            → applyConfigDiff() → removeEdge() 移除 overlay
```

关闭任何一个开关，对应边缘的 overlay 窗口会被直接移除，**完全绕过了规则编辑器的配置**。

### Q4: 规则编辑器的「应用规则」是否真正传递到了 GestureEngine？

**没有。** `RuleConfigViewModel.applyRules()` 第 77-81 行：

```kotlin
fun applyRules() {
    if (_conflicts.value.isNotEmpty()) return
    _appliedRules.value = _rules.value.toList()
    // TODO: compile and push to AccessibilityService via GestureRuleGraph.compile()
}
```

这个 TODO 从未实现。`applyRules()` 仅更新了 ViewModel 内部的 `_appliedRules`，**没有调用 `OpenSwipeApp.applyRules(graph)`**。因此：
- 规则编辑器的修改永远不会持久化到 DataStore
- 规则编辑器的修改永远不会更新 `_compiledRuleSet`
- GestureEngine 读到的 `compiledRuleSetFlow` 始终是启动时从 DataStore 加载的旧值（或默认 Presets.DEFAULT）

---

## 冲突根源总结

1. **applyRules() 是空壳**：RuleConfigViewModel 的 TODO 未实现，规则编辑器的配置从未到达 GestureEngine。
2. **旧开关拥有否决权**：GestureConfig 的 enabled 开关控制 overlay 的存在，优先级高于一切规则。
3. **两套机制语义重叠**：旧的"关闭左边缘"和新的"左边缘没有任何规则"表达同一件事，但走不同路径。

---

## 修复方案

### Phase 1: 接通规则编辑器（必须先做）

**修改 `RuleConfigViewModel.applyRules()`**，调用 `OpenSwipeApp.applyRules()`：

```kotlin
fun applyRules() {
    if (_conflicts.value.isNotEmpty()) return
    val graph = GestureRuleGraph(rules = _rules.value)
    _appliedRules.value = _rules.value.toList()
    viewModelScope.launch {
        (getApplication<Application>() as OpenSwipeApp).applyRules(graph)
    }
}
```

### Phase 2: 统一配置源，移除旧开关

**目标**：让 `CompiledRuleSet` 成为唯一的"边缘是否启用"判据。

1. **修改 `GestureEngine.createOverlayWindows()`**：不再读 `currentConfig.leftEnabled` 等布尔值，改为根据 `compiledRuleSetFlow.value` 中该边缘是否有规则来决定是否创建 overlay。

```kotlin
private fun createOverlayWindows() {
    val ruleSet = compiledRuleSetFlow.value
    for (edge in Edge.entries) {
        if (ruleSet.hasRulesFor(edge)) {
            addEdgeOverlay(edge)
        }
    }
}
```

2. **让 GestureEngine 同时 collect compiledRuleSetFlow**，当规则变化时重建 overlay：

```kotlin
fun start() {
    scope.launch {
        combine(configFlow, compiledRuleSetFlow) { config, ruleSet -> config to ruleSet }
            .collect { (newConfig, ruleSet) ->
                currentConfig = newConfig
                rebuildOverlays(ruleSet)
            }
    }
}
```

3. **从 HomeScreen 移除三个独立开关**（或将它们改为快捷操作，直接启用/禁用对应边缘的所有规则）。

4. **从 GestureConfig 移除 `leftEnabled/rightEnabled/bottomEnabled`**，仅保留物理参数（triggerWidth、triggerHeight、triggerMode、灵敏度等）。

5. **在 `CompiledRuleSet` 中添加 `hasRulesFor(edge: Edge): Boolean`** 辅助方法。

### Phase 3: 移除 fallback 硬编码映射

`GestureEngine.mapResultToAction()` 是遗留代码。当 Phase 1 完成后，所有动作都应由 `CompiledRuleSet` 提供。移除 `mapResultToAction()` 和 `handleGestureResult()` 中的 fallback 分支。

---

## 推荐实施顺序

| 步骤 | 改动 | 风险 |
|------|------|------|
| 1 | 接通 `RuleConfigViewModel.applyRules()` → `OpenSwipeApp.applyRules()` | 低，纯接线 |
| 2 | 在 `CompiledRuleSet` 添加 `hasRulesFor()` | 低 |
| 3 | `GestureEngine` 同时 collect `compiledRuleSetFlow`，根据规则决定 overlay | 中，需测试 overlay 重建逻辑 |
| 4 | HomeScreen 开关改为规则快捷操作或移除 | 中，UI 变更 |
| 5 | 清理 `GestureConfig` 中的 enabled 字段和 `mapResultToAction()` | 低，纯删除 |

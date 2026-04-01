# OpenSwipe 开源合规审核报告

**审核日期**: 2026-03-31
**项目**: [ARCJ137442/OpenSwipe](https://github.com/ARCJ137442/OpenSwipe)
**目标许可证**: MIT

---

## 1. 许可证

| 项目 | 状态 | 说明 |
|------|------|------|
| LICENSE 文件 | ❌ 阻塞 | **LICENSE 文件不存在**。仓库根目录缺少 LICENSE 文件，GitHub 无法识别许可证类型。必须添加 MIT LICENSE 文件。 |
| 源文件头声明 | ✅ 通过 | MIT 许可证不强制要求源文件头声明，当前无文件头是可接受的。 |
| 第三方依赖兼容性 | ✅ 通过 | 所有依赖均为 Apache 2.0 许可（AndroidX、Jetpack Compose、Kotlin Coroutines、DataStore），均与 MIT 兼容。无 GPL/AGPL 依赖。 |

## 2. 敏感信息泄露

| 项目 | 状态 | 说明 |
|------|------|------|
| `.gitignore` 排除 `local.properties` | ✅ 通过 | `.gitignore` 中包含 `local.properties`（出现三次），且该文件未被 git 跟踪。 |
| 硬编码 API Key/密钥/Token | ✅ 通过 | 源码中未发现任何 API Key、Secret、Token 或密码。 |
| 个人信息 | ✅ 通过 | 源码中未发现邮箱、手机号等个人信息。 |
| Git 历史泄露 | ✅ 通过 | 历史提交中无敏感文件的添加或删除记录。 |
| `google-services.json` | ✅ 通过 | 文件不存在，项目不使用 Firebase。 |

## 3. 代码清洁度

| 项目 | 状态 | 说明 |
|------|------|------|
| 调试代码 (`Log.d`/`println`) | ✅ 通过 | 源码中未发现 `Log.d`、`println`、`System.out` 等调试输出。 |
| 注释掉的大段代码 | ✅ 通过 | 未发现大段注释代码。 |
| FIXME/HACK/TODO 注释 | ✅ 通过 | 源码中无 TODO、FIXME、HACK、XXX 注释。 |
| 包名一致性 | ✅ 通过 | `namespace`、`applicationId`、`package` 均为 `com.openswipe`，一致。 |

## 4. 文档完整性

| 项目 | 状态 | 说明 |
|------|------|------|
| README.md（中文） | ✅ 通过 | 完整，包含功能列表、隐私承诺、安装说明、技术架构。 |
| README-en.md（英文） | ✅ 通过 | 完整，与中文版内容对应。 |
| LICENSE 文件 | ❌ 阻塞 | 缺失（同第 1 项）。 |
| CONTRIBUTING.md | ⚠️ 建议改进 | 缺失。建议添加贡献指南，说明如何提 Issue、PR 流程、代码风格等。 |
| CHANGELOG.md | ⚠️ 建议改进 | 缺失。建议添加变更日志，方便用户了解版本更新内容。 |

## 5. 构建可复现性

| 项目 | 状态 | 说明 |
|------|------|------|
| `gradlew` + `gradle-wrapper.jar` | ✅ 通过 | 两者均已提交到 git。 |
| Gradle 版本固定 | ✅ 通过 | `gradle-wrapper.properties` 固定为 `gradle-8.11.1-bin.zip`。 |
| 依赖版本固定 | ✅ 通过 | `libs.versions.toml` 中所有版本均为精确版本号，无 `+` 动态版本。 |
| Clone 后可构建 | ✅ 通过 | 项目结构标准，新用户只需配置 Android SDK 路径即可 `./gradlew assembleDebug`。 |

## 6. GitHub 仓库配置

| 项目 | 状态 | 说明 |
|------|------|------|
| 仓库描述 | ✅ 通过 | 已设置中英双语描述。 |
| Topics/Tags | ❌ 阻塞 | **未设置任何 Topic**。建议添加：`android`, `gesture-navigation`, `accessibility`, `kotlin`, `jetpack-compose`, `open-source`。缺少 Topics 会严重影响项目在 GitHub 的可发现性。 |
| Issue Template | ⚠️ 建议改进 | 缺少 `.github/ISSUE_TEMPLATE`。建议添加 Bug 报告和功能请求模板。 |

## 7. 安全性

| 项目 | 状态 | 说明 |
|------|------|------|
| ProGuard/R8 混淆 | ⚠️ 建议改进 | Release 构建 `isMinifyEnabled = false`，未启用代码混淆。开源项目可接受，但发布 APK 时建议启用以减小体积。 |
| 权限声明 | ⚠️ 建议改进 | `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` 权限较敏感，Google Play 可能要求说明理由。建议在 README 中说明此权限的用途（保活服务）。其余权限（VIBRATE、FOREGROUND_SERVICE、RECEIVE_BOOT_COMPLETED、POST_NOTIFICATIONS）对手势导航应用合理。 |
| AccessibilityService 权限范围 | ⚠️ 建议改进 | `canRetrieveWindowContent="true"` 已启用但 `canPerformGestures="false"`。建议确认是否真的需要读取窗口内容——如果只需要检测手势触发，设为 `false` 可进一步最小化权限。 |

---

## 审核总结

### 阻塞项（必须修复）

1. **添加 LICENSE 文件** — 在项目根目录添加 MIT License 全文。没有 LICENSE 文件，代码在法律上默认为"保留所有权利"，与开源意图矛盾。
2. **设置 GitHub Topics** — 添加相关标签以提高项目可发现性。

### 建议改进项

3. 添加 CONTRIBUTING.md 贡献指南
4. 添加 CHANGELOG.md 变更日志
5. 添加 `.github/ISSUE_TEMPLATE/` 模板
6. 在 README 中说明 `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` 权限用途
7. 评估 `canRetrieveWindowContent` 是否可设为 `false`
8. Release 构建考虑启用 R8 混淆

### 最终结论

## **NOT READY**

需修复 2 个阻塞项后方可视为开源就绪。其中 LICENSE 文件是最关键的阻塞项。

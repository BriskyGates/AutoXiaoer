# 悬浮窗在长时间/切应用时「消失又出现」问题说明

## 现象

部分流程在「思考较久」或已切到其它应用（例如微信）执行耗时段时，**悬浮窗会消失**；回到 AutoXiaoer（小二）后，**悬浮窗又出现**。

本文结合当前代码说明**可能原因**与**如何验证**，便于后续优化或排障。

---

## 1. 截图流程会主动移除悬浮窗（占用时间与截屏一致）

**位置**：`ScreenshotService.capture()`

- 截屏前在主线程调用 `FloatingWindowController.hide()`，避免悬浮窗进入画面。
- 整段 `captureScreen()`（`screencap`、解码、压缩等）在 `finally` 里才 `showAndBringToFront()` 恢复。

因此：**只要 Agent 循环里仍在截屏，在每次截屏完成前悬浮窗都会处于被移除状态**。若 `screencap` 阻塞或很慢（工程内还有与唤醒/超时相关的逻辑），用户会感到「想很久、悬浮窗一直不在」——这往往与「截屏窗口期」重叠，而不一定是业务状态机错误。

---

## 2. 触摸/输入类动作会反复 hide / show

**位置**：`ActionHandler`（如 `executeTap`、`executeSwipe`、`executeType`、`executeLongPress`、`executeDoubleTap` 等）

- 操作前 `hideFloatingWindow()`，避免遮挡触摸。
- `finally` 中 `showFloatingWindow()`：仅约 `50ms` 延迟后调用 `show()`。

**对比**：`executeWait` 等纯等待**不会**走上述 hide，故若仅长时间等待而没有任何手势/截屏，按设计悬浮窗应仍可见；若仍消失，更可能来自截屏链或服务生命周期（见下）。

---

## 3. `show()` 与 `hide()` 均为异步，且 `show` 不等待贴窗完成

**位置**：`FloatingWindowService` 中 `hide()` / `show()` 在 `serviceScope.launch` 中执行。

- `ActionHandler.hideFloatingWindow()` 会轮询直到 `isVisible()` 为 false。
- `showFloatingWindow()` **没有**等待 `isVisible()` 再次为 true。

在截屏与手势交错、主线程繁忙时，可能出现短暂「还没贴上」的观感；一般不单独解释「直到回 App 才出现」，除非与进程被杀叠加。

---

## 4. 服务非前台保活，后台长时间可能被系统回收

**位置**：`FloatingWindowService.onStartCommand` 注释（不再依赖本 Service 的前台通知保活悬浮窗）。

长时间停留在其它应用（如微信）、内存紧张时，**普通 Service 可能被系统杀掉**，`onDestroy` 会移除窗口，悬浮窗随之消失。回到 AutoXiaoer 后，生命周期与 `FloatingWindowStateManager` / 任务逻辑可能再次启动服务并 `show()`，表现为「一进小二又有了」。

---

## 5. 悬浮窗状态机与「前后台」规则（与部分现象可能不一致）

**位置**：`FloatingWindowStateManager`

- `FORCED_VISIBLE`（任务执行中）：按设计在前台**不应**因 `onAppForeground` 被关掉。
- `VISIBLE_WHEN_BACKGROUND`（用户仅开启悬浮窗、非强制任务态）：**前台隐藏、后台显示**。

若用户观察是「微信里没了、回小二又有了」，与「仅 `VISIBLE_WHEN_BACKGROUND`」的直觉相反（那种配置下往往是前台藏、后台露）。因此若确认是上述现象，更应怀疑 **截屏隐藏时长、服务被杀、异步 show**，而不是单靠前后台状态机解释。

---

## 小结表

| 可能原因 | 代码/模块 |
|----------|-----------|
| 截屏期间窗口被移除直至 capture 结束 | `ScreenshotService.capture()` |
| 连续点击/滑动/输入导致反复 hide/show | `ActionHandler` |
| 后台进程/Service 被杀 | `FloatingWindowService` 生命周期 |
| show 未完成就已进入下一轮 hide | `FloatingWindowService` 异步 `show()` / `ActionHandler.showFloatingWindow()` |

---

## 建议的验证方式

在复现场景下查看日志标签（工程内已有 `Logger`）：

- `ScreenshotService`：`Hiding floating window` / `Restoring floating window` 与时间间隔。
- `FloatingWindow`：`hide()` / `show()` / `Window added` / `Service destroying` / `Failed to add window` 等。

若消失时段始终夹在「截屏 hide」与「finally restore」之间，则优先优化截屏路径或缩短不可见窗口；若出现 `Service destroying` 而无对应恢复，则偏向保活与后台策略。

---

## 可选改进方向（实现时需单独评审）

- 缩短截屏占用主路径时间，或仅在必须「画面无悬浮窗」时 hide。
- 对 `showFloatingWindow()` 与 hide 对称地做「可见性」等待（注意死锁与超时）。
- 评估是否为悬浮窗相关能力使用稳定的前台服务或与现有 `ContinuousListeningService` 等保活策略对齐（以产品与安全合规为前提）。

---

*文档对应代码版本以仓库当前实现为准；涉及路径均为 `app/src/main/java` 下相对包路径。*

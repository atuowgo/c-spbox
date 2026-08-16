# 居家自律 HomeLock · Android 实现文档

> 配套文档：产品设计 `docs/design.md`（v0.2）、交互原型 `docs/prototype.html`
> 平台：Android（本文档范围）；iOS 见 design.md 第 11 节，另行设计
> 目标读者：负责实现的 Android 工程师

本文档把 App 拆成 **15 个可独立验证的步骤**。每一步给出：**做什么 → 实现方式 → 涉及文件路径 → 验证方法**。
建议按顺序实现，每步完成即可自测通过再进入下一步。

---

## 技术栈与基线（先读）

| 项目 | 选型 | 说明 |
|------|------|------|
| 语言 | Kotlin | |
| UI | Jetpack Compose + Material 3 | 深色为主，见 prototype |
| 架构 | 单 module + MVVM + 简单分层（data / domain / feature）| 首版不必多 module |
| DI | Hilt | 可选，推荐；不用则手写单例 |
| 持久化 | Room（会话/事件）+ DataStore（配置/模板）| |
| 后台常驻 | Foreground Service（`specialUse`/`location` 类型）| 核心 |
| 定时/兜底 | AlarmManager（精确闹钟）+ WorkManager（周期兜底）| |
| 到家检测 | Play Services Geofencing + WifiManager + 手动 | 三信号融合 |
| 使用监控 | UsageStatsManager（首选）或 AccessibilityService（可选增强）| |
| 遮罩/告警界面 | `SYSTEM_ALERT_WINDOW` 悬浮窗 + 全屏 Activity（`turnScreenOn`）| |
| 告警 | `Vibrator` + `AudioManager.STREAM_ALARM` + 勿扰放行 | |
| 最低/目标版本 | minSdk 26（Android 8）/ targetSdk 35（Android 15）| targetSdk 34+ 需声明前台服务类型 |
| 构建 | Gradle Kotlin DSL + Version Catalog | |

**包名约定**：`com.homelock`（下文路径以 `app/src/main/java/com/homelock/` 为根，简写为 `…/`）。

**推荐目录结构**（在步骤 0 建立）：

```
app/src/main/
├── AndroidManifest.xml
├── java/com/homelock/
│   ├── HomeLockApp.kt                 // Application，初始化通道/WorkManager
│   ├── MainActivity.kt
│   ├── di/                            // Hilt modules
│   ├── data/
│   │   ├── db/{entity,dao}, AppDatabase.kt
│   │   ├── prefs/SettingsStore.kt     // DataStore
│   │   └── repo/{SessionRepository,ConfigRepository,StatsRepository}.kt
│   ├── domain/
│   │   ├── model/{SessionState,DisciplineConfig,...}.kt
│   │   └── StateMachine.kt
│   ├── detection/
│   │   ├── WifiHomeDetector.kt, GeofenceManager.kt
│   │   ├── GeofenceBroadcastReceiver.kt, WifiStateReceiver.kt
│   │   └── HomeArrivalCoordinator.kt
│   ├── service/{DisciplineService.kt, BootReceiver.kt, WatchdogWorker.kt}
│   ├── monitor/{UsageMonitor.kt, ScreenReceiver.kt, ForegroundAppDetector.kt, EmergencyAllowlist.kt}
│   ├── alarm/{AlarmController.kt, OverlayController.kt, InterceptActivity.kt}
│   ├── notify/Notifications.kt
│   └── ui/{onboarding,home,config,active,settings,theme}/
├── res/  (layout for overlay, drawable, values)
└── ...
app/src/test/        // JVM 单元测试
app/src/androidTest/ // 仪器测试(Espresso/Compose/Room)
```

**总验证策略**：单元测试跑状态机/时间计算；仪器测试跑 Room/Compose；系统行为（服务、权限、告警、检测）用真机 + `adb` 命令验证。文末附**总验收清单**。

---

## 步骤 0 · 项目脚手架与依赖

**做什么**：建立可运行的空壳工程、目录结构、依赖与权限声明。

**实现方式**
1. Android Studio 新建 Empty Compose Activity 工程，包名 `com.homelock`，minSdk 26 / targetSdk 35。
2. `gradle/libs.versions.toml` 加入：`compose-bom`、`material3`、`lifecycle-*`、`room-*`、`datastore-preferences`、`work-runtime`、`play-services-location`、`hilt`。
3. `AndroidManifest.xml` 预声明全部权限（后续步骤逐一使用）：
   ```xml
   <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
   <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
   <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE"/>
   <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
   <uses-permission android:name="android.permission.VIBRATE"/>
   <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
   <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
   <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION"/>
   <uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
   <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
   <uses-permission android:name="android.permission.ACCESS_NOTIFICATION_POLICY"/>
   <uses-permission android:name="android.permission.PACKAGE_USAGE_STATS" tools:ignore="ProtectedPermissions"/>
   <uses-permission android:name="android.permission.READ_PHONE_STATE"/>
   ```
4. 建立上文目录结构（空文件/占位），`HomeLockApp` 注册为 `android:name`。

**涉及路径**：`build.gradle.kts`、`gradle/libs.versions.toml`、`app/src/main/AndroidManifest.xml`、`…/HomeLockApp.kt`、`…/MainActivity.kt`

**验证方法**
- `./gradlew assembleDebug` 构建通过。
- 装到真机能启动，显示空首页。
- `adb shell dumpsys package com.homelock | grep permission` 能看到已声明权限。

---

## 步骤 1 · 数据层（Room + DataStore）

**做什么**：落地 design.md 第 6 节的数据模型。

**实现方式**
1. **DataStore（配置/模板）**：`SettingsStore` 保存 `HomeConfig`（WiFi SSID 列表、围栏经纬度/半径、检测时段）与 `DisciplineTemplate`（延迟、时长、豁免次数、每次时长、isDefault）。
2. **Room（运行时/统计）**：
   - `DisciplineSessionEntity`：`id, state, startAt, prepareEndAt, endAt, exemptionsUsed, currentExemptionEndAt, templateSnapshotJson, createdAt`。
   - `UsageEventEntity`：`id, sessionId, type(枚举), timestamp, durationMs`。
   - `SessionDao` / `EventDao`（含 `observeActiveSession()` 返回 Flow）。
   - `AppDatabase`（version 1）。
3. Repository 封装读写：`SessionRepository`、`ConfigRepository`、`StatsRepository`。
4. 枚举 `SessionState { IDLE, PREPARING, ACTIVE, EXEMPTED, ALARMING, FINISHED }`、`UsageEventType`。

**涉及路径**：`…/data/db/entity/*.kt`、`…/data/db/dao/*.kt`、`…/data/db/AppDatabase.kt`、`…/data/prefs/SettingsStore.kt`、`…/data/repo/*.kt`、`…/domain/model/*.kt`

**验证方法**
- 仪器测试 `androidTest/.../SessionDaoTest.kt`：用 `Room.inMemoryDatabaseBuilder` 插入/查询/更新 session，断言 `observeActiveSession` 发射正确。
- 单元测试：`DisciplineTemplate` 的默认值与序列化（templateSnapshotJson）往返一致。
- `./gradlew connectedDebugAndroidTest` 通过。

---

## 步骤 2 · 权限管理与引导流程

**做什么**：首启引导用户逐项授予敏感权限，并做权限自检。

**实现方式**
1. `PermissionManager`（domain/util）：封装每项权限的"是否已授予 + 跳转授权 Intent"：
   - 通知：`POST_NOTIFICATIONS`（运行时请求，API 33+）。
   - 定位：前台 `ACCESS_FINE_LOCATION` 运行时请求；后台 `ACCESS_BACKGROUND_LOCATION` 单独二次引导（Android 11+ 需跳设置）。
   - 悬浮窗：`Settings.canDrawOverlays()` → `ACTION_MANAGE_OVERLAY_PERMISSION`。
   - 使用情况：`AppOpsManager` 检查 `OPSTR_GET_USAGE_STATS` → `ACTION_USAGE_ACCESS_SETTINGS`。
   - 勿扰访问：`NotificationManager.isNotificationPolicyAccessGranted()` → `ACTION_NOTIFICATION_POLICY_ACCESS_SETTINGS`。
   - 电池优化：`isIgnoringBatteryOptimizations()` → `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`。
2. Compose 引导页 `OnboardingScreen`：分卡片列出各项，实时显示 ✔/去授权，全部就绪才允许进入主流程。
3. 设置页复用同一 `PermissionManager` 做"权限自检"（对应 prototype 设置页）。

**涉及路径**：`…/domain/util/PermissionManager.kt`、`…/ui/onboarding/OnboardingScreen.kt`、`…/ui/onboarding/OnboardingViewModel.kt`

**验证方法**
- 真机全新安装：逐项授权，页面状态实时变绿。
- 手动撤销某权限（系统设置）后回到 App，自检项应变红并提供跳转。
- `adb shell appops get com.homelock GET_USAGE_STATS` 返回 `allow` 验证使用情况权限。
- `adb shell dumpsys deviceidle whitelist | grep homelock` 验证电池白名单。

---

## 步骤 3 · 到家检测（WiFi + 围栏 + 手动）

**做什么**：三信号任一命中"离家→到家"翻转即上报到家。

**实现方式**
1. **WiFi 检测** `WifiHomeDetector`：监听 `CONNECTIVITY_ACTION`/`NetworkCallback`，连上后读当前 SSID（Android 10+ 需定位权限），与 `HomeConfig.homeWifiSsids` 比对。
2. **地理围栏** `GeofenceManager`：用 `GeofencingClient.addGeofences` 注册以家为中心的围栏（`GEOFENCE_TRANSITION_ENTER/EXIT`），触发进 `GeofenceBroadcastReceiver`。
3. **手动** 首页"我到家了"按钮直接上报。
4. **协调与去抖** `HomeArrivalCoordinator`：
   - 维护"当前是否在家"状态（DataStore 持久化），仅在 **false→true 翻转**时判定"到家"。
   - 同一"到家会话"只上报一次；被拒后当天不再重复（可配置）。
   - 支持"仅在检测时段内生效"。
   - 命中后调用步骤 4 的到家提醒。

**涉及路径**：`…/detection/WifiHomeDetector.kt`、`GeofenceManager.kt`、`GeofenceBroadcastReceiver.kt`、`WifiStateReceiver.kt`、`HomeArrivalCoordinator.kt`

**验证方法**
- 单元测试 `HomeArrivalCoordinatorTest`：模拟信号序列（离家→到家→到家），断言只上报一次；跨天重置正确。
- 真机：把家 WiFi 设为当前 SSID，断开重连 → 触发到家（看日志/通知）。
- 围栏：`adb shell` 用模拟位置或 Android Studio 的 Location 模拟进/出围栏，观察 `GeofenceBroadcastReceiver` 日志。
- 手动按钮直接触发提醒。

---

## 步骤 4 · 到家提醒（通知 + 配置入口）

**做什么**：到家后弹高优先级通知，点击进入参数配置；含"忽略"。

**实现方式**
1. `Notifications`：集中创建通道——`arrival`（高优先）、`foreground`（低优先常驻）、`alarm`（可绕过勿扰，`setBypassDnd(true)`）。
2. 到家提醒用 `arrival` 通道，`setFullScreenIntent` 可选，含两个 action：`开启`（→ ConfigActivity/route）、`忽略`（回写"当天已拒"）。
3. 点击"开启"进入步骤 5 配置界面。

**涉及路径**：`…/notify/Notifications.kt`、`…/detection/HomeArrivalCoordinator.kt`（调用）、`…/ui/config/*`

**验证方法**
- 触发步骤 3 的到家 → 收到通知，样式/按钮正确。
- 点"开启"跳配置页；点"忽略"当天不再弹（再次触发到家验证）。
- `adb shell dumpsys notification | grep homelock` 查看通道优先级。

---

## 步骤 5 · 参数配置界面与模板

**做什么**：实现 prototype 的配置页四项步进 + 保存默认模板。

**实现方式**
1. Compose `ConfigScreen` + `ConfigViewModel`：四个步进器（开启延迟/自律时长/豁免次数/每次时长），取值范围与步长见 design.md 5.2。
2. "保存为默认模板"写入 DataStore；下次到家提醒可"一键沿用默认"。
3. 点"开始"：由 ViewModel 创建一条 `DisciplineSession`（state=PREPARING，算出 `prepareEndAt`、`endAt`），交给步骤 6 的服务启动。

**涉及路径**：`…/ui/config/ConfigScreen.kt`、`ConfigViewModel.kt`、`…/data/repo/ConfigRepository.kt`

**验证方法**
- Compose 测试 `ConfigScreenTest`：点 +/− 断言显示值与边界钳制正确。
- 保存模板后重启 App，默认值被记住。
- 点"开始"后 DB 中出现一条 PREPARING 会话，`endAt = now + delay + duration`（单元测试校验时间计算）。

---

## 步骤 6 · 自律状态机 + 前台服务（核心）

**做什么**：把会话状态机跑在常驻前台服务里，驱动准备期→自律期→结束。

**实现方式**
1. **纯函数状态机** `StateMachine`：输入 `(当前状态, 事件, 时间)`，输出 `(新状态, 副作用列表)`。事件包括：`PrepareTimeout / UsageDetected / GraceTimeout / ExemptionStart / ExemptionEnd / EndReached / UserCancel / EmergencyActive`。副作用如 `StartAlarm/StopAlarm/ShowOverlay/ScheduleAt(...)`。**不依赖 Android**，便于单测。
2. **DisciplineService**（`startForeground`，类型 `specialUse`+`location`）：
   - 持有当前 session，订阅监控事件（步骤 7）与定时（AlarmManager），把事件喂给 `StateMachine`，执行副作用。
   - 用 `AlarmManager.setExactAndAllowWhileIdle` 安排 `prepareEndAt`、`endAt`、豁免到期、宽限到期等时间点（Doze 下也能触发）。
   - 状态每次变更持久化到 Room（供崩溃/重启恢复）。
   - 常驻通知显示当前状态与剩余时间。
3. 生命周期：配置页"开始" → `ContextCompat.startForegroundService`；结束/取消 → `stopSelf`。

**涉及路径**：`…/domain/StateMachine.kt`、`…/service/DisciplineService.kt`、`…/data/repo/SessionRepository.kt`

**验证方法**
- 单元测试 `StateMachineTest`（重点）：覆盖 design.md 第 4 节所有转移，尤其
  `ACTIVE --UsageDetected--> (Overlay+Grace)`、`--GraceTimeout--> ALARMING`、
  `ALARMING --StopUsing--> ACTIVE`、`* --EndReached--> FINISHED`、豁免进出、紧急放行。
- 真机：把延迟/时长设很短（如 1 分钟）跑完整周期，观察常驻通知文案随状态变化。
- `adb shell dumpsys activity services com.homelock`：确认前台服务存活且类型正确。
- `adb shell dumpsys alarm | grep homelock`：确认关键时间点已排程。

---

## 步骤 7 · 使用监控 + 紧急放行（整机口径）

**做什么**：自律期内判断"是否在使用手机"，命中且非紧急即上报 `UsageDetected`。

**实现方式**（对应 Q2：整机，但放行电话等紧急）
1. **屏幕/解锁** `ScreenReceiver`：监听 `SCREEN_ON`、`USER_PRESENT`、`SCREEN_OFF`（`SCREEN_OFF`→上报"停止使用"）。
2. **前台应用** `ForegroundAppDetector`：优先用 `UsageStatsManager.queryEvents` 取最近 `MOVE_TO_FOREGROUND` 的包名（轮询 1–2s，仅在自律期开启）；如启用了无障碍则用 `AccessibilityService` 的窗口变化事件实时判断（更省电、更实时，作为可选增强）。
3. **紧急放行** `EmergencyAllowlist`（Q5）：满足任一即视为"不算违规"：
   - 通话状态：`TelephonyManager`/`TelephonyCallback` 处于 `RINGING`/`OFFHOOK`；
   - 前台包属于放行类别（电话/紧急、闹钟、导航——按包名/类别白名单，可在设置页维护）；
   - 系统正在响铃闹钟（`AudioManager` 或已知闹钟包）。
4. `UsageMonitor` 汇总：解锁且屏幕亮且非紧急 → 向服务发 `UsageDetected`；否则发"停止使用"。

**涉及路径**：`…/monitor/UsageMonitor.kt`、`ScreenReceiver.kt`、`ForegroundAppDetector.kt`、`EmergencyAllowlist.kt`、（可选）`…/monitor/HomeLockAccessibilityService.kt` + `res/xml/accessibility_config.xml`

**验证方法**
- 单元测试 `EmergencyAllowlistTest`：给定通话状态/前台包，断言放行判定正确。
- 真机（自律期内）：点亮解锁 → 触发劝返（步骤 8）；息屏 → 解除。
- 拨打/接听电话时使用手机 → **不**触发告警（放行验证）。
- 打开地图导航 → 不触发；打开普通 App → 触发。
- `adb shell dumpsys usagestats`/日志确认前台包识别正确。

---

## 步骤 8 · 全屏劝返遮罩（30 秒宽限）

**做什么**：`UsageDetected` 后显示劝返界面并倒计时 30 秒（Q3）。

**实现方式**
1. `OverlayController`：用 `WindowManager` 加 `TYPE_APPLICATION_OVERLAY` 悬浮窗（需悬浮窗权限），或用全屏 `InterceptActivity`（`setShowWhenLocked/turnScreenOn`，可在锁屏上弹）。首版推荐**悬浮窗遮罩**，遮住当前界面但保留系统关键交互。
2. 遮罩内容：文案"自律模式进行中" + 30 秒环形倒计时 + 两个按钮：`放下手机(锁屏)`、`使用豁免`；对应 prototype 的 intercept 界面。
3. 计时由服务用 AlarmManager 排 `GraceTimeout`；30 秒内若"停止使用"或"启用豁免"→ 撤下遮罩、取消排程；否则 → 服务收到 `GraceTimeout` → 进入 ALARMING（步骤 9）。
4. "放下手机"可调用锁屏（需 Device Admin 才能强制锁屏；首版可仅收起遮罩并提示，或引导息屏）。

**涉及路径**：`…/alarm/OverlayController.kt`、`…/alarm/InterceptActivity.kt`、`res/layout/overlay_intercept.xml`（若用 View）或 Compose overlay

**验证方法**
- 真机（自律期）：解锁使用 → 立即出现遮罩与 30s 倒计时。
- 30s 内点"放下手机" → 遮罩消失、无告警。
- 30s 内不操作 → 自动进入告警（步骤 9）。
- 锁屏场景下遮罩/Activity 能正确显示（若走 InterceptActivity）。

---

## 步骤 9 · 告警：震动 + 嗡鸣（穿透静音）

**做什么**：ALARMING 状态持续震动+嗡鸣，停止条件满足即止（design.md 5.5）。

**实现方式**
1. `AlarmController`：
   - **震动**：`Vibrator`/`VibratorManager` 播放循环 `VibrationEffect.createWaveform(pattern, repeat=0)`。
   - **声音**：`MediaPlayer` 或 `Ringtone`，`AudioAttributes.USAGE_ALARM` + `STREAM_ALARM`，`isLooping=true`；配合勿扰访问确保静音/勿扰下也能响。
   - 音量/强度可配；设"单次最长告警时长"上限（如 5 分钟）后自动降级，防意外扰民。
2. 告警界面复用遮罩（红色态）+ `alarm` 通道全屏通知（`fullScreenIntent`，锁屏也能弹）。
3. **停止条件**：`SCREEN_OFF`/停止使用、启用豁免、到达结束时间、命中紧急放行 → 立刻 `stopAlarm()`。
4. 紧急场景（来电）优先级最高：来电时强制静音告警并放行。

**涉及路径**：`…/alarm/AlarmController.kt`、`…/notify/Notifications.kt`（alarm 通道）、`…/service/DisciplineService.kt`（副作用调用）

**验证方法**
- 真机：不理会劝返 30s → 开始持续震动+响铃。
- 手机切静音/开勿扰 → 告警仍能响（勿扰放行验证）。
- 息屏或点"使用豁免" → 告警立即停止。
- 来电进来 → 告警自动停止并放行。
- 计时到"单次最长" → 自动降级停止（把上限调短便于验证）。

---

## 步骤 10 · 豁免机制

**做什么**：应急临时放行，消耗额度，到期自动恢复（Q4：可提前结束、每周期清零）。

**实现方式**
1. 遮罩/告警/自律页的"使用豁免"→ 服务收 `ExemptionStart`：`exemptionsUsed++`，`currentExemptionEndAt = now + 每次时长`，状态转 EXEMPTED，`stopAlarm()`，撤遮罩，排程 `ExemptionEnd`。
2. 额度用尽（`exemptionsUsed >= 次数`）时按钮置灰。
3. 到期前 30s 轻震预告；到期 `ExemptionEnd` → 回 ACTIVE，若仍在使用则重新走监控→劝返。
4. "提前结束豁免"：立即触发 `ExemptionEnd`（省下剩余时间，但已消耗的次数不退——文案说明）。

**涉及路径**：`…/service/DisciplineService.kt`、`…/domain/StateMachine.kt`、`…/ui/active/ActiveScreen.kt`

**验证方法**
- 单元测试：`ExemptionStart` 使 `exemptionsUsed` 递增并设正确到期时间；用尽后再次 `ExemptionStart` 被拒。
- 真机：告警中点豁免 → 进入豁免、告警停、倒计时开始；到期自动回自律。
- 提前结束 → 立即回自律；额度用尽后豁免按钮不可点。
- 每周期清零：完成一次会话后新会话额度恢复满。

---

## 步骤 11 · 结束与统计

**做什么**：会话结束落库并展示本次回顾（prototype 结束页 + 首页今日统计）。

**实现方式**
1. `EndReached`/`UserCancel` → state=FINISHED，写入结束事件，服务 `stopSelf`。
2. `StatsRepository` 聚合：专注时长、违规次数、告警次数、豁免使用、净放下时长；供首页与结束页读取（Flow）。
3. Compose `FinishedScreen` + 首页 `HomeScreen` 今日统计卡。

**涉及路径**：`…/data/repo/StatsRepository.kt`、`…/ui/active/FinishedScreen.kt`、`…/ui/home/HomeScreen.kt`

**验证方法**
- 仪器测试：插入若干 `UsageEvent`，断言聚合数值正确。
- 真机跑完一次短会话 → 结束页与首页统计数字与实际一致。

---

## 步骤 12 · 保活、重启恢复与厂商适配

**做什么**：进程被杀/重启后恢复未结束会话，降低被杀概率。

**实现方式**
1. `BootReceiver`（`RECEIVE_BOOT_COMPLETED`）：开机后查 Room 是否有未结束会话（`endAt > now`），有则重新 `startForegroundService` 并按持久化时间重排 AlarmManager。
2. `WatchdogWorker`（WorkManager 周期任务，15 分钟）：兜底检查——应在运行却没运行则拉起服务；已过 `endAt` 的僵尸会话置 FINISHED。
3. 服务 `onStartCommand` 返回 `START_STICKY`；崩溃恢复时从 Room 重建状态。
4. 厂商适配：引导小米/华为/OV 等加"自启动/后台白名单/省电白名单"，设置页提供跳转与说明（作为已知风险项，见 design.md 8.2）。

**涉及路径**：`…/service/BootReceiver.kt`、`…/service/WatchdogWorker.kt`、`…/HomeLockApp.kt`（注册 Work）、`…/ui/settings/*`

**验证方法**
- 真机：会话进行中"强行停止"App → WatchdogWorker/黏性重启后服务恢复（`dumpsys activity services` 复查）。
- `adb shell am broadcast -a android.intent.action.BOOT_COMPLETED -p com.homelock`（或真机重启）→ 未结束会话恢复、通知重现、`dumpsys alarm` 有重排。
- `adb shell cmd jobscheduler run -f com.homelock <jobId>` 手动触发 WorkManager 兜底逻辑。

---

## 步骤 13 · 设置页

**做什么**：实现 prototype 设置页全部分区。

**实现方式**
- Compose `SettingsScreen`：到家检测（WiFi 选择、围栏地图选点/半径、手动兜底开关）、默认模板编辑、告警强度（震动/音量/宽限/单次上限）、放行清单维护（增删放行 App/类别）、权限自检（复用步骤 2）。各项读写 DataStore。

**涉及路径**：`…/ui/settings/SettingsScreen.kt`、`SettingsViewModel.kt`、`…/data/prefs/SettingsStore.kt`

**验证方法**
- 改任一设置 → 重启 App 后仍生效（DataStore 持久化）。
- 修改围栏/WiFi 后，步骤 3 的检测按新配置生效。
- 放行清单增删后，步骤 7 放行判定随之变化。

---

## 步骤 14 · 联调、测试与验收

**做什么**：端到端跑通 + 自动化测试 + 手动验收矩阵。

**实现方式**
1. **单元测试**（`app/src/test`）：`StateMachineTest`、`EmergencyAllowlistTest`、`HomeArrivalCoordinatorTest`、时间计算工具测试。
2. **仪器测试**（`app/src/androidTest`）：Room DAO、关键 Compose 页面（Config/Home）。
3. **手动 QA 矩阵**（真机）见下。

**手动验收矩阵**

| 场景 | 期望 |
|------|------|
| 到家（WiFi/围栏/手动）| 各自都能触发一次提醒，不重复 |
| 拒绝提醒 | 当天不再弹 |
| 配置→准备期→自律 | 状态与倒计时正确，常驻通知更新 |
| 自律期解锁使用 | 出现劝返 + 30s 倒计时 |
| 30s 内放下 / 忽略 | 放下不告警；忽略则告警 |
| 告警 | 持续震动+嗡鸣，静音/勿扰下仍响 |
| 来电/闹钟/导航 | 放行，不告警 |
| 使用豁免 / 提前结束 / 用尽 | 行为符合步骤 10 |
| 到结束时间 | 进入结束页，统计正确 |
| 杀进程 / 重启手机 | 会话恢复 |
| 厂商省电限制 | 引导白名单后保活正常 |

**验证方法**
- `./gradlew testDebugUnitTest connectedDebugAndroidTest` 全绿。
- 按矩阵逐项在真机通过（建议至少覆盖一台原生 Android + 一台国产 ROM）。
- 出包：`./gradlew assembleRelease`，安装冒烟测试。

---

## 附录 A · 常用调试命令速查

```bash
# 前台服务状态
adb shell dumpsys activity services com.homelock
# 已排程闹钟
adb shell dumpsys alarm | grep homelock
# 通知与通道
adb shell dumpsys notification | grep -A3 homelock
# 使用情况权限
adb shell appops get com.homelock GET_USAGE_STATS
# 电池白名单
adb shell dumpsys deviceidle whitelist | grep homelock
# 模拟开机恢复
adb shell am broadcast -a android.intent.action.BOOT_COMPLETED -p com.homelock
# 强杀后观察黏性重启
adb shell am force-stop com.homelock
```

## 附录 B · 关键风险与降级

| 风险 | 应对 |
|------|------|
| 国产 ROM 后台杀进程 | 前台服务 + 白名单引导 + WatchdogWorker 兜底；文档明示局限 |
| Android 无法真正锁死手机 | 定位为"强打扰劝返"，非硬封锁（design.md 8.3）|
| 用户卸载逃避 | 消费级 App 无法防止，列为已知局限 |
| 悬浮窗/勿扰权限被拒 | 降级为高优先通知 + 系统音量提醒，并强提示去授权 |
| 前台包识别延迟（轮询）| 可选启用无障碍服务提升实时性 |

---

**实现顺序建议**：0→1→2 打基础；3→4→5 打通"到家到配置"；6→7→8→9 打通"监控到告警"核心闭环；10→11 完善出口与统计；12→13→14 稳定化与验收。每完成一步用本文对应"验证方法"自测通过再继续。

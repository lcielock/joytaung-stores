# 表情系统说明

## 优先级等级总览

| 等级 | 名称 | 优先级数值 | 锁定时间 | 说明 |
|------|------|-----------|----------|------|
| P0 | 打断优先级 (Interrupt) | 3 (最高) | 3秒 | 摇晃/触摸触发，AI会话中被P0打断 |
| P1 | AI语音优先级 (AiVoice) | 2 | 24小时 | AI对话期间的表情，loop=true |
| P2 | 发呆优先级 (Idle) | 1 | 5秒 | P3无活动30秒后进入发呆状态 |
| P3 | 默认优先级 (Default) | 0 (最低) | 31秒 | 待机默认表情，loop=true |

**优先级规则**：数值越高越优先。高优先级可以打断低优先级，低优先级无法替换高优先级。

---

## 各等级详细说明

### P0 - 打断优先级

**触发条件**：
- 摇晃设备（六轴传感器 QMI8658 检测到加速度变化）
- 触摸屏幕

**SD卡目录**：`/sdcard/assets/emotions/p0`

**内置表情**：`shaking`, `ss`, `pain`, `lol`

**时间参数**：
- `QMI8658_SHAKE_ACCEL_DELTA_THRESHOLD = 0.6f`（摇晃灵敏度阈值，越小越灵敏）
- `QMI8658_SHAKE_COOLDOWN_MS = 1500`（摇晃触发冷却时间）
- `kP0DisplayHoldUs = 3秒`（P0锁定时间，3秒内不能再次触发P0）

**行为**：
- 播放非循环GIF（`loop=false`），播放完后自动恢复之前的AI表情或进入P3
- 如果AI会话正在进行，打断P1队列，清空排队中的表情
- 锁定期间新的P0触发会被拒绝

**触发来源代码位置**：
- 摇晃：`TriggerPhysical()` — 六轴传感器中断
- 触摸：`TriggerTouch()` — 触摸屏点击

---

### P1 - AI语音优先级

**触发条件**：
- AI会话期间，服务器下发表情指令
- 表情持续循环播放直到AI会话结束

**SD卡目录**：`/sdcard/assets/emotions/p1`

**内置表情**：`neutral`, `happy`, `laughing`, `funny`, `sad`, `angry`, `crying`, `loving`, `embarrassed`, `surprised`, `shocked`, `thinking`, `winking`, `cool`, `relaxed`, `delicious`, `kissy`, `confident`, `sleepy`, `silly`, `confused`

**时间参数**：
- `kP1DisplayHoldUs = 24小时`（实际表示直到AI会话结束）

**行为**：
- 表情 `loop=true`，无限循环播放
- 多个P1表情会排队播放：当前表情播放完毕后，队列中的下一个表情自动出队播放
- AI会话结束时，所有P1表情清空，自动进入P3

**队列机制**：
- `HandleAiEmotion()` 检查当前是否有表情在播放
- 如果有，新表情入队；没有则立即播放
- `OnEmotionFinished()` 负责出队下一个表情

---

### P2 - 发呆优先级

**触发条件**：
- P3（默认状态）无活动达到30秒后自动进入

**SD卡目录**：`/sdcard/assets/emotions/p2`

**内置表情**：`looking_around`, `ok`

**时间参数**：
- `kP2DisplayHoldUs = 5秒`（发呆状态持续5秒）
- `kP3ToP2IntervalUs = 15秒`（P3无活动15秒后进入P2）

**行为**：
- 播放非循环GIF，5秒后自动回到P3

---

### P3 - 默认优先级

**触发条件**：
- 系统启动后默认进入
- P0/P2/P1结束后自动恢复
- AI会话结束后自动恢复

**SD卡目录**：`/sdcard/assets/emotions/p3`

**内置表情**：`moren`

**时间参数**：
- `kP3DisplayHoldUs = 31秒`（锁定时间，防止同优先级重复替换）
- `kP3ToP2IntervalUs = 15秒`（无活动15秒后进入P2）

**行为**：
- 表情 `loop=true`，无限循环播放
- 30秒无活动后自动进入P2发呆状态

---

## 表情文件要求

### 文件格式
- **GIF**：动态表情，支持循环/非循环
- **PNG**：静态表情（内置emoji使用）

### 文件命名
- 文件名（不含扩展名）即为表情名称
- 例如：`happy.gif`、`laughing.gif`、`sad.png`

### 存储位置优先级
1. SD卡：`/sdcard/assets/emotions/p{0-3}/`
2. 内置资源：代码中硬编码的表情列表

**注意**：SD卡表情和内置表情名称相同时，SD卡版本优先。

---

## 表情显示流程

### 静态图片（非GIF）的完成触发
内置emoji（如 `sad`、`neutral` 等PNG图片）本身不会产生"播放完成"事件。为了支持P1队列机制，系统使用 `esp_timer` 为非循环的静态图片启动一个1.5秒的定时器，定时器到期后触发 `OnEmotionFinished()`，使队列中的下一个表情得以出队播放。

### P0打断P1的完整流程
```
1. AI正在播放P1表情（loop=true）
2. 用户摇晃/触摸 → TriggerPhysical/TriggerTouch
3. P0打断P1：清空P1队列，设置 p1_emotion_playing_=false
4. 播放P0表情（loop=false）
5. P0播放完毕 → OnEmotionFinished → RecoverAfterP0Locked
6. RecoverAfterP0Locked：
   - 如果AI还在 → 恢复最后一个AI表情（loop=true）
   - 如果AI已结束 → 进入P3
```

---

## 时间参数汇总

| 参数名 | 当前值 | 说明 |
|--------|--------|------|
| `kP0DisplayHoldUs` | 3秒 | P0锁定时间，防止频繁触发 |
| `kP1DisplayHoldUs` | 24小时 | P1锁定时间（实际表示AI会话时长） |
| `kP2DisplayHoldUs` | 5秒 | P2发呆状态持续时间 |
| `kP3DisplayHoldUs` | 31秒 | P3锁定时间 |
| `kP3ToP2IntervalUs` | 15秒 | P3无活动多久后进入P2 |
| `QMI8658_SHAKE_ACCEL_DELTA_THRESHOLD` | 0.6f | 摇晃灵敏度（越小越灵敏） |
| `QMI8658_SHAKE_COOLDOWN_MS` | 1500ms | 摇晃触发冷却时间 |

---

## 相关代码文件

- `main/boards/waveshare/esp32-s3-touch-lcd-1.85-qmi/esp32-s3-touch-lcd-1.85-qmi.cc` — 表情状态机 `PriorityEmotionArbiter` 类
- `main/boards/waveshare/esp32-s3-touch-lcd-1.85-qmi/config.h` — 配置参数定义
- `main/display/lcd_display.cc` — 表情显示逻辑 `LcdDisplay::SetEmotion()`
- `main/display/lvgl_display/gif/lvgl_gif.cc` — GIF动画控制器

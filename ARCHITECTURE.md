# HALOVIA 制谱器 (Chart Editor) v5 — 架构文档

> **版本**: v5 (`.hvf` format v2.0)  
> **技术栈**: 纯前端原生 JavaScript (ES5, `"use strict"`), HTML5 Canvas 2D, Web Audio API, 纯 CSS 暗色主题  
> **零依赖**: 无框架, 无构建工具, 无 npm, 无服务端. 直接打开 `index.html` 即可运行.  
> **目的**: 为 HALOVIA 3D 无轨道下落式音游制作谱面.

---

## 目录

1. [项目概览](#1-项目概览)
2. [文件结构与加载顺序](#2-文件结构与加载顺序)
3. [全局数据模型](#3-全局数据模型)
4. [表达式系统](#4-表达式系统)
5. [轨道与路径引擎](#5-轨道与路径引擎)
6. [轨道面板 (Track Panels)](#6-轨道面板-track-panels)
7. [时间轴 (Timeline)](#7-时间轴-timeline)
8. [3D 预览渲染](#8-3d-预览渲染)
9. [音频系统](#9-音频系统)
10. [游戏循环与判定](#10-游戏循环与判定)
11. [键位映射与游玩模式](#11-键位映射与游玩模式)
12. [导入导出 (I/O)](#12-导入导出-io)
13. [撤销/重做](#13-撤销重做)
14. [UI 组件与弹窗](#14-ui-组件与弹窗)
15. [样式系统](#15-样式系统)
16. [常见修改指南](#16-常见修改指南)

---

## 1. 项目概览

### 1.1 核心功能

| 功能 | 描述 |
|------|------|
| 路径编辑 | 关键帧 + 运动预设插值, 10 种预设 (直线/贝塞尔/椭圆/Lissajous/自定义表达式等) |
| 3D 实时预览 | Canvas 2D 透视投影渲染音符下落、判定环运动、Hold 轨迹 |
| 时间轴编辑 | 节拍线、波形、音符 (Tap/Keep/Hold)、关键帧、A/B 循环、吸附 |
| 游玩模式 | 完整的判定系统 (P/G/B/M)、Combo、计分 (满分 1,000,000) |
| 双格式 I/O | `.hvf` (JSON 保留表达式) 用于编辑, `.hvp` (紧凑十进制) 用于游戏运行时 |
| 数学表达式 | 所有数值输入支持 `sin/cos/pi/^` 等, 以字符串形式保存 |

### 1.2 数据流

```
用户输入 → state 对象 → applyStateToUI()
                           ├── renderTrackDirectory()  (轨道目录)
                           ├── renderTrackPanels()     (轨道面板, 重建 DOM)
                           └── renderTimeline()        (时间轴 Canvas)

游戏循环:
  requestAnimationFrame(renderLoop)
    ├── 更新 game.gameTime
    ├── processAutoJudgments() / processHoldContinuous()
    ├── renderPreviewFrame(gameTime)  (3D 预览 Canvas)
    └── renderTimeline()               (时间轴 Canvas)
```

---

## 2. 文件结构与加载顺序

`index.html` 中脚本加载顺序严格, 后面的文件可访问前面的全局变量:

```
index.html
  ├── css/style.css         (全局样式)
  ├── js/state.js           (核心数据模型, 表达式系统, 工具函数)
  ├── js/presets.js         (运动预设, 路径插值引擎)
  ├── js/validation.js      (导入数据校验)
  ├── js/ui.js              (UI 工具: Toast, 弹窗, AB 循环, 全局 Esc)
  ├── js/audio.js           (音频加载/播放/频谱分析)
  ├── js/sfx.js             (音效管理)
  ├── js/io.js              (.hvp/.hvf 导入导出)
  ├── js/notes.js           (音符管理, 编辑弹窗)
  ├── js/tracks.js          (轨道面板渲染, 目录, 关键帧/分段操作)
  ├── js/timeline.js        (时间轴 Canvas 渲染与事件)
  ├── js/preview.js         (3D 预览 Canvas, 游戏循环, 预览区拖拽)
  ├── js/gameplay.js        (游玩模式, 判定系统, 键位映射)
  ├── js/undo.js            (撤销/重做栈)
  └── js/init.js            (初始化入口)
```

### 2.1 各文件职责

| 文件 | 行数 | 关键导出 | 依赖 |
|------|------|----------|------|
| `js/state.js` | ~218 | `state` 对象, `evalNumericInput`, `bindExprInput`, 时间/坐标工具 | 无 |
| `js/presets.js` | ~186 | `PRESETS`, `evalSegmentAt`, `evalTrackAt`, `createDefaultTrack` | `state.js` |
| `js/validation.js` | ~60 | `validateImportedData()` | `state.js` |
| `js/ui.js` | ~210 | `showToast`, `syncGlobals`, AB 循环函数, `openImportModal`, `close*`, Esc 处理 | `state.js` |
| `js/audio.js` | ~340 | `loadAudio`, `startAudioAt`, `stopAudio`, 频谱能量 | `state.js` |
| `js/sfx.js` | ~85 | `playSfx`, `loadSfx` | 无 |
| `js/io.js` | ~320 | `encodeHVP`, `encodeHVF`, `decodeHVP`, `decodeHVF`, `doImport` | `state.js`, `presets.js`, `validation.js` |
| `js/notes.js` | ~140 | `addNote`, `removeNoteByIndex`, `openNoteEditModal` | `state.js`, `ui.js` |
| `js/tracks.js` | ~410 | `renderTrackPanels`, `renderTrackDirectory`, `addKeyframe`, `changeSegPreset`, `scrollToTrack` | `state.js`, `presets.js`, `validation.js`, `ui.js` |
| `js/timeline.js` | ~770 | `renderTimeline`, 鼠标/键盘/滚轮事件 | `state.js`, `presets.js`, `notes.js` |
| `js/preview.js` | ~450 | `renderPreviewFrame`, `renderLoop`, 预览区拖拽, `_scrollToKeyframe` | 全部前面文件 |
| `js/gameplay.js` | ~320 | `processAutoJudgments`, `processHoldContinuous`, `showPlayModal`, 键位映射 | `state.js`, `presets.js`, `ui.js`, `sfx.js`, `audio.js`, `preview.js` |
| `js/undo.js` | ~108 | `pushUndo`, `undo`, `redo` (Ctrl+Z/Y) | `state.js` |
| `js/init.js` | ~50 | `init()` 入口, `DOMContentLoaded` 启动 | 全部 |

---

## 3. 全局数据模型

### 3.1 `state` 对象 (`state.js`)

```javascript
const state = {
  // ---- 时间/播放 ----
  bpm: 120,                   // 每分钟节拍数
  sampleRateReal: 10.0,       // 采样频率 (Hz)
  offsetSec: 0,               // 时间偏移 (秒)
  duration: 30,               // 谱面总时长 (秒)
  fov: 55,                    // 3D 视野角度
  noteScale: 1.0,             // 音符大小比例 (0.2–2.0)
  speed: 1.0,                 // 播放速度 (0.05–5.0)
  loopA: null,                // 循环起点 (秒), null=未设置
  loopB: null,                // 循环终点 (秒)
  loopEnabled: false,         // 是否启用循环

  // ---- 轨道/音符数据 ----
  tracks: [],                 // Array<Track>
  openedTracks: [],           // 当前打开的面板 ID 数组
  notes: [],                  // Array<Note>

  // ---- 音频 ----
  audioPath: "",
  audioBuffer: null,          // decoded AudioBuffer
  audioCtx: null,             // AudioContext
  audioSource: null,          // 当前播放节点
  audioStartedAt: 0,          // performance.now() 开始播放时
  audioStartOffset: 0,        // 音频内的播放偏移
  _musicGain: null,           // 音量 GainNode

  // ---- 键位映射 ----
  keyMap: {},                 // { trackId: [keyString, ...] }
};
```

### 3.2 轨道模型 (`presets.js`)

```javascript
// 轨道对象
{
  id: 0,
  keyframes: [
    { time: 0, x: 0, y: 0, hidden: false },
    { time: 30, x: 0, y: 0, hidden: false }
  ],
  segmentPresets: [
    { preset: "line", params: {} }
  ],
  _sampledPath: null  // .hvp 导入采样数据时填充
}

// 音符
{ track: 0, type: 1, time: 1.5, endTime: null }
// type: 1=Tap, 2=Keep, 3=Hold

// 运动预设参数 (每个预设不同, 详见 presets.js)
PRESETS = {
  hidden:      { name: "不显示", params: [] },
  static:      { name: "静止",   params: [], fn: ... },
  line:        { name: "直线",   params: [], fn: ... },
  "line-easein":  { name: "直线-缓入",   params: ["k"], fn: ... },
  "line-easeout": { name: "直线-缓出",   params: ["k"], fn: ... },
  "line-easeinout": { name: "直线-缓入出", params: ["k"], fn: ... },
  bezier:      { name: "贝塞尔曲线", params: ["bzX1","bzY1","bzX2","bzY2"], fn: ... },
  ellipse:     { name: "椭圆/圆", params: ["cx","cy","ax","ay","omega","phiX","phiY"], fn: ... },
  lissajous:   { name: "Lissajous", params: ["cx","cy","ax","ay","omegaX","omegaY","phiX","phiY","funcX","funcY"], fn: ... },
  custom:      { name: "自定义",   params: ["xExpr","yExpr"] }
}
```

### 3.3 游戏运行时状态 (`gameplay.js`)

```javascript
var game = {
  mode: "preview",           // "preview" | "play"
  playing: false,
  paused: false,
  gameTime: 0,               // 当前时间 (秒)
  startRealTime: 0,          // performance.now() 开始本次播放时
  startGameTime: 0,          // 播放开始时的 gameTime
  fullscreen: false,
  judgments: {},             // { noteIdx: { judged, result, judgeTime, holdState } }
  combo: 0, score: 0,
  effects: [],               // 判定特效 (绘制用)
  holdRingShrink: {},        // Hold 判定环缩小动画
  autoHits: new Set(),       // 自动击打集合
  keysDown: {},              // 当前按下的键 { keyStr: true/false }
  simulHitKeys: new Set(),
  playFromAFlag: false
};
```

### 3.4 时间轴状态 (`timeline.js`)

```javascript
var timeline = {
  canvas: null, ctx: null,
  W: 0, H: 0,                       // Canvas 尺寸
  viewStart: 0, viewEnd: 30,        // 可视范围 (秒)
  rulerH: 24,                        // 标尺高度
  trackH: 0,                         // 轨道高度 (动态计算)
  mouseTime: 0,
  hoveredTrack: -1,
  playheadDrag: false,
  panning: false, panStartX: 0, panStartView: 0,
  draggingKF: null,
  draggingNote: null,
  rightDragging: null
};
```

---

## 4. 表达式系统 (`state.js`)

### 4.1 `evalNumericInput(str)`

核心解析器:
1. 空值返回 `NaN`
2. 纯数字直接返回
3. 字符白名单: `[0-9a-zA-Z_.+\-*/^() \t,]`
4. 将 `^` 替换为 `**`, 注入 `Math.PI` / `Math.E` 别名, 暴露 22 个 `Math.*` 函数
5. `eval()` 执行, 返回结果或 `NaN`

### 4.2 表达式对象

```javascript
{ raw: "pi/4", value: 0.7854 }
```

所有数值输入统一用此格式, 在编辑/导出周期中完整保留 `raw` 字符串.

### 4.3 `bindExprInput(el, onCommit)`

绑定到 DOM `<input>`:
- `input` 事件: 实时求值, 有效时调用 `onCommit(expr)`, 无效时加 `expr-invalid` class
- `blur`: 同 input 逻辑
- `Enter`: blur
- `dblclick`: 打开大编辑弹窗 (`openExprZoom`)

### 4.4 工具函数

| 函数 | 说明 |
|------|------|
| `toExpr(v)` | 统一转为 `{raw, value}` 格式 |
| `exprVal(v)` | 安全取值 |
| `exprRaw(v)` | 安全取原始字符串 |
| `secToTimestamp(s)` / `timestampToSec(ts)` | 秒 ↔ 采样帧 |
| `beatToSec(b)` / `secToBeat(s)` | 拍 ↔ 秒 |
| `coordToStored(v)` / `storedToCoord(s)` | 坐标 ↔ .hvp 存储格式 |
| `hueForTrack(i, total)` | 轨道颜色 (色相) |

---

## 5. 轨道与路径引擎 (`presets.js`)

### 5.1 `evalTrackAt(track, globalTime)`

评估轨道在指定时间的路径位置:

1. 有 `_sampledPath` (来自 .hvp 导入) → 直接读取缓存采样
2. 否则: 找到 `globalTime` 所在的分段
   - 计算分段内进度: `localP = (time - fromKF.time) / (toKF.time - fromKF.time)`
   - 调用 `evalSegmentAt(seg, localP, fromKF, toKF)`
   - 考虑 `hidden` 状态

### 5.2 `evalSegmentAt(seg, localP, fromKF, toKF)`

执行单个运动预设的路径计算:

| 预设 | 计算方式 |
|------|----------|
| `hidden` | 返回 `{hidden: true}` |
| `static` | 返回 `fromKF` 坐标不变 |
| `line` | 线性插值 `fromKF → toKF` |
| `line-easein` | `ep = p^k` 后线性插值 |
| `line-easeout` | `ep = 1-(1-p)^k` |
| `line-easeinout` | 分段三次幂 |
| `bezier` | 牛顿迭代解贝塞尔曲线 X→时间, 再用 Y 插值 |
| `ellipse` | 参数方程: `cx + ax*cos(ωp+φ)`, `cy + ay*sin(ωp+φ)` |
| `lissajous` | 双参数方程: 独立 ωx/ωy 和 sin/cos 选择 |
| `custom` | `eval(expr)` 用户自定义, 暴露 `p,P,t,T` |

---

## 6. 轨道面板 (Track Panels) (`tracks.js`)

### 6.1 布局

```
#leftPanel
  ├── #trackDirectory (轨道目录, 标签页)
  │     ├── .dir-header ("轨道目录" + 添加按钮)
  │     └── #trackDirList (.dir-item 列表)
  └── #trackPanelsContainer (overflow-x: auto)
        └── #trackPanels (display: flex; flex-direction: row)
              ├── .track-panel (flex: 0 0 370px)   ← 每个打开的面板
              └── .track-panel ...
```

### 6.2 轨道目录标签页

- 每个轨道一个 `.dir-item`, 点击 `toggleTrackOpen(id)`
- **三种行为**:
  - 未打开 → 打开 + 切换到该面板
  - 已打开且是当前活动 → 关闭面板
  - 已打开但不是活动 → 切换到该面板
- 右击删除轨道 (含确认弹窗)
- 样式: `.dir-item.active` 高亮当前活动轨道

### 6.3 面板内容

每个 `.track-panel` 包含:

```
.track-panel-header (flex, justify-content: space-between)
  ├── .track-panel-title (色点 + "轨道 N")
  └── button.small "+ 关键帧" (右对齐)

.segment-card (关键帧 #N)         ← 每个关键帧一个卡片
  ├── .segment-row
  │     ├── 色点 + #N (strong)
  │     ├── .field.float-label (时间输入)
  │     ├── .field.float-label (X输入)
  │     ├── .field.float-label (Y输入)
  │     ├── .field (隐藏复选框)
  │     └── .field.delete (✕ 按钮, 首尾关键帧无)
  └── (分段预设卡片)

.segment-card (分段 #N~#N+1)      ← 每段一个卡片
  ├── .segment-row
  │     ├── 标签 "#N~#N+1"
  │     ├── .field.float-label (预设下拉)
  │     └── .field (预设参数)...
```

### 6.4 浮动标签

所有输入框使用 `.float-label` 样式: 标签骑跨在输入框上边框, `translateY(-50%)` 居中.

### 6.5 面板间同步滚动

`panel.onscroll`: 按百分比同步所有打开面板的垂直滚动位置. `_syncingScroll` 防止循环触发.

### 6.6 滚动恢复

`renderTrackPanels()` 重建 DOM 前:
1. 按轨道 ID 保存每个面板的 `scrollTop` (存于 `scrollTopMap`)
2. 重建前 `blur()` 聚焦元素
3. 用 `DocumentFragment` + `replaceChildren(frag)` 原子替换
4. `_syncingScroll = true` 恢复 `scrollTop`, 防止 onscroll 互相覆盖

### 6.7 `scrollToTrack(tid)`

根据面板索引计算 `scrollTo({ left: pi * 370 })` 水平滚动到轨道面板位置.

### 6.8 `_scrollToKeyframe(trackId, insIdx)` (preview.js)

在预览区拖拽/创建关键帧后调用:
1. `scrollToTrack(trackId)` 水平定位面板
2. `requestAnimationFrame` → `scrollIntoView({ behavior: "smooth", block: "center" })` 垂直居中关键帧 `<strong>#N</strong>` 标签

---

## 7. 时间轴 (Timeline) (`timeline.js`)

### 7.1 Canvas 渲染顺序

1. 清空画布 → 背景色填充
2. 音频波形 (3 层: 全频/低频/高频)
3. 时间标尺 (刻度 + 标签)
4. 吸附网格 & 节拍线
5. 轨道分隔线
6. 关键帧菱形 (◊)
7. 音符 (Tap◯ / Keep● / Hold 圆角矩形 + 尾线)
8. A/B 循环标记 (绿色区域 + 标签)
9. 播放头 (红色竖线 + 三角)
10. 鼠标悬停指示线

### 7.2 事件系统

| 事件 | 处理函数 | 行为 |
|------|----------|------|
| `mousedown` | `onTimelineMouseDown` | 右键→开始右拖; 标尺/游标附近→拖播放头; Ctrl+菱形→拖关键帧; 点音符→拖音符; 空白→添加 Tap |
| `mousemove` | `onTimelineMouseMove` | 处理所有拖拽操作, 平移/缩放/吸附 |
| `mouseup` | `onTimelineMouseUp` | 右键拖→创建 Hold/Flick; 清除所有拖拽状态 |
| `mouseleave` | `onTimelineMouseUp` | 同上, 确保鼠标移出画布时清理 |
| `dblclick` | `onTimelineDblClick` | 打开音符编辑弹窗 |
| `wheel` | `onTimelineWheel` | Ctrl+滚轮=缩放, 普通滚轮=平移 |

### 7.3 吸附 (Snap)

- 由 `#chkSnap` 控制开关
- 步长: `majorStep / 4` (动态适应缩放级别)
- Alt 键临时禁用吸附

### 7.4 A/B 循环

- A/B 按钮设置循环起点/终点
- 启用后播放头限制在 [A, B] 区间
- 循环区域绿色高亮渲染

### 7.5 播放头拖动 (`playheadDrag`)

- 点击标尺区域或播放头附近触发
- 更新 `game.gameTime`, 调用 `onSeekNotify()` 处理判定
- 循环启用时受 [A, B] 限制

### 7.6 `onSeekNotify(newTime, oldTime)` (关键函数)

拖动时间轴时处理音符判定:

- **向前拖** (`newTime > oldTime`): 跳过区间的未判定音符标记为静默
- **向后拖** (`newTime < oldTime`): 清空目标位置之后的判定, 以便重新击打

---

## 8. 3D 预览渲染 (`preview.js`)

### 8.1 坐标系

- 屏幕空间: `W × H` px (CSS 尺寸, 乘以 `devicePixelRatio`)
- 归一化空间: `(nx, ny) ∈ [-1, 1]`
- Z 轴: 音符从 `zFar` 向 `zNear` 移动, 速度 `fallSpeed = 5.12 units/s`
- 透视: `project3D(nx, ny, z) = (nx/(z+d), ny/(z+d))`, `d = 1/tan(fov/2)`

### 8.2 渲染循环 (`renderLoop`)

每帧执行:

1. 更新 `game.gameTime` (播放时)
2. A/B 循环跳转逻辑
3. `processAutoJudgments(gameTime)` — 自动判定
4. `processHoldContinuous(gameTime)` — Hold 持续判定
5. `updateAllHoldShrinks()` — 判定环缩小动画
6. `renderPreviewFrame(gameTime)` — 3D 场景
7. `renderTimeline()` — 时间轴
8. `requestAnimationFrame(renderLoop)`

### 8.3 渲染顺序 (`renderPreviewFrame`)

1. 清空背景
2. 网格线
3. 节拍线 (可选)
4. 收集并排序音符 (远→近)
5. 渲染 Hold 轨迹多边形 (20 段细分)
6. 渲染音符圆环/实心点 (按类型)
7. 渲染判定环 (于当前游戏时间)
8. 渲染特效 (扩张环动画)
9. HUD (Combo, 分数, 调试信息)

### 8.4 预览区拖拽创建/修改关键帧

**`mousedown`**:
1. 逆投影鼠标位置
2. 找最近的轨道判定环
3. 如果该时间无关键帧 → 插入新关键帧
4. 调用 `renderTrackPanels()` + `_scrollToKeyframe()`
5. 设置拖拽状态

**`mousemove`** (节流 ~80ms):
- 直接更新 `state.tracks[].keyframes[].x/y`
- 直接更新输入框 DOM (不重建面板, 防止滚动跳动)

**`mouseup`**:
- 调用 `renderTrackPanels()` 完整重建
- 调用 `_scrollToKeyframe()` 确保定位

---

## 9. 音频系统 (`audio.js` + `sfx.js`)

### 9.1 主音频

| 函数 | 功能 |
|------|------|
| `loadAudio(file)` | 读取文件, `decodeAudioData`, 写入 `state.audioBuffer` |
| `startAudioAt(gameTime)` | 从指定时间开始播放, 创建 `AudioBufferSourceNode`, 连接 `GainNode` + 分析器 |
| `stopAudio()` | 停止播放, 断开节点 |
| `pauseAudio()` | 记录偏移, 停止 SourceNode |
| `resumeAudio()` | 从暂停偏移继续播放 |

### 9.2 频谱分析

- `audioEnergy` 对象: `{ data: Float32Array, low, high, bandWidth }`
- 实时更新三个频段的 RMS 能量
- 用于时间轴波形渲染

### 9.3 音效 (`sfx.js`)

- 加载 Tap/Keep/Hold 的 .mp3
- 使用 `<audio>` 元素池 (每个类型 3 个实例)
- `playSfx(type)`: 选择未播放的音频元素播放, 防止重叠

---

## 10. 游戏循环与判定 (`gameplay.js` + `preview.js`)

### 10.1 判定窗口

| 等级 | 窗口 (±ms) | 分数倍率 |
|------|-----------|----------|
| Perfect (P) | ≤ 60ms | 1.0x |
| Good (G) | ≤ 100ms | 0.6x |
| Bad (B) | ≤ 120ms | 0x |
| Miss (M) | > 120ms | 0x |

### 10.2 判定处理

- **预览模式 (auto)**: 所有未映射轨道自动 Perfect
- **游玩模式 (play)**: 按键触发 `handleKeyPress(keyStr)`:
  - 找到按下的键对应的所有轨道
  - 每个轨道找最近的未判定音符
  - 根据时间差返回 P/G/B/M

### 10.3 Hold 持续判定

- Hold 开始判定为 P/G → 进入 `holding` 状态
- 持续按住键直到结束 → `done`
- 中途松手 → `broken`, 倒扣分数

### 10.4 计分

- 每个音符分值 = 1,000,000 / 总音符数
- Perfect = 100%, Good = 60%, 其他 = 0%
- Hold 中途断 = 撤销之前加的分

---

## 11. 键位映射与游玩模式 (`gameplay.js`)

### 11.1 支持多键

`state.keyMap[tid]` 改为字符串数组, 一个轨道可绑多个键:

```javascript
{ 0: ["D", "F"], 1: ["J"], 2: ["K", "L"] }
```

### 11.2 键位映射弹窗

```
[●] 轨道 0    [D] [F] [+]
[●] 轨道 1    [J] [+]
              [+] = 添加键, 悬停 × = 删除
```

| 操作 | 行为 |
|------|------|
| 点击 `[+]` | 进入绑定状态, 显示 `?`, 按键盘键追加 |
| 悬停按键 | 文本变为 `×`, 点击删除 |
| 「确认并开始」 | 右下角, 检查所有轨道至少绑一键 |

### 11.3 默认键位

| 轨道数 | 默认键位 |
|--------|----------|
| 4 | D F J K |
| 5 | D F (space) J K |
| 6 | S D F J K L |
| 7 | S D F (space) J K L |
| 8 | W E R V N U I O |

### 11.4 游玩流程

1. 点击 🎮 → 检测已绑轨道 → 弹出键位映射弹窗
2. 确认后 → `startPlayMode()` → `game.mode = "play"`
3. 渲染循环中 `processAutoJudgments` 跳过已映射轨道
4. 玩家按键 → `handleKeyPress` → 判定
5. 谱面结束 → 弹出结算弹窗 (总分)

---

## 12. 导入导出 (I/O) (`io.js`)

### 12.1 `.hvf` 格式 (JSON 完整格式)

```javascript
{
  version: "2.0",
  bpm: 120,
  sampleRateReal: 10,
  offsetSec: 0,
  duration: 30,
  fov: 55,
  noteScale: 1,
  tracks: [
    {
      id: 0,
      keyframes: [
        { time: 0, x: 0, y: 0 },
        { time: 30, x: 0, y: 0 }
      ],
      segmentPresets: [
        { preset: "line", params: {} }
      ]
    }
  ],
  notes: [
    { track: 0, type: 1, time: 1.5 }
  ]
}
```

### 12.2 `.hvp` 格式 (紧凑十进制编码)

紧凑的纯数字编码, 用于游戏运行时:

```
HVPv1|bpm|sr|offset|duration|fov|note_scale|
  trackCount|
  t0_id|t0_kfCount|t0_time1|t0_x1|t0_y1|t0_h1|...|
  t0_segCount|t0_preset1|t0_params1|...|
  track1...|
  noteCount|n0_track|n0_type|n0_time|n0_end|...|
```

- 坐标存储: `coordToStored(v) = round(v * 1000 + 5000)` (5000 偏置)
- 参数存储: 数字 `coordToStored`, 字符串直接存
- `endTime` 为 0 时表示 `null` (非 Hold)

### 12.3 校验规则 (`validation.js`)

导入时检查:
- `version` 是否支持
- 必填字段是否存在
- 数据类型是否正确
- 时间范围是否合理
- 轨道 ID 是否连续

---

## 13. 撤销/重做 (`undo.js`)

```javascript
var undoStack = [];      // 快照数组
var redoStack = [];

function pushUndo()      // 当前 state 快照入栈
function undo()          // Ctrl+Z: 弹出 undo, 压入 redo, 恢复
function redo()          // Ctrl+Y: 弹出 redo, 压入 undo, 恢复
```

**快照**: `JSON.parse(JSON.stringify(state))` 深拷贝 (不含 `audioCtx`, `audioBuffer` 等非序列化字段).

**触发条件**: 任何编辑操作前调用 `pushUndo()`. 不包含播放/UI 操作.

---

## 14. UI 组件与弹窗

### 14.1 弹窗系统

所有弹窗使用 `.modal-overlay` (全屏遮罩) + `.modal` 容器:

| 弹窗 | ID | 功能 |
|------|-----|------|
| 导入 | `importModal` | 选择格式 + 粘贴/上传 |
| 音符编辑 | `noteEditModal` | 修改轨道/类型/时间/持续时长 |
| 键位映射 | `keyMapModal` | 绑定按键到轨道 |
| 暂停 | `pauseModal` | 继续/重试/退出 |
| 结算 | `resultModal` | 显示总分 |
| 帮助 | `helpModal` | 操作说明 (点遮罩层关闭) |

### 14.2 全局 Esc 关闭 (`ui.js`)

```javascript
document.addEventListener("keydown", function(e) {
  if (e.key !== "Escape") return;
  // 顺序检查各弹窗, 关闭第一个 active 的
  [importModal, noteEditModal, keyMapModal, resultModal, helpModal]
});
```

### 14.3 Toast

`#toast` 元素, `showToast(msg, type)` 显示 3 秒后自动消失.

---

## 15. 样式系统 (`css/style.css`)

### 15.1 CSS 变量

```css
:root {
  --bg: #1a1a2e;           // 最暗背景
  --bg2: #16213e;          // 面板背景
  --bg3: #0f3460;          // 元素背景
  --bg4: #0d1b2a;          // 输入框背景
  --fg: #e0e0e0;           // 主文本
  --fg2: #a0a0c0;          // 次要文本
  --fg3: #6a6a8a;          // 弱文本
  --accent: #e94560;        // 主强调色 (红色)
  --accent2: #533483;       // 次要强调色 (紫色)
  --border: #2a2a4a;       // 边框
  --input-bg: #0d1b2a;     // 输入框背景
  --input-border: #1b2838; // 输入框边框
  --btn: #e94560;           // 按钮主色
  --btn-hover: #ff6b81;    // 按钮悬停
  --success: #2ecc71;      // 成功
  --warn: #f39c12;         // 警告
  --err: #e74c3c;          // 错误
  --topbar-h: 44px;        // 顶栏高度
}
```

### 15.2 关键布局

```
body (overflow: hidden)
  header#topbar (height: var(--topbar-h))
  .main-layout (flex, height: calc(100vh - topbar-h))
    aside#leftPanel (width: 38%, flex column)
      #trackDirectory
      #trackPanelsContainer (flex:1, overflow-x: auto)
        #trackPanels (flex row)
          .track-panel (flex: 0 0 370px)
    main#rightPanel (flex:1, flex column)
      #topRow (flex)
        #previewSection
        #globalSection
      #timelineSection
```

### 15.3 预定义 ICON 按钮 (Scratch 风格)

```css
.icon-btn {
  width: 26px; height: 26px;
  background: var(--accent2);  // 紫色闲置
  color: #fff;
  border-radius: 4px;
}
.icon-btn:hover {
  background: var(--btn);  // 红色悬停
}
```

---

## 16. 常见修改指南

### 16.1 新增运动预设

1. 在 `presets.js` 的 `PRESETS` 中添加新预设
2. 定义 `params` 和 `fn(localP, fromKF, toKF, P)`
3. 在 `PARAM_DEFAULTS` 和 `PARAM_LABELS` 中添加对应的参数

### 16.2 修改面板宽度

搜索 `flex: 0 0 370px` 在 `css/style.css` 中修改.

### 16.3 修改输入框样式

`.float-label` 相关 CSS 在 `css/style.css` 浮动标签部分.

### 16.4 修改判定参数

在 `gameplay.js` 的 `getJudgment(diffMs)` 函数中调整窗口阈值.

### 16.5 修改导出格式

在 `io.js` 的 `encodeHVP` / `encodeHVF` 函数中修改.

### 16.6 添加全局快捷键

在 `init.js` 或 `ui.js` 的 `document.addEventListener("keydown", ...)` 中添加.

### 16.7 滚动行为

- 面板垂直滚动自动恢复: `tracks.js` → `renderTrackPanels()` → `scrollTopMap`
- 面板间同步滚动: `tracks.js` → `panel.onscroll`
- 标签页切换: `tracks.js` → `scrollToTrack(tid)` → `scrollTo({ left: pi * 370 })`
- 关键帧聚焦: `preview.js` → `_scrollToKeyframe()`

### 16.8 DOM 重建保护

`renderTrackPanels()` 是核心渲染函数, 频繁调用. 每次重建会:
1. `blur()` 失焦防止浏览器滚动
2. `replaceChildren(frag)` 原子替换防止闪烁
3. `_syncingScroll = true` 恢复垂直滚动, 防止 `onscroll` 互相覆盖

**不要在拖拽中调用此函数**. `preview.js` 的 mousemove 处理器已改为直接更新 DOM 而不重建.

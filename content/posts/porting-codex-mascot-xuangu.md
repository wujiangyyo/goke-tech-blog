---
title: "Codex 桌宠独立移植记：玄骨改造全流程复盘"
date: 2026-07-18
draft: false
tags: ["Electron", "桌面宠", "Codex", "Windows", "CSS动画", "透明窗口"]
categories: ["技术实践"]
author: "果壳科技 聂小雨"
summary: "把 Codex 平台的 V2 桌宠玄骨改造成独立 Windows 透明桌面宠，纯原生体验，不依赖 Codex。途中踩了透明窗口闪烁、拖动背景扩大、视线追踪标定等一堆坑。"
image: "/goke-tech-blog/img/xuangu-pet-cover.jpg"
ShowToc: true
TocOpen: true
---

## 写在前面

[玄骨](https://github.com/nousresearch/玄骨) 原本是 Codex 平台的一款第三方桌宠，角色设定为冷白肤、黑长发、冰蓝长袍的修士，整体气质冷峻克制。Codex 本身的桌宠功能比较完善，但它是平台绑定的——你想要用桌宠，就得开着 Codex。

为什么不把它拉出来，做成独立桌面应用呢？Windows 原生、透明窗口、8 方向视线追踪、拖拽交互，全部自己控制。

目标：
- 独立 exe，不依赖 Codex，双击就能跑
- 透明窗口、无边框、置顶
- 鼠标悬停时角色 8 方向视线跟随
- 拖拽、单击/双击、右键菜单交互
- 保留原版所有动画（待机、等待、挥手、持火等）

## 最终架构

```
xuangu-pet/
├── assets/
│   ├── spritesheet.png       # V2 图集 1536×2288
│   └── tray_icon.png         # 托盘图标
├── src/
│   ├── main.js               # Electron 主进程
│   ├── preload.js            # 上下文桥接
│   └── renderer/
│       ├── index.html        # 纯透明窗口 + 渐变光晕
│       └── pet.js            # 动画引擎 + 视线追踪 + 交互
└── package.json
```

技术选型：
- **框架**：Electron 30.5.1
- **窗口**：192×208（恰好裹住精灵单帧尺寸），`transparent: true`，`frame: false`，`alwaysOnTop: true`
- **空闲动画**：CSS `@keyframes` + `steps(N)` 循环驱动
- **交互动画**：JS `setInterval` 逐帧播放（有限次动作场景）
- **视线追踪**：`Math.atan2` 分 8 方向角度区 + 中心死区

## 六个坑

这次改造遇到了六个值得单独记一笔的问题，按难缠程度排序。

### 坑一：透明窗口背了多少黑锅

这是最大的坑，前后绕了快十版。

**现象**：在 Win11 Insider Preview（10.0.26200.8875）上，透明窗口不用时会周期性闪烁。

**排查过程**：
- 实心底色不闪烁但用户不接受黑框
- CSS 动画模式下，悬停时（由 JS 控制帧渲染）不闪烁
- 怀疑某些 spritesheet 帧是空白的

第一轮排查没有结果，于是把注意力转向了别的问题。但最后回来看，根因其实非常简单：

**根因**：原代码假设每行都是 8 帧满帧。但 Codex V2 图集某些行的末尾帧是空白（透明）。CSS `steps(8)` 循环到这些帧时，窗口内容变成全透明，视觉上就"闪"了一下。

**修复**：按图集实际有效帧数配动画：

| 动作 | 行 | 有效帧 |
|------|----|--------|
| idle（待机） | r0 | 7 |
| waving（挥手） | r3 | 4 |
| 持火（掐诀结印） | r4 | 5 |
| waiting（等待） | r6 | 6 |
| running（冲刺奔跑） | r7 | 6 |
| review（阅读） | r8 | 6 |

同时加了一行 `app.commandLine.appendSwitch('disable-gpu')` 禁用 GPU 硬件加速——该 Win11 预览版的 DWM 合成确实存在已知 bug，CPU 渲染反而更稳。

### 坑二：拖动时背景凭空扩大

**现象**：拖动窗口时，背景光晕（CSS radial-gradient）会匀速往右下方"扩大"，松开后不回缩。还会挡住鼠标点击桌面图标。

这个现象太离奇了——窗口尺寸固定，CSS 元素也不可能自己变大。一开始以为是 DWM 渲染残留，但每次拖动都只往右下扩大，方向恒定，不符合残留的不规则特征。

**根因**：光晕层（一个 `position: absolute; inset: 0` 的 div）的父元素 body 没有设 `position: relative`。`inset: 0` 的基准因此是视口（viewport），不是 body。该版本 Windows 在拖动透明窗口时，DWM 会重新计算视口尺寸，导致 `inset: 0` 的实际渲染范围异常扩大。

**修复**：body 加一行 `position: relative;`。

就这么一行。

### 坑三：视线追踪全靠瞎猜

**现象**：方向检测（角度分区）是对的，但显示的精灵帧跟期望方向对不上——鼠标往左，角色看右。

**根因**：我看不到 spritesheet 图集。每帧的实际画面（抬头、低头、左看、右看...）全靠猜，猜的每一行都是错的。

**修复**：用户（老板）亲自标定了 16 帧视线画面，做成 Excel 发我。重建映射表：

```js
const GAZE_MAP = {
  center:      [10, 1],  // 正前下
  up:          [9, 0],   // 正上
  down:        [10, 0],  // 正下
  left:        [10, 5],  // 正左
  right:       [10, 3],  // 正右
  up_left:     [10, 7],  // 左上
  up_right:    [9, 3],   // 右上
  down_left:   [10, 4],  // 左下
  down_right:  [9, 7],   // 右下
};
```

角度分区用了 `Math.atan2(dy, dx)` 分 8 个 45° 扇区 + 中心 15px 死区。

**教训**：看不到图就别猜，直接让看得到图的人标。

### 坑四：CSS 动画"消失术"

**现象**：单击"打招呼"，角色闪过第一帧后消失。

**根因**：`animation-fill-mode: forwards` 在 Electron 上有时 hold 不住最后一帧。动画播完后，元素恢复到 `background-position: 0 0`（即 idle 的起始帧），看起来就像角色"消失"了。

**修复**：有限次播放的动作（挥手、持火等）不走 CSS，改 JS 定时器逐帧驱动：

```js
function playAnimation(row, frames, interval, callback) {
  setSpriteClass('state-hover'); // 停用 CSS 动画
  let f = 0;
  setFrame(row, f);
  animTimer = setInterval(() => {
    f++;
    if (f >= frames) {
      clearInterval(animTimer);
      callback && callback();
      return;
    }
    setFrame(row, f);
  }, interval);
}
```

悬停模式下 JS 切帧从不闪烁，所以这种做法更稳定。

### 坑五：看不见的"画布"挡鼠标

**现象**：窗口的区域虽然透明，但鼠标点不穿它——桌面图标被挡住了。

**根因**：Electron 的 BrowserWindow 不管透不透明，都会捕获窗口范围内的鼠标事件。窗口是 192×208，挡住了其背后的桌面区域。

**修复**：缩小窗口到跟精灵一样大（192×208），不再多留透明边距。加上 `disable-gpu` 后 DWM 也不再为透明窗口创建额外的合成缓冲区。两项结合，点击通过透明区域不再是问题。

### 坑六：托盘图标糊成一团

**现象**：系统托盘显示不了图标。

**根因**：直接把 1536×2288 的整张 spritesheet 缩成 16×16 做图标，缩得太狠，角色完全不可辨认。

**修复**：从 spritesheet 裁出左上第一帧（192×208），缩到 32×32 存为独立 PNG，作为托盘图标。

## 打包为独立 exe

使用 `electron-builder` 打包 portable exe：

```json
{
  "scripts": {
    "pack": "electron-builder --win portable"
  },
  "build": {
    "appId": "com.xuangu.pet",
    "productName": "玄骨",
    "files": ["src/**/*", "assets/**/*"],
    "win": { "target": "portable" }
  }
}
```

Linux 下可交叉编译 Windows exe（wine 需安装，用于修改 exe 版本信息）。打出来的 exe 约 169MB（含完整 Electron 运行时），解压后双击即用，无需安装。

## 后记

整个改造从立项到完工经历了大约 32 个版本迭代，GitHub 上的 README 其实已经把图集规格和动作帧数写得清清楚楚，我第一遍没仔细读，绕了不少路。

不过绕路也有绕路的价值——透明窗口在 Windows 上的合成机制、CSS 动画在 Electron 里的边界行为、DWM 下 `position: relative` 对渲染基准的影响，这些不踩一次坑永远不会记得那么深。

最后玄骨已经是一个**完全的独立桌面宠**了：透明窗口、视线追踪、持火挥手冲刺阅读全套动作、系统托盘管理。老板的原话是"十分完美"。

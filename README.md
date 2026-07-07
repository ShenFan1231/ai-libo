# Vue + PixiJS + GSAP Limbo Demo

这是一个用于理解线上 Limbo 类游戏前端结构的最小 demo。它不是完整真钱游戏，也没有接入后端接口、钱包、风控、下注结算或随机数服务；它只演示前端技术分层：

- Vue 3 负责页面 UI、表单状态、余额展示、历史记录和按钮交互。
- PixiJS 负责中间游戏舞台，也就是 `<canvas>` 里的图形、曲线、火箭和网格。
- GSAP 负责动画时间线，例如倍率增长、飞行动画、火焰闪烁和结束飞出效果。

## 文件结构

```text
vue-pixi-gsap-limbo-demo/
├── index.html
└── README.md
```

当前 demo 为了方便查看，全部代码都写在 `index.html` 里：

- `<style>`：页面布局、控制面板、舞台容器、响应式样式。
- `<template>` 部分：实际写在 `#app` 根节点内，由 Vue 接管。
- `<script>`：Vue 逻辑、PixiJS 初始化、GSAP 动画逻辑。

如果要改成正式项目，建议拆成：

```text
src/
├── App.vue
├── components/
│   ├── BetPanel.vue
│   ├── GameStage.vue
│   └── RoundHistory.vue
├── game/
│   ├── createPixiStage.js
│   ├── drawScene.js
│   └── roundAnimation.js
└── main.js
```

## 运行方式

### 方式一：直接打开

直接双击或在浏览器中打开：

```text
index.html
```

这个 demo 使用 CDN 加载依赖：

```html
<script src="https://unpkg.com/vue@3/dist/vue.global.prod.js"></script>
<script src="https://pixijs.download/release/pixi.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js"></script>
```

所以需要浏览器能访问这些 CDN。

### 方式二：启动本地静态服务

进入 demo 目录后运行：

```bash
python -m http.server 5173
```

然后访问：

```text
http://localhost:5173
```

如果没有 Python，也可以用任意静态服务器，例如 VS Code 的 Live Server。

## 技术分工

### Vue 负责什么

Vue 主要管理 DOM 和业务状态：

- 投注金额：`betAmount`
- 自动兑现倍率：`cashoutAt`
- 余额：`balance`
- 当前倍率：`multiplier`
- 当前回合是否运行：`isRunning`
- 是否已经兑现：`cashedOut`
- 回合状态文本：`roundLabel`
- 历史记录：`history`

这些状态会驱动界面展示，例如：

```html
<div class="multiplier">{{ multiplier.toFixed(2) }}x</div>
<button :disabled="isRunning" @click="startRound">BET</button>
<button :disabled="!isRunning || cashedOut" @click="cashOut">CASH OUT</button>
```

也就是说，Vue 不直接画游戏画面，它只负责页面层 UI 和状态绑定。

### PixiJS 负责什么

PixiJS 负责创建和维护 canvas 舞台：

```js
app = new PIXI.Application();
await app.init({
  backgroundAlpha: 0,
  antialias: true,
  autoDensity: true,
  resizeTo: stageEl.value,
});
```

创建好后，把 Pixi 的 canvas 插入页面：

```js
stageEl.value.appendChild(app.canvas);
```

demo 里 PixiJS 画了这些东西：

- 背景网格：`grid`
- 飞行曲线：`curve`
- 曲线光晕：`trail`
- 火箭容器：`rocket`
- 火焰图形：`flame`
- Pixi 文本：`pixiText`

核心绘制函数是：

```js
drawScene(progress)
```

`progress` 从 `0` 到 `1`，表示飞行动画进度。函数会根据进度计算火箭的位置，然后重新绘制曲线：

```js
const endX = pad + (width - pad * 2) * progress;
const endY = baseY - (height - pad * 2) * Math.pow(progress, 0.72);
```

### GSAP 负责什么

GSAP 在 demo 中主要负责两类动画。

第一类是倍率增长动画：

```js
roundTween = gsap.to(state, {
  value: crashAt,
  duration: 1.8 + Math.random() * 1.4,
  ease: "power2.in",
  onUpdate: () => {
    multiplier.value = state.value;
    drawScene(Math.min((state.value - 1) / 4, 1));
  },
  onComplete: () => {
    endRound(crashAt);
  },
});
```

这里 GSAP 不直接操作 DOM，而是改变 `state.value`。每一帧更新时：

1. 把 `state.value` 写入 Vue 的 `multiplier`。
2. 调用 Pixi 的 `drawScene()` 重画舞台。
3. 检查是否到达自动兑现倍率。

第二类是循环动效，比如火焰闪烁：

```js
pulseTween = gsap.to(flame.scale, {
  x: 1.25,
  y: 1.25,
  duration: 0.16,
  repeat: -1,
  yoyo: true,
  ease: "sine.inOut",
});
```

这段直接作用于 Pixi 对象的 `scale`，所以火焰会持续有呼吸感。

## 回合流程

一个完整回合大概是：

```text
点击 BET
  ↓
扣除 betAmount
  ↓
随机生成 crashAt
  ↓
GSAP 推动 multiplier 从 1.00x 增长到 crashAt
  ↓
每帧更新 Vue 倍率文本，并调用 Pixi 重绘火箭位置
  ↓
如果到达 cashoutAt，自动调用 cashOut()
  ↓
动画结束，调用 endRound()
  ↓
更新历史记录，重置舞台
```

相关函数：

- `startRound()`：开始一局。
- `cashOut()`：兑现当前倍率。
- `endRound(crashAt)`：结束一局，写入历史。
- `resetRound()`：重置倍率和舞台。
- `drawScene(progress)`：绘制 Pixi 舞台。

## 和真实线上游戏的区别

这个 demo 只模拟前端表现。真实游戏通常还会有：

- 登录态和 auth token。
- 币种、语言、主题、运营商配置。
- 后端下注接口。
- 后端结算接口。
- 公平性证明或随机数种子。
- WebSocket 推送回合状态。
- 错误处理、断线重连、余额同步。
- 风控、限额、审计日志。

真实项目里，`crashAt` 不应该由前端随机生成。前端最多只做展示，最终结果应该来自后端。

demo 里这段只是为了演示效果：

```js
const crashAt = Number((1.05 + Math.random() * 5.2).toFixed(2));
```

正式项目应改成接口返回，例如：

```js
const { roundId, crashAt } = await api.createBet({
  amount: betAmount.value,
  cashoutAt: cashoutAt.value,
});
```

## 控制台验证

打开 DevTools 后，可以检查页面使用的库：

```js
Object.keys(window).filter((key) => /pixi|gsap|vue/i.test(key))
```

可以查看 Pixi app：

```js
window.__PIXI_APP__
```

可以查看 Pixi 版本：

```js
PIXI.VERSION
```

可以查看 GSAP 版本：

```js
gsap.version
```

这个 demo 故意暴露：

```js
window.__PIXI_APP__ = app;
```

线上项目有时也会为了调试暴露类似对象。如果你在真实站点控制台看到 `__PIXI_APP__`，基本就能判断它使用了 PixiJS。

## 如何扩展成更真实的版本

### 1. 拆分 Vue 组件

可以把投注面板和舞台拆开：

```text
BetPanel.vue
GameStage.vue
RoundHistory.vue
```

`GameStage.vue` 内部只负责 Pixi 初始化和销毁；外部通过 props 传入倍率、回合状态等。

### 2. 使用 Vite

正式项目建议用 Vite：

```bash
npm create vite@latest limbo-demo -- --template vue
cd limbo-demo
npm install
npm install pixi.js gsap
npm run dev
```

然后使用模块导入：

```js
import { Application, Graphics, Container, Text } from "pixi.js";
import { gsap } from "gsap";
```

这样比 CDN 更适合工程化开发。

### 3. 后端驱动回合

前端可以保留 GSAP 动画，但回合结果由后端提供：

```text
POST /api/bet
GET /api/round/:roundId
WS  /api/rounds/stream
```

前端只根据后端返回的数据播放动画。

### 4. 加载真实素材

Pixi 可以加载图片、精灵图、骨骼动画等资源：

```js
const texture = await PIXI.Assets.load("/assets/rocket.png");
const sprite = new PIXI.Sprite(texture);
```

当前 demo 为了保持自包含，用 `PIXI.Graphics` 画了简化火箭。

## 常见问题

### 为什么页面里既有 DOM 又有 canvas？

这是 Web 游戏常见结构。DOM 适合做表单、按钮、弹窗、文字排版；canvas 适合做高频动画、粒子、曲线和游戏画面。

### 为什么不用纯 Vue 画动画？

Vue 可以做 UI 动画，但频繁更新复杂图形时，canvas 更合适。PixiJS 对 WebGL/Canvas 渲染做了封装，适合 2D 游戏画面。

### 为什么要用 GSAP？

GSAP 的时间线、缓动、暂停、恢复、kill 动画都很成熟。它可以同时操作普通对象、DOM、Pixi 对象，非常适合做游戏回合动画。

### PixiJS 是游戏引擎吗？

PixiJS 更准确地说是 2D 渲染引擎。它不直接提供完整游戏框架里的关卡、物理、场景管理、资源管线等全部能力，但很适合拿来做 H5 游戏的渲染层。

### 这个 demo 是否可以直接用于生产？

不建议直接用于生产。它适合学习架构和验证思路。生产项目需要加入接口、安全、异常处理、资源加载、状态机、测试和构建流程。

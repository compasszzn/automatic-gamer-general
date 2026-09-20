# Web/HTML5 Playbook — 自动化游戏助手落地手册

> 读者前提：已读过引擎无关版 SKILL.md，即将在 Web/HTML5 游戏工程上落地 bot。本文只写 web 落地差异，不复述通用流程（分析→策略→编码→sweep→迭代）。序列 schema 与唯一注入口原则见 `references/sequence-record-replay.md`，事件字段必须与它对齐。Unity WebGL 导出的游戏见 §8 并转读 `engine-unity.md`。

## 0. 通用概念 → Web 技术对照表

| 通用概念 | Web 做法 | 注意事项 |
|---|---|---|
| 帧索引 | 游戏主循环自增的 tick 计数（rAF 驱动即 rAF 次数），对局开始为第 0 帧 | 浏览器没有 `Time.frameCount` 等价物，在调试桥里自建计数器；录制/回放都以它对齐，不用墙钟 |
| 不受速度影响的时钟 | `performance.now()`（单调，毫秒），记秒时除以 1000 | web 的 speed 是游戏代码自己的 dt 乘子，不缩放浏览器时钟 → t 天然 unscaled；但 rAF 节流会让 tick 本身变慢（§5.3） |
| 游戏速度控制 | 定位主循环 delta 乘子（分析阶段必须找到），经调试桥暴露 `setSpeed(x)` | 没有全局 timeScale API；PixiJS 有 `app.ticker.speed` 现成乘子，其余多在自研循环里找（§5.1） |
| 状态读取 | 路线 A：import 游戏模块 / 访问全局 game 对象；路线 B：`page.evaluate` 读调试桥 | 一律经调试桥（§3），禁止 bot 散装引用游戏对象 |
| 输入注入 | ① CDP/Playwright 输入（isTrusted=true）② 游戏输入管线直写 ③ DOM `dispatchEvent`（isTrusted=false，最弱） | 唯一注入口 `window.__AGinject`（§4）；isTrusted 是 web 落地第一坑 |
| 视频录制 | 优先 Playwright `recordVideo`（每 context 一个 webm）→ ffmpeg 转 mp4；备选 CDP screencast、`canvas.captureStream`+MediaRecorder | 见 §6；headless WebGL 软渲染需确认，黑帧排查见 §6 |
| 存档目录（localStorage/IndexedDB） | origin 级存储，browser context 天然按 context 隔离 | web 相对其他引擎的最大优势：并发 sweep 存档隔离开箱即用（§7）；fresh run 仍要防持久化残留（§8） |
| 无头运行 | headless Chromium：页面视为可见，rAF 正常触发 | WebGL 走 SwiftShader 软渲染（CPU）；别盲加 `--disable-gpu`；新版本 WebGL 兜底已弃用自动回退（§6） |
| 参数传入（URL query/环境变量/注入配置） | URL query 最简；build-time 注入 `__AUTO_GAMER_CONFIG__`；Playwright `addInitScript` | 三种方式对应通用流程的命令行/环境变量语义，见 §7 |
| 构建 | `npm run build`（vite/webpack/esbuild）→ dist/；纯静态 JS 无需构建 | bot 代码必须过游戏的 build 与类型检查；sourcemap 是否产出影响运行时定位成本（§3/§8） |

## 1. 技术栈识别

判定顺序：package.json → index.html → 构建产物。三者交叉验证，别只看一处。

**package.json 看：**
- `dependencies`：`phaser` / `pixi.js` / `three` → 框架（下表）；再看 `scripts.build`/`scripts.dev` 与 `devDependencies`（vite / webpack / esbuild / typescript）
- 构建配置里是否产出 sourcemap：vite 看 `build.sourcemap`，webpack 看 `devtool`——影响 §3/§8 的运行时定位成本

**index.html 看脚本引用形态：**
- `<script type="module" src="/src/main.ts">` → bundler 开发模式，源码工程在 `src/`，可直接加 bot 模块
- `<script src="assets/bundle.xxxx.js">` → 只有产物（本 skill 前提是有源码，产物仅做运行对照）
- CDN 引用（unpkg/jsdelivr 的 phaser.min.js 等）→ 框架在 window 全局上，无构建链

**构建产物特征：** dist 内文件名带 hash（vite/webpack 产物）；存在 `.map` 才有 sourcemap；出现 `.wasm` + `Build/*.json` + loader 脚本 → 是编译到 wasm 的引擎（Unity WebGL 见 §8）。

| 框架 | 源码特征 | 运行时特征 | 对 bot 的直接影响 |
|---|---|---|---|
| Phaser | `new Phaser.Game(config)` | `window.Phaser`；`game.scene.scenes` 可遍历场景 | 自带输入系统与场景管理；无单一全局速度乘子（物理 `world.timeScale`、tween、timer 各自独立），加速要找游戏自己的逻辑 dt |
| PixiJS | `new Application(...)` / `app.ticker` | `window.PIXI`；`app.stage` | 只管渲染，输入/逻辑多为自研；`app.ticker.speed` 是现成速度乘子 |
| Three.js | `new THREE.WebGLRenderer` | `renderer` / `scene` / `camera` | 主循环通常自研 rAF，速度乘子在自研循环里找 |
| 自研 canvas/WebGL | `canvas.getContext('2d'/'webgl')` + 手写 loop | 全局对象少 | 输入监听直接挂在 window/canvas 上，注入点最直接 |

**TypeScript/ESM/bundler 对"往游戏里加 bot 代码"方式的影响：**
- ESM + bundler（vite 最常见）：bot 建成 `src/auto_gamer/` 普通 ES 模块，从入口 import——最干净，还能用 dev server 热更快速迭代
- 带 TypeScript：bot 代码必须通过 tsc 类型检查，否则 `npm run build` 直接红。访问游戏内部可用 `as any` 逃生，但引用的类/字段名必须真实存在
- webpack / esbuild / tsc-only：同为模块注册进入口链，无本质差异
- 无构建纯 `<script>`：bot 只能做成独立脚本标签，拿不到模块内部，只能依赖全局对象——把调试桥（§3）作为唯一抓手，等价于路线 B 的页面内变体

## 2. Bot 代码隔离与组织

两条落地路线，先定路线再写代码：

**路线 A 内挂式（主路线）** —— bot 作为源码模块打进游戏：
- 代码全部放 `src/auto_gamer/`（决策引擎、观察器、注入器、回放器、log/视频记录都在页面内），入口 `main.ts` 只加 3-5 行：按 URL 参数 / `import.meta.env` / 注入配置决定是否激活
- 最小侵入红线：只新增文件 + 入口少量挂载；修改游戏原文件逐条登记
- 优势：有源码、可读任意内部状态、注入可直达游戏输入管线——决策质量上限最高
- 注意：URL 开关是运行时的，bot 代码仍进 bundle；对外发布游戏需构建期剥离（vite `define` / webpack DefinePlugin 注入常量，让 dead-code elimination 删掉）

**路线 B 外部驱动式** —— Playwright/CDP 从浏览器外驱动：
- 完全不改游戏代码：`page.evaluate` 读状态、`page.mouse` / `page.keyboard` 注入（isTrusted=true，§4）
- 天然适合：序列回放（Node 侧读 `sequence.json` 按帧注入）、黑盒等价性验证、批量并发
- 局限：读不到未暴露的内部状态（除非页面有桥）；每轮读状态跨 CDP 往返有毫秒级延迟，高频决策受限

**推荐组合：A 为主，B 用于等价性验证与黑盒对照。** 页面内桥（§3）+ Node sweep 脚本：bot 在页面里跑，Node 只负责"开 context → 传参 → 等终局 → 收产物 → close"。等价性验证时 B 单独驱动 replay 局作对照。

**sweep 语义换算：** 通用流程里"启动一个游戏进程"= web 下 `browser.newContext()` 开一个 browser context（内含一个 page）；"进程退出" = `context.close()`；并发度 = 同时存活的 context 数（§7）。

## 3. 游戏状态读取（观察器）

**推荐"调试桥"模式——路线 A 的标准形态。** 游戏启动完成（进入可玩状态）后暴露：

```js
window.__autoGamer = {
  ready: true,
  frame(),              // 对局开始后的主循环 tick 计数（序列的帧索引来源）
  t(),                  // performance.now()/1000，秒
  readState(),          // 可序列化快照：bot 决策输入（敌人/HP/冷却/棋盘…按分析报告列字段）
  subscribe(type, cb),  // 游戏事件：level_start / level_end / upgrade_offer / player_dead…
  markEvent(name, data),// battle log 关键事件打点
  setSpeed(x), getSpeed(),
}
```

- bot 的所有模块（决策/观察/回放/录制/log）**只经桥读写**，不散装引用游戏对象——与游戏代码解耦：游戏重构不牵连 bot；录制、回放、battle log 全挂桥上
- `readState()` 返回**可序列化**对象（禁止返回 class 实例/canvas/DOM 引用）；路线 B 直接 `page.evaluate(() => window.__autoGamer.readState())` 复用
- `subscribe` 用于终局检测与阻塞 UI 事件推送；`markEvent` 对应 battle log 关键事件
- 桥自己 import 游戏模块取数——"桥耦合游戏，bot 只耦合桥"。桥的属性名（`__autoGamer`）是自己起的，不受产物压缩影响

**source map 与压缩产物：**
- 本 skill 前提是有全部源码——**分析以 `src/` 为准**，dist/ 只在"打包后行为与源码不符"时做对照
- 运行时报错栈来自压缩产物：有 `.map` 就用 sourcemap 还原定位；没有就按函数名/字符串常量在产物里反查（§8）
- 路线 B 无桥时的读法：`page.evaluate` 里找全局 game 对象（Phaser 常见 `window.game`；自研游戏找挂 window 的 manager）。找不到就回源码加一行暴露——有源码就别硬绕

## 4. 输入注入与唯一注入口

### 4.1 第一坑：isTrusted

- 症状：`dispatchEvent(new KeyboardEvent('keydown', {...}))` 事件发出去了、游戏毫无反应；或 UI 收到事件但逻辑不执行
- 根因：DOM 合成事件 `isTrusted=false`——游戏/框架常检查它；浏览器还会剥掉合成事件的默认行为。附带陷阱：**合成的 MouseEvent 不会派生 pointer 事件**，只监听 `pointerdown` 的游戏收不到你 dispatch 的 MouseEvent
- 有源码时的三种解法（按优先级）：
  1. **送进游戏自己的 input pipeline**（最贴）：找到游戏的 keydown 监听器/输入 state 对象，把事件对象直接交给其处理函数或直写字段——绕开 isTrusted 整套问题
  2. **移除检查**：源码搜 `isTrusted`，删掉或 gate 成"auto 模式跳过"（登记修改）
  3. **CDP 注入**：CDP `Input.dispatchKeyEvent` / `Input.dispatchMouseEvent` 产生浏览器级真实输入，`isTrusted=true`、默认行为与 pointer 派生事件俱全——**Playwright 的 `page.keyboard` / `page.mouse` 底层就是它**。注意：CDP 只能从浏览器外打（路线 B 或 Node 侧），页面内 JS 做不到 CDP 级

### 4.2 InputInjector：唯一注入口

与 Unity 版同一设计原则：一个函数集，所有输入都经它；每次注入 push 一条记录进序列数组。

路线 A（页面内）：

```js
window.__AGinject = {
  axis(vec),               // 移动轴
  keyPress(k), keyRelease(k),
  mouseWarp(pos), mousePress(btn, pos), mouseRelease(btn, pos),
  uiCall(targetPath),      // DOM overlay 直调（见 4.4）
}
// 每个函数内部：先执行注入，再 push {frame: __autoGamer.frame(), t: __autoGamer.t(),
//                                  device, action, ...} 进序列数组
```

路线 B（Node 侧）：`page.keyboard` / `page.mouse` 的每次调用配一条记录；frame/t 在注入前用 `page.evaluate(() => window.__autoGamer.frame())` 取。回放时按事件的 `inject` 分路：`cdp_input` 用 `page.keyboard`/`page.mouse` 重放，`game_state`/`handler_call` 用 `page.evaluate` 调 `__AGinject` 同名入口——注入与回放必须走同一条路。

### 4.3 序列事件 schema（与 sequence-record-replay.md 对齐）

事件字段一一对应，web 取值约定：

| schema 字段 | web 取值 | 说明 |
|---|---|---|
| `frame` | 对局开始后的主循环 tick 计数（rAF 驱动即 rAF 计数） | 第 0 帧 = 对局开始帧（与回放文档同款约定）；不用墙钟对齐 |
| `t_unscaled` | `performance.now()/1000`（秒） | 浏览器时钟不受游戏 speed 缩放，天然 unscaled |
| `device` | `"keyboard"` / `"mouse"` | 同 schema |
| `action` | `press` / `release` / `axis` / `warp` | 同 schema；`warp` = 光标移位 |
| `key` / `button` | 按键名 / `left`/`right`/`middle` | 同 schema |
| `pos` | 屏幕像素坐标，**左下原点（schema 惯例）** | **web DOM 坐标是左上原点——桥内统一翻转：`schemaY = canvasHeight - domY`**；注入与回放共用同一约定，否则回放点错位置 |
| `vec` | 归一化移动向量 | 移动轴节流规则照抄 schema：值不变不重复记，回放保持上次值 |
| `inject` | `game_state` / `dom_dispatch` / `cdp_input` / `handler_call` | 对应 Unity 版 `queue_state_event`/`warp_cursor`/`execute_events` 的语义位：标注走了哪条注入路径，**回放必须同路** |

录制规则沿用回放文档：单行紧凑 JSON、原子写（临时文件 + rename）；meta 里 `input_system` 填实际管线（如 `"phaser-input"` / `"dom-listener"` / `"cdp"`）。

### 4.4 移动轴与 UI 点击的注入形态

**移动轴（`action:"axis"`）——两种形态，选一种贯穿录制与回放：**
- 游戏**维护自己的 input state 对象**（每 tick 读 `keys.down` / 轴向量字段）→ 桥直写字段，`inject:"game_state"`——最稳，无 isTrusted 问题，首选
- 游戏**监听 DOM 事件** → `dispatchEvent`（`inject:"dom_dispatch"`，注意 4.1 的 isTrusted/pointer 坑）或走 Node 侧 CDP（`inject:"cdp_input"`）

**UI 点击——DOM overlay 按钮的两种路径，等价性差异必须用 `inject` 标注：**

| 路径 | 序列形态 | 等价性 |
|---|---|---|
| 直调 handler（有源码：`querySelector` 找到按钮，直接调它的事件处理函数） | `target: "Overlay/HUD/BtnStart"` + `inject:"handler_call"` | 回放必须同样直调同一 target；它绕过了遮挡/hit-test/禁用态——坐标点击复现不了 |
| CDP 坐标点击（`page.mouse` 或 `Input.dispatchMouseEvent`） | `pos: [x, y]` + `inject:"cdp_input"` | 与真实玩家同管线；按钮被遮挡/动画中禁用时行为与直调不同 |

对应回放文档 ExecuteEvents 行的约定：handler 直调记 `target` 不记 `pos`；两种方式混录时回放按 `inject` 分路执行，并在回放侧断言"同一事件同一注入方式"，防止路径混用静默漂移。

## 5. 时间、速度与暂停

### 5.1 游戏速度：找到 dt 乘子（分析阶段必须完成）

- web 没有全局 timeScale。加速 = 主循环里 delta 的乘子，典型形态：

```js
function loop(t) {
  requestAnimationFrame(loop);
  const dt = Math.min((t - last) / 1000, MAX_DT) * SPEED;   // ← 找这个 SPEED
  last = t; update(dt); render();
}
```

- 分析阶段**必须**定位乘子变量并经桥暴露 `setSpeed`/`getSpeed`；同时记录 dt 的 clamp 上限（`MAX_DT`）——高速跑局时帧间隔被放大，clamp 会吃掉加速（Unity 版 timeScale 陷阱的 web 变体）
- 框架捷径：PixiJS `app.ticker.speed` 是官方速度乘子；Phaser 无单一全局乘子（物理/tween/timer 各有 timeScale），要找游戏自己的逻辑 dt；Three.js/自研必在自研循环里
- **每帧重断言 speed**：与 Unity 版同款陷阱——暂停 UI 恢复时常把速度重置回 1，bot 越频繁开面板越接近全程 1×。对策相同：每帧把 `SPEED` 断言回目标值

### 5.2 CDP Emulation.setVirtualTimePolicy：别当加速用

- CDP 有 `Emulation.setVirtualTimePolicy`（**Experimental**）：参数 `policy: "advance" | "pause" | "pauseIfNetworkFetchesPending"` + `budget`（虚拟毫秒数，耗尽后暂停并发 `virtualTimeBudgetExpired` 事件）——把真实时间替换为合成时间源，为确定性测试设计
- 对 rAF 游戏不适合：合成时间服务于 Blink 调度器的 timer/任务，**渲染与合成器按真实 vsync 走**——快进 timer 时画面不跟着画，视频录制（同为合成器依赖）与 audio/墙钟相关逻辑全部脱钩
- 结论：**sweep 加速一律走游戏自己的 dt 乘子**。setVirtualTimePolicy 只在"纯定时器驱动 + 要确定性"的实验里考虑，且先标【待验证】再上：跑两局对比虚拟时间下与真实时间下 rAF 帧率、视频画面、终局结果是否一致，重点看画面是否随 timer 同步推进

### 5.3 rAF 节流与"失焦不暂停"的 web 答案

- 查证结论：Chrome **不在隐藏/后台标签页调用 rAF**（官方文档确认，该行为自 2011 年起就有）；后台页 timer 也会被节流（低至 1 次/秒，长时间后台更紧）
- **headless 下页面被视为可见（visibilityState=visible），rAF 正常触发**——Playwright 默认 headless 跑动画站并录出视频即依赖此行为。因此 **sweep 用 headless 即天然免疫"窗口失焦暂停"**，通用流程里 Unity 版要设 runInBackground 的坑在 web 下基本消失
- 三个残留陷阱（症状→解法）：
  - headed 调试时窗口被遮挡/最小化 → 游戏停摆。解法：调试期别遮挡；正式 sweep 无脑 headless
  - 游戏自己监听 `visibilitychange`/`blur` 自动暂停。症状：headless 一切正常，headed 一下 bot 就停。解法：源码搜 `visibilitychange`、`blur`，gate 掉（登记修改）
  - 环境异常时 rAF 被系统降频。解法：桥里暴露 `document.visibilityState` 与近 1s 的 rAF 计数，sweep 前跑一局自检帧率是否符合预期（如 60±20），别等 VLM 看到黑帧才发现

### 5.4 阻塞式 UI（web 天然好处理，但同类陷阱仍在）

- DOM overlay 弹窗是 querySelector 的天下：主菜单/升级选卡/结算屏，`document.querySelector('#btn-start')` 找到按钮**直接调其 handler**（走 §4.4 的 `handler_call`，记录 target）
- 照搬 Unity 版经验处理同类陷阱：
  - **动画中按钮不可点**：按钮等入场动画结束才 enable（`disabled`/`pointer-events:none`）→ 直调 handler 会绕过禁用态提前生效，**序列回放若改坐标点击就不等价**——`inject` 标注的意义就在这
  - **关闭异步完成**：面板关闭带 transition/animation（几百毫秒后才 remove）→ 注入后等面板真消失（轮询 querySelector 为 null，或 `subscribe('panel_closed')`）再走下一步；重复点击会打断关闭卡在半开状态
  - **看门狗**：面板开启超 N 秒（unscaled）未关 → 强制移除 + 恢复 speed + 触发游戏侧继续回调，保证永不死锁
- web 版"暂停"实现多样：停 rAF / ticker 暂停 / `paused` 标志——分析阶段把每个阻塞点的"暂停机制 + 恢复入口"记进分析报告，bot 逐个处理（对应 Unity 版 timeScale=0 清单）；终局检测必须先于"暂停就 return"的早退执行

## 6. 视频录制

按优先级选型，第一优先与批量 sweep 的 context 模型天然对齐：

### ① Playwright recordVideo（首选）

```js
const context = await browser.newContext({
  viewport: { width: 1280, height: 720 },
  recordVideo: { dir: 'out/level-1/round-3/videos/', size: { width: 1280, height: 720 } },
});
const page = await context.newPage();
const video = page.video();          // context 存活时拿句柄
// …跑完整局…
await context.close();               // 视频在 context close 时落盘，必须 await
const webm = await video.path();     // 自动生成文件名的 .webm；也可 video.saveAs(path)
```

- 官方文档确认的事实：**每 page 一段视频**（一局一个 page 即一局一段）；格式 **webm**；`size` 不填则取 viewport 缩到 800x800 内（viewport 未配则 800x450）；**视频只在 page/context 关闭后可用**
- webm → mp4：`ffmpeg -i in.webm -c:v libx264 -pix_fmt yuv420p -crf 23 out.mp4`——VLM 与通用流程要求的 mp4 统一走这一步
- 一局一个 context → 一局一个 webm，路径写进 per-run 目录与汇总报告，与 sweep 产物组织天然匹配

### ② CDP Page.startScreencast（需要逐帧控制时）

- 参数（已核对）：`format: "jpeg"|"png"`、`quality: 0-100`（仅 jpeg）、`maxWidth` / `maxHeight`、`everyNthFrame`
- 帧经 `Page.screencastFrame` 事件送达（`data` 为 base64 + `sessionId`）；**收到必须回 `Page.screencastFrameAck({sessionId})`，否则帧流停止**
- 帧率不保证：按合成器节奏出帧，低负载页面可能远低于预期，社区已知低帧率问题——别假设 60fps；拿到的帧 pipe 给 ffmpeg（`-f image2pipe`）拼视频
- 适用：不想留全程视频、只要低帧率抽帧诊断，或需要自定义管线时。Playwright 里用 `context.newCDPSession(page)` 建会话

### ③ canvas.captureStream() + MediaRecorder（游戏内自主录制，路线 A）

```js
const stream = canvas.captureStream(30);
const mime = MediaRecorder.isTypeSupported('video/webm;codecs=vp9')
  ? 'video/webm;codecs=vp9' : 'video/webm;codecs=vp8';
const rec = new MediaRecorder(stream, { mimeType: mime });   // 产出 webm → 同①转 mp4
rec.ondataavailable = e => chunks.push(e.data);
```

- 页面内录制，产物经桥回传（chunk → base64 上报，Node 落盘）；不依赖 Playwright
- 陷阱：录制期间别改 canvas 尺寸（尺寸变化中断/污染流）；rAF 停则流无新帧（headless 无此问题）；别指望浏览器直出 mp4——版本相关，统一 ffmpeg 转

### headless 渲染完整性与黑帧排查

- 查证结论：headless Chromium 无 GPU 时 **WebGL 走 SwiftShader 软件渲染**（Chromium 官方定位：GPU-less 环境的软件 WebGL fallback），画面功能完整、VLM 可用，但 CPU 渲染较慢
- 官方已**弃用自动 SwiftShader fallback**：较新 Chrome 在拿不到 GPU 时 WebGL 上下文可能**直接创建失败**（画布全黑）而不是悄悄软渲——此时显式加 `--use-gl=angle --use-angle=swiftshader-webgl --enable-unsafe-swiftshader`，不同大版本行为有差异，以 `chrome://gpu` 为准
- **别盲加 `--disable-gpu`**：它会强制回 SwiftShader 软渲染，有 GPU 也用不上
- 黑帧排查顺序（症状→解法）：
  1. 视频全黑且 console 无异常 → WebGL 上下文创建失败 → 查 `chrome://gpu`/页内探针确认 WebGL 状态，按上条加开关；或 headed 对照验证
  2. 片头黑、随后正常 → 录进了加载/黑屏阶段 → ffmpeg `-ss` 截掉片头（recordVideo 是 context 全程录）
  3. CDP screencast 只收到首帧 → 忘了 `screencastFrameAck` → 补 ack
  4. 偶发黑帧/卡顿 → 软渲染过慢或并发过高 → 降 viewport、降 recordVideo `size`、减并发
  5. 兜底探针：桥的 readState 里加 canvas 像素抽样（每秒 readPixels 一次记进 log），sweep 后统计非黑帧占比

## 7. 构建、启动与无人值守运行

### 构建与本地承载

- 有构建链：`npm run build` → dist/（vite/webpack/esbuild 同理）；纯静态 JS 直接用文件
- 静态服务器承载：`npx serve dist -l 8080` 或 `python3 -m http.server 8080`；**禁止用 file:// 打开**——ESM 加载/CORS/存储全废
- 顺手确认 sourcemap 是否产出（§3/§8 用得上）

### sweep 执行单位：一局 = 一个 browser context

- Node 脚本（playwright）并发开 N 个 context（可共用一个 browser 实例，context 数过百时分摊到多实例防内存压力）：
  - 每局流程：`newContext({ recordVideo, viewport })` → `newPage()` → goto → 等就绪 → 跑局 → 产物落盘 → `await context.close()`
- **存档隔离是 web 的最大结构性优势**：每个 context 的 localStorage / sessionStorage / IndexedDB / cookies / HTTP 缓存**全部天然隔离**（Playwright context = 独立 non-persistent 会话）。通用流程里 Unity 并发要手动隔离 persistentDataPath 的坑，web 下 concurrency>1 开箱即用——前提：**别用 launchPersistentContext 复用同一 profile**
- `context.close()` = "进程退出"：对局结束 5 秒内必须 close（视频此刻落盘）；"game over 后不退出"的兜底在 Node 侧做（墙钟超时强制 close，close 前先抢救已产出的 log/序列）

### 自动开局与 ready 标志

- goto `http://localhost:8080/?ag=1&...` → `page.waitForFunction(() => window.__autoGamer?.ready)` → 经桥程序化进入对局（主菜单点击走 §4.4）
- fresh run：`addInitScript` 里 `localStorage.clear()` + 视需要 `indexedDB.deleteDatabase(...)`（§8 持久化干扰）

### 参数传入（对应命令行/环境变量语义）

| 方式 | 做法 | 适用 |
|---|---|---|
| URL query | `?ag=1&speed=2&seed=123&level=3`，页面内 `new URLSearchParams(location.search)` | 最简、零构建耦合，路线 A/B 通吃——默认选择 |
| build-time 注入 | vite `define: { __AUTO_GAMER_CONFIG__: JSON.stringify(cfg) }` / webpack DefinePlugin | 需要构建期常量、随发布剥离 bot 时（§2） |
| addInitScript | `context.addInitScript(cfg => { window.__AUTO_GAMER_CONFIG__ = cfg; }, cfg)`——在任何页面脚本前执行 | 路线 B 免构建注入；顺带做 fresh-run 清理与随机播种 |

- **seed 落地**：web 游戏随机几乎全是 `Math.random()`——在 `addInitScript`（或桥初始化）用固定 seed 播种的 PRNG 覆写 `Math.random`，seed 记入 battle log——确定性清单第 1 条的 web 实现，等价性验证前提

### 失败检测（Node 侧全量挂监听）

- `page.on('pageerror')`：未捕获 JS 异常 → 记入该局 log 并计失败
- `page.on('console', m => m.type() === 'error')`：引擎红字（WebGL 创建失败/管线报错）
- `page.on('crash')`：renderer 崩溃（并发内存压力下会出现）→ 释放槽位重跑，别死等
- ready 标志 N 秒未现 / 墙钟超时未终局 → 强制 close 并标记该局失败
- 逐局结果原子写（临时文件 + rename）、按 round 目录判重断点续跑、每局完成输出进度——通用流程的 sweep 工程健壮性要求在 Node 侧照做

## 8. 已知坑与版本兼容

| 坑 | 症状 | 解法 |
|---|---|---|
| isTrusted（§4.1） | dispatchEvent 合成事件发出去了、游戏无反应 | 三选一：游戏管线直写 / 移除 isTrusted 检查 / CDP 注入（Playwright 输入） |
| pointer-only 输入 | 注入 MouseEvent 无反应，源码里只见 `pointerdown` 监听 | 合成事件不派生 pointer：走 CDP（自动派生 pointer 事件），或 dispatchEvent 改用 `PointerEvent` |
| rAF 节流（§5.3） | headed 窗口一遮挡游戏就停；个别环境 sweep 帧率异常低 | sweep 用 headless；桥暴露 visibilityState + rAF 计数自检；gate 掉游戏的 visibilitychange/blur 暂停 |
| 主逻辑在 Web Worker | DOM 注入怎么发都没用，画面在动但输入不进 | Worker 够不着 DOM。注入点改选**主线程→Worker 的输入通道**：找到 postMessage 的输入消息结构或 SharedArrayBuffer 输入区，桥直写（`inject:"game_state"`） |
| iframe 嵌套 | `page.evaluate` 找不到 game 全局对象 | 游戏在 iframe 里。Playwright 走 `page.frames()` 按 URL 选 frame 再 evaluate/操作；裸 CDP 需按 frame tree attach 对应 target |
| Unity WebGL 导出 | 产物是 `.wasm` + `Build/*.json` + loader 脚本 + `unityInstance` | **它就是 Unity**——转读 `engine-unity.md`，按 Unity 流程加内挂后导出 WebGL。WebGL 特有注意点见下 |
| sourcemap 缺失 | 运行时报错只有压缩栈，定位困难 | 分析以源码为准（skill 前提）；产物仅对照"打包后行为漂移"：按函数名/字符串常量反查；长期方案是构建配置开 sourcemap |
| storage 持久化干扰 fresh run | 局与局串档、开局不在起跑线、跑的还是旧构建 | `newContext()` 天然隔离是前提；用了 persistent profile 或复用 page 时必须清：addInitScript `localStorage.clear()` + `indexedDB.deleteDatabase`；Service Worker 缓存旧构建也在此时清（或构建产物带 hash 文件名规避） |

**Unity WebGL 特有注意点**（只列 web 差异，Unity 通用流程见 `engine-unity.md`）：
- 输入管线在 wasm 内部，JS 侧只看得到 canvas 与 `unityInstance`——注入入口 = canvas 上 Unity 注册的 DOM 事件监听（合成 dispatchEvent 能到达 Unity 的 JS 监听层），或从 CDP 打；C# 侧 Input 语义要回源码工程里对
- 逻辑直调走 `unityInstance.SendMessage('GameObject', 'Method', value)`（官方 WebGL JS API）——对应 Unity 版"逻辑层直调"，序列里照 §4 标 handler 类 `inject`
- 存档映射到 localStorage/IndexedDB → context 隔离照常可用；但 Unity WebGL 的 data files 体积大、首载慢，评估吞吐时把加载时长从对局时长里剔除
- 版本兼容总则：Playwright 各版本固定配套 Chromium 构建、不随系统 Chrome 漂移——锁定 Playwright 版本即锁定浏览器行为。自装 Chrome/手工 flags 的行为随大版本变化（如软件 WebGL 自动 fallback 弃用，§6），升级前先跑回归局


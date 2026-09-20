# 原生/小引擎 Playbook — 自动化游戏助手落地手册

> 覆盖 pygame、LÖVE（Love2D）、GameMaker、Electron、Bevy/Rust 与 SDL/C++ 自研引擎。这些框架没有大一统的"引擎级"设施（对象系统/输入系统/构建管线形态各异），落地时要逐项找到**主循环 / 输入事件队列 / 状态容器 / 帧计数器 / 存档路径**这五样东西。各框架注入事件必须映射 `references/sequence-record-replay.md` 的 schema（frame/t_unscaled/device/action/key/button/pos/vec/inject），`inject` 字段取值在各节给出，回放必须同路径。
>
> **结构说明**：本手册按框架分节组织（§1-§4 各框架、§5 通用方法论、§6 共性陷阱），不采用其它引擎手册的 §2-§8 主题式章节——SKILL.md 中"手册 §N"的引用在本手册中对应"各框架小节内的同主题内容"，§0 速查表仍然有效。

## 0. 框架速查表

| 框架 | 识别标志 | 事件注入 | 状态读取 | 速度控制 | 录像 | headless | 存档隔离 |
|------|---------|---------|---------|---------|------|---------|---------|
| pygame | `import pygame`、`display.set_mode`、主循环 `Clock.tick` | `pygame.event.post(Event(...))`（**不更新轮询状态**） | Python 对象直接读 | 主循环 dt 乘子 | `pygame.image.tobytes(screen,"RGB")` → ffmpeg pipe | `SDL_VIDEODRIVER=dummy`（像素仍可读） | 独立 cwd 启动 |
| LÖVE 11.x | `main.lua` 含 `love.draw/update`、`conf.lua` | `love.event.push("keypressed",...)`（**不更新轮询状态**） | Lua 表直接读 | 自定义 `love.run` dt 乘子 | `captureScreenshot(cb)` → `Data:getString()` → ffmpeg pipe | 无官方（Linux: Xvfb；macOS 只能带窗口） | `love.load(arg)` 解析参数 → `setIdentity("inst_N")` |
| GameMaker (GMS2) | `*.yyp`（GMS1: `.gmx/.gmz`） | `keyboard_key_press/release`（**更新轮询状态**）；鼠标无模拟 API | GML 全局/实例变量 | `game_set_speed(n, gamespeed_fps)` | `screen_save`/`surface_save` 每帧 PNG → ffmpeg | 无官方（Linux: Xvfb） | 存档在用户目录 → HOME 覆盖 / 参数化存档路径 |
| Electron | package.json 含 electron、`main.js` / `app.asar` | 主进程 `webContents.sendInputEvent()`（**需窗口聚焦**） | `executeJavaScript()` | 渲染进程按 engine-web.md | `beginFrameSubscription` / `desktopCapturer` | headless Chromium 可用 | `--user-data-dir` 每实例独立 |
| Bevy / Rust | Cargo.toml 含 bevy | 写 `KeyInput` 等事件（同时喂轮询资源） | ECS Query/Resource 直接读 | `Time<Virtual>::set_relative_speed` | 截图 API headless 失效 → OS 级录屏兜底 | `ScheduleRunnerPlugin::run_loop` + MinimalPlugins | 看游戏实现 |

## 1. pygame（Python）

**隔离**：bot 做成独立模块 `auto_gamer/`，被游戏主入口 import（改 1-2 行，删目录 + 删 import 即移除）。Python 同进程互访，无编译隔离问题。

**注入（InputInjector 设计）**：
- 注入 API：`pygame.event.post(pygame.event.Event(KEYDOWN, key=K_w, mod=0))` / `KEYUP` / `MOUSEBUTTONDOWN(button=...)` / `MOUSEBUTTONUP` / `MOUSEMOTION(pos=..., rel=...)`
- `inject` 字段取值建议：`event_post`
- **帧计数**：pygame 无 frameCount API——bot 在主循环 hook 处自维护 tick（每帧 +1）作为序列 `frame`；`t_unscaled` 用 `time.monotonic()`
- 所有注入走唯一注入口函数（注入 + 喂录制器，模式同 sequence-record-replay.md）

**🔴 大坑：事件注入不更新轮询状态（pygame 已知 issue #235）**
- 症状：bot 注入 KEYDOWN，游戏毫无反应；游戏代码用 `pygame.key.get_pressed()` / `pygame.mouse.get_pressed()` 轮询——`event.post` **不会**更新这些函数的返回值
- 解法（二选一）：
  - ① monkeypatch 轮询函数：bot 维护"注入键状态表"，包装 `pygame.key.get_pressed` / `pygame.mouse.get_pressed` 把注入状态 OR 进返回值（不改游戏代码）
  - ② 让游戏改事件驱动（读 KEYDOWN/KEYUP 维护状态）——侵入更大，非首选
- 判别：先 grep 游戏读输入的方式——`event.get()` 队列型可直接 post；`get_pressed()` 轮询型必须走解法 ①

**速度**：主循环 `Clock.tick(framerate)` 限帧——加速用 dt 乘子（游戏循环的 dt 通常来自 `clock.tick()` 返回值，bot 在主循环 hook 处把 dt 乘 speed 再传给 update）；每帧重新断言速度防游戏复位。

**录像**：每帧 `pygame.image.tobytes(screen, "RGB")`（pygame 2.1.3+，优先于 tostring）→ pipe 到 ffmpeg rawvideo（`-pixel_format rgb24`）。若用 `pygame.surfarray.array3d`，返回形状是 (width, height, 3)、x 在前——需 transpose 才符合行主序视频。采集必须与主循环同帧触发。

**headless**：`os.environ["SDL_VIDEODRIVER"] = "dummy"`（必须在 `pygame.init()` 之前设置）——`display.set_mode` 可用、surface 只存在内存、**像素仍可读**（软件 surface）、display flip 为 no-op。CI 无头测试是成熟模式。

**存档隔离**：pygame 游戏存档多为相对 cwd 的文件——sweep 给每个实例用独立 cwd 启动即可；多进程并发天然隔离。

**自动开局/退出**：同进程内 bot 直接调游戏的"开始关卡"函数；游戏结束后 bot 调 `pygame.quit()` + `sys.exit()`。

## 2. LÖVE / Love2D（Lua）

**识别**：`main.lua` 定义 `love.draw`/`love.update`；`conf.lua`；`.love` 打包文件。

**隔离**：bot 做成 require 模块（`require("auto_gamer")`），挂在 `love.load` 包装处（main.lua 改 1-2 行）；与游戏同进程互访 Lua 全局表。

**注入**（11.x 签名，全部官方核实）：
- `love.event.push("keypressed", key, scancode, isrepeat)` / `("keyreleased", key, scancode)`
- `("mousepressed", x, y, button, istouch, presses)`——button 是数字：1=主键 2=次键 3=中键（与序列 left/right/middle 映射）
- `("mousemoved", x, y, dx, dy, istouch)` / `("wheelmoved", x, y)`
- 旧版 0.9.x 的 keypressed 是 2 参签名——注入前先确认 LÖVE 版本
- `inject` 字段取值建议：`event_push`；帧计数 bot 在主循环 hook 自维护 tick；`t_unscaled` 用 `love.timer.getTime()`（真实时钟，不受速度影响）

**🔴 坑：isDown 轮询看不到 push 注入（与 pygame 同母题）**
- 症状：游戏用 `love.keyboard.isDown` / `isScancodeDown` 轮询，bot push 的 keypressed 无效（官方文档明确 isDown 与 keypressed 回调是两套机制）
- 解法：① 包装 `love.keyboard.isDown`——bot 维护注入键状态表 OR 进返回值；② 游戏改事件驱动

**速度**：自定义 `love.run`（11.x 官方支持），把 dt 乘 speed 因子再传 `love.update(dt)`。**自定义 love.run 必须保留 `love.event.pump()` 与事件 poll 循环**——否则 push 的事件不被派发、且 OS 认为进程挂起。每帧重新断言速度。

**失焦/暂停**：LÖVE 引擎本身不因失焦暂停，但游戏自己可能实现——检查 `love.focus(focus)` 回调里有没有暂停逻辑，无人值守必须处理。

**录像**：`love.graphics.captureScreenshot(callback)`——**异步**，当帧 `love.draw` 完成后回调传 ImageData；ImageData 是 Data 子类型，用 `Data:getString()` 取原始 RGBA 字节 → pipe 到 ffmpeg（`-pixel_format rgba`）。按 video-fps 间隔发起采集。

**headless**：无官方模式。Linux 用 Xvfb 虚拟显示；**macOS 没有 Xvfb**——只能带窗口跑（处理好失焦暂停即可）。需要视频的场合本来就是"有渲染"模式，与 Xvfb 兼容。

**存档隔离**：`love.filesystem.setIdentity(name)` 可运行时改写存档目录名（只能改名不能改位置）；`love.load(arg, unfilteredArg)` 拿得到命令行参数表——per-instance 方案：love.load 早期解析 bot 参数并 `setIdentity("save_inst_N")`，再首次文件访问。未找到官方 `--identity` CLI 旗标【备选：拷贝工程改 conf.lua 的 `t.identity`】。注意 `love.filesystem.isFused()` 影响 getSaveDirectory 位置（fused 在 Appdata 根，非 fused 在 Appdata/LOVE/ 下）。

## 3. GameMaker（GML）

**识别**：`*.yyp`（GMS2+）；`.gmx/.gmz`（GMS1）；`rooms/`、`objects/`、`sprites/` 资源目录。

**隔离**：bot 作为 persistent object（游戏开局事件创建、跨 room 存活），或做成 extension。不改游戏原始对象。

**注入**：
- 键盘：`keyboard_key_press(key)` / `keyboard_key_release(key)`——模拟按键，**会置位 `keyboard_check` 轮询状态直到对应 release 被调用**（与 pygame/LÖVE 相反：注入直接喂轮询）
- `inject` 字段取值建议：`keyboard_key_press` / `event_perform` / `window_mouse_set`
- 鼠标：**GML 没有任何模拟鼠标按下的 API**——`device_mouse_*` 全为只读。可用手段：
  - `window_mouse_set(x, y)` 设鼠标位置——仅桌面平台、**仅游戏聚焦时有效**、Ubuntu Wayland 下失效、macOS 坐标更新有事件延迟一帧的坑
  - 点击解法 ①：`event_perform(type, numb)` 在目标实例上直接触发其事件（如 `event_perform(ev_mouse, ev_left_button)`、`event_perform(ev_keypress, ord("W"))`）；注意 `ev_draw` 不能在非 draw 阶段强制触发
  - 点击解法 ②：拥有源码 + git——把游戏的 `mouse_check_button(...)` 调用点全局替换为包装脚本（GML 无运行时 monkeypatch，只能改调用点，属可接受的最小侵入）

**坑与注意**：
- `keyboard_key_press` 不配对 `keyboard_key_release` → 键永久按死（症状：bot 移动停不下来）。对"按一下"语义（`keyboard_check_pressed` 类检测），press + release 要在同帧配对调用
- **Windows 下反复调用 `keyboard_key_press` 会向系统层泄露按键事件，影响其它聚焦窗口**——并发 sweep 多实例时危险，必须在同实例内闭合 press/release，且不要依赖系统焦点
- 帧计数：bot 在 persistent object 的 Step 事件里自维护 tick；`t_unscaled` 用 `current_time`（毫秒真实时钟）

**速度**：`game_set_speed(n, gamespeed_fps)`（或 `game_set_speed(33333, gamespeed_microseconds)`）；旧 `room_speed` 是 GMS1 遗产。`display_set_timing_method(tm_systemtiming)` 解除帧率上限让加速真生效。每帧重新断言。

**录像**：`screen_save(fname)` 存整窗渲染 / `surface_save` 存 surface——每帧存 PNG，事后 ffmpeg 合成（`ffmpeg -framerate N -i frame_%05d.png -c:v libx264 -pix_fmt yuv420p out.mp4`）。GameMaker 渲染走 application surface，采集事件挂在 Draw 末段、与主循环同帧触发。

**headless**：无官方模式。Linux: Xvfb；macOS 带窗口。

**构建（Igor CLI）**：官方命令行构建，需要 IDE 生成的 Access Key：
```
Igor.exe /uf=[user_folder] /rp=[runtime_path] /project=[project.yyp] /cache=[cache_dir] /temp=[temp_dir] /of=[output] /tf=[target_file] /device=[device_name] -- Windows PackageZip
```
- `/runtime=YYC` 为 YYC 编译目标；`-- Windows Run` 为直接运行；Runner 支持 `-game <filename>` 加载游戏文件
- Igor 可执行文件路径因平台/安装方式而异【待验证，排查方向：Windows 常见在 `C:\ProgramData\GameMakerStudio2\Igor.exe`，macOS 在 app bundle内】；社区封装（Rubber、bscotch/igor-build）可参考

**存档隔离**：默认存档/工作目录——Windows `%LOCALAPPDATA%/[Game Name]/`、macOS `~/Library/Application Support/[Game Name]/`。per-instance 方案：① 存档路径跟随 HOME 的，用环境变量覆盖 HOME 启动每实例；② 在游戏代码里找到存档路径常量，参数化它。

## 4. Electron / 桌面 JS 壳

**识别**：package.json 依赖含 electron；主进程入口 `main.js`；打包产物 `app.asar`。

**与 engine-web.md 的关系**：渲染进程里跑的还是 web 游戏——渲染进程内的一切（状态读取、调试桥、游戏输入管线、DOM 弹窗）按 `engine-web.md` 处理；本节只补 **Electron 主进程独有** 的能力。

**注入（主进程）**：`webContents.sendInputEvent(event)`——浏览器级真实输入（isTrusted=true）：
- type：`mouseDown/mouseUp/mouseMove/mouseWheel/keyDown/keyUp/char`（另有 mouseEnter/mouseLeave/contextMenu）
- 键盘：`keyCode` 为单个 UTF-8 字符或 Accelerator 键名（enter/backspace/escape…）；modifiers 数组（shift/control/alt/meta/isKeypad…）
- 鼠标：`x`,`y` 必填；`button` 取 left/middle/right；可带 globalX/globalY/movementX/movementY/clickCount
- `inject` 字段取值建议：`send_input_event`
- **🔴 大坑：BrowserWindow 必须聚焦，`sendInputEvent` 才生效**——并发多实例时"每实例一个聚焦窗口"不成立。对策：并发 sweep 优先走渲染进程内合成事件注入（engine-web.md 路线 A，不受焦点限制）；`sendInputEvent` 留给单实例、等价性验证等需要"真实输入"的场合

**状态读取**：`webContents.executeJavaScript(code)` 从主进程读渲染进程状态（engine-web.md 路线 B 的加强版）。

**录像**：
- `webContents.beginFrameSubscription([onlyDirty,] callback)`——逐帧拿 NativeImage，取原始像素 pipe 到 ffmpeg（最贴合通用"帧采集 + ffmpeg pipe"模式）；`endFrameSubscription` 结束
- `desktopCapturer.getSources({types:['screen','window'], thumbnailSize})` + 渲染进程 `getUserMedia`（chromeMediaSource: 'desktop' + chromeMediaSourceId）取 MediaStream 录制

**并发隔离**：`--user-data-dir` 每实例独立（`requestSingleInstanceLock` 按 userData 路径判定，不同 dir 即可多开）；渲染进程存档（localStorage/IndexedDB）随 userData 目录隔离。

## 5. 自研/未知引擎通用方法论

当游戏不属于任何已知引擎时，落地前逐项定位（读代码 + 跑起来观察）：

1. **主循环**：找 while/for 主循环或 update/render 回调（`while True`、`love.run`、SDL 的 `while(running)`、ECS schedule）——bot 的决策、速度断言、采集都要挂进主循环
2. **输入事件队列或轮询 API**：找 post/push/dispatch/Inject 类函数；没有队列时找轮询函数（get_pressed / isDown / check 类）——**先判别"事件式还是轮询式"再定注入方案**（见 §6 母题）
3. **状态容器**：全局单例、ECS registry、场景/实体列表——观察器从这里读
4. **帧计数器**：引擎没有就 bot 在主循环 hook 自维护 tick
5. **存档路径**：找 save/write/serialize 相关代码；隔离方法见 §6
6. **速度/时间步进变量**：找 dt 计算处（clock.tick / getDelta / 固定步长常量），插乘子；每帧重新断言
7. **录像兜底**：引擎无截帧 API 时用 OS 级录屏——Linux `ffmpeg -f x11grab`、macOS `-f avfoundation`、Windows `-f gdigrab`（受窗口遮挡影响，虚拟显示/独占桌面可规避）。Bevy 注意：headless 下截图 API 回调不触发（官方 issue #11493）——要录像就带渲染表面

**按语言的注入形态**：
- Python：运行时 monkeypatch（最灵活，pygame 同款）
- Lua：环境 hook / 函数包装（LÖVE 同款）
- C/C++：编译期 hook、链接期符号替换、全局函数指针覆写
- Rust：trait 对象替换、函数指针替换（静态分发需改调用点）
- **Bevy/Rust 专项**：直接写 `KeyInput` 等事件——内部系统消费事件同时更新 `ButtonInput<KeyCode>` 轮询资源，两种读法都喂到；headless 用 `ScheduleRunnerPlugin::run_loop(Duration::from_secs_f64(1.0/60.0))` + MinimalPlugins（禁用默认 bevy_window 特性）；变速 `Time<Virtual>::set_relative_speed(f32)`

## 6. 跨框架共性陷阱

**母题：轮询式 vs 事件式输入（注入前必判别）**

| 游戏读输入的方式 | 注入手段 | 后果/陷阱 |
|--------------|---------|-----------|
| 事件驱动（`event.get()` / `love.keypressed` 回调 / DOM 事件） | 往事件队列 post/push | 直接生效 |
| 轮询式（`get_pressed` / `isDown`）—— pygame、LÖVE | 事件注入**不更新**轮询状态 | bot 注入无反应——必须 monkeypatch 轮询函数或改调用点 |
| 轮询式（`keyboard_check`）—— GameMaker | `keyboard_key_press` **会更新**轮询状态 | 例外——必须配对 release，否则按键永久生效 |

判别方法：grep 游戏代码读输入的 API；不确定时两种路径都试并观察游戏反应。**很多"bot 策略没反应"的问题根因在此，先查控制层再调策略**（与 log-only-diagnosis 的"症状→根因层"对照表一致）。

**其余共性**：
- **帧索引约定**：小引擎普遍没有帧计数器——bot 在主循环 hook 自维护 tick 作为序列 `frame`，录制/回放同源；没有主循环 hook 就做不了逐帧对齐，优先解决挂载点
- **录像与主循环同帧触发**：采集点不在主循环内会丢帧/重帧（rAF/事件回调型引擎尤其）
- **headless 兜底链**：引擎内 dummy 驱动（pygame）→ Xvfb 虚拟显示（Linux）→ 带窗口后台跑（macOS 无 Xvfb）。注意"失焦暂停"多为游戏自己实现（LÖVE 的 love.focus、GM 的窗口焦点语义、Electron sendInputEvent 的聚焦要求），逐个检查处理
- **多实例并发隔离清单**：独立 cwd（pygame 存档）/ 独立存档目录（setIdentity / HOME 覆盖 / --user-data-dir）/ 独立端口（游戏开本地服务时）/ 独立临时文件目录
- **速度看门狗**：小引擎没有引擎级"UI 复位速度"陷阱，但游戏自己的暂停菜单/弹窗代码同样可能重置速度倍率——每帧重新断言的原则不变


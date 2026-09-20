# Godot Playbook — 自动化游戏助手落地手册

> 适用 Godot 4.x，兼顾 3.x 差异（改名对照见 §8）。SKILL.md 通用流程在 Godot 工程里的具体落地方式，章节号与通用流程引用一一对应。

## 0. 通用概念 → Godot API 对照表

| 通用概念 | Godot API / 做法 | 注意事项 |
|----------|------------------|----------|
| 帧索引（序列时间基准） | `Engine.get_process_frames()`（G3: `get_idle_frames()`） | **禁止用 `get_frames_drawn()`**——headless / `--disable-render-loop` 下恒为 0 |
| 不受速度影响的时钟 | `Time.get_ticks_msec()`（G3.0-3.4: `OS.get_ticks_msec()`） | 一切计时基准；`Engine.time_scale` 只缩放 delta 不影响它 |
| 游戏速度控制 | `Engine.time_scale`；启动参数 `--time-scale <scale>` | 不影响音频速度（音频另用 `AudioServer.playback_speed_scale`）；每帧重新断言 |
| 节点/场景遍历 | `get_tree().get_nodes_in_group()`、`find_child()/find_children()`、`%UniqueName` | 全树遍历慢——缓存 + 0.3-0.5s 刷新 |
| 输入注入 | `Input.action_press(action, strength)`（轮询型）/ `Input.parse_input_event(event)`（事件型） | 按"游戏怎么读输入"选层；**headless 下事件注入失效**（GitHub #73557） |
| 视频帧采集 | ① Movie Maker（`--write-movie`）② SubViewport + `get_texture().get_image()` → ffmpeg pipe | `--headless` 下两者都不可用（dummy 渲染） |
| 用户存档目录 | `user://`（各平台解析见 §7） | 并发实例用环境变量隔离（XDG_DATA_HOME / HOME / APPDATA） |
| 失焦不暂停 | Godot 桌面默认失焦不暂停；游戏自己实现的暂停要逐个压制 | grep 游戏代码 `NOTIFICATION_APPLICATION_FOCUS_OUT` |
| 无头运行 | `--headless`（= dummy 显示驱动 + Dummy 音频驱动） | 场景树/脚本/物理/轮询输入照常跑——优于多数引擎的"无图形=跑不了逻辑" |
| 命令行/环境变量传参 | `OS.get_cmdline_user_args()`（`--` 之后的参数，G4 独有）+ `OS.get_environment()` | **未知参数静默忽略、无任何警告**——参数没生效先查拼写 |
| 命令行导出 | `--headless --import` → `--export-release "预设名" 输出路径` | 预设须匹配 export_presets.cfg；目标目录必须已存在 |

## 1. 技术栈识别

- 工程标志：`project.godot`、`*.tscn` / `*.escn` 场景、`*.gd` / `*.cs` 脚本、`res://` 资源协议
- 版本判定：G4 的 `project.godot` 含 `config_version=5` 且 `config/features=PackedStringArray("4.3")`（首个元素即版本号）；G3 为 `config_version=4`
- GDScript vs C# 工程：决定 bot 实现语言（两者都经 autoload 隔离，见 §2）；C# 导出有额外坑（§7 / §8）

## 2. Bot 代码隔离与组织

- 推荐姿势：bot 放 `res://auto_gamer/` 目录 + 在 `project.godot` 的 `[autoload]` 段注册单例：

```ini
[autoload]

AutoGamer="*res://auto_gamer/auto_gamer.gd"
```

  `*` 前缀 = 单例启用。**这是唯一侵入点**——删除目录 + 删除该行即完全移除，符合"最小侵入、删除即移除"
- 不改游戏脚本；bot 与游戏的交互走 group / 信号 / 节点查找（见 §3）
- C# 工程：autoload 也可以是 C# 脚本或 .tscn；从 C# 侧访问 GDScript 单例用 `GetNode("/root/AutoGamer")` / `Engine.GetSingleton`
- GDExtension 仅在确需原生库时引入（ffmpeg pipe 用 OS API 已够，见 §6），不作为默认选项

## 3. 游戏状态读取（观察器）

- **group 优先**：bot 启动时给关键实体 `add_to_group()`（游戏代码没加 group 时按节点名/类名扫描一次建组），之后 `get_tree().get_nodes_in_group("enemies")` 取列表
- 查找：`get_node()` / `get_node_or_null()` / `find_child(pattern, recursive, owned)` / `find_children()` / `%UniqueName`（场景内唯一名访问）
- GDScript 动态访问：`obj.get("hp")`——脚本的成员变量即属性，任意脚本成员可读；`get_property_list()` 列出可用属性。**C# 对象经 `Get()` 只能看到 `[Export]` 成员**——观察 C# 游戏时受限成员要走源码内的访问器
- 配置数据：`.tres` 资源用 `load()` 直接读
- 性能：全树遍历慢——观察器缓存节点引用 + 0.3-0.5s 定时刷新；游戏有信号（如 `hp_changed`）时优先订阅信号而非轮询

## 4. 输入注入与唯一注入口

### 两层注入 API（先判别游戏怎么读输入）

| 层 | API | 适用 | 陷阱 |
|----|-----|------|------|
| 动作层 | `Input.action_press(action, strength)` / `Input.action_release(action)` | 游戏用 `is_action_pressed()` / `is_action_just_pressed()` / `get_action_strength()` 轮询 | **官方明确"不会触发任何 `_input()` 回调"**——只喂轮询；strength 0-1 供非布尔 action |
| 事件层 | `Input.parse_input_event(InputEventKey / InputEventMouseButton / InputEventMouseMotion)` | 游戏用 `_input` / `_unhandled_input` / `_gui_input` 事件回调，或需要 UI 命中检测 | 事件先进缓冲（`use_accumulated_input` 默认开），**下一帧 flush 才派发**；`Input.flush_buffered_events()` 可立即派发（当帧生效，但同步执行游戏回调） |

判别方法：grep 游戏读输入的 API——轮询型走动作层、事件型走事件层。移动轴注入优先用 action + strength（配合 `Input.get_vector` 的游戏最顺），其次合成 `InputEventMouseMotion`。

**🔴 headless 大坑（GitHub #73557，至今 Open）**：`--headless` 模式下 `parse_input_event` 注入的事件**完全不派发**——无头 sweep 只能驱动"轮询型"游戏；事件回调型 / 需要 UI 点击的游戏必须有真实渲染环境（Linux 用 Xvfb，见 §6/§7）。

### 坐标与事件细节

- `Input.warp_mouse(position)` 移动光标（G3: `warp_mouse_position`）
- **坐标系**：Godot 屏幕/窗口坐标 **Y 向下、左上原点**——与 Unity 序列约定的左下原点相反。序列 `pos` 字段统一约定后记入 meta（`"origin": "top-left"`），回放按同一约定还原
- `InputEventKey` 注入建议**同时设 `keycode` + `physical_keycode`**——游戏读法不一，两个都给最稳
- `InputEventMouseButton`：`button_index`、`pressed`、`position`；`InputEventMouseMotion`：`position` + `relative`

### 唯一注入口设计（对齐 sequence schema）

所有注入走 bot 单例的统一入口，注入 + 喂录制器，字段对齐 `references/sequence-record-replay.md`：

- `frame` = `Engine.get_process_frames()`
- `t_unscaled` = `Time.get_ticks_msec() / 1000.0`
- `inject` 取值建议：`parse_input_event`（真实事件路径）/ `action_press`（动作层路径）/ `warp_mouse`（光标移动）/ `direct_call`（UI 直调，见下）
- **UI 点击两路径**：真实事件（warp + press/release，走引擎 UI 命中检测 → `inject: parse_input_event`）vs 直调（`button.pressed.emit()` 或调用游戏处理函数 → `inject: direct_call`）——回放必须同路径；直调路径记录按钮节点路径而非坐标，回放时路径不存在立即报告分歧

## 5. 时间、速度与暂停

### 速度与计时

- `Engine.time_scale` 缩放 delta；启动参数 `--time-scale` 可预置。**它不影响音频播放速度**——依赖音频节奏的游戏要另设 `AudioServer.playback_speed_scale`
- **每帧重新断言** `Engine.time_scale`（游戏 UI 关面板可能复位速度——同 Unity timeScale 陷阱）与 `Engine.max_fps`（游戏失焦处理可能压成 1）
- 计时一律用 `Time.get_ticks_msec()`；帧率上限 `Engine.max_fps`（G3: `target_fps`）

### 🔴 暂停语义与阻塞式弹窗（Godot 版"处理先于暂停早退"）

- `get_tree().paused = true` 时，可暂停节点的 `_process` / `_physics_process` / `_input` / `_input_event` 全部停调——**但信号仍会触发**
- **bot 节点必须设 `process_mode = Node.PROCESS_MODE_ALWAYS`**（G3: `pause_mode = PAUSE_MODE_PROCESS`），才能在暂停期继续运行、检测并关闭阻塞式弹窗——缺了它，弹窗把树一暂停 bot 就死。这是 Godot 侧最核心的一条
- **额外坑（论坛实测）**：只把 bot 节点设 ALWAYS 可能仍收不到 `_input`——需要把场景树的**根节点也设为 ALWAYS**
- 暂停 UI 面板本身惯用 `PROCESS_MODE_WHEN_PAUSED`（只在暂停期运行）——识别游戏弹窗时可按此推断哪些面板走暂停路线
- 弹窗处理原则与 SKILL.md 3.2b 一致：优先逻辑层直调（调 manager 应用选择 + 关面板 + 触发其"关闭"信号），加看门狗超时强关

### 失焦行为

- Godot 桌面**默认失焦不暂停**——没有 Unity `runInBackground` 式的引擎开关问题
- 但游戏可能自己实现：grep 游戏代码 `NOTIFICATION_APPLICATION_FOCUS_OUT`（G3: `NOTIFICATION_WM_FOCUS_OUT`）——常见实现是置 `get_tree().paused = true` 并把 `Engine.max_fps` 压到 1。bot 每帧重断言两个值即可压制
- 检测手段：`Window.has_focus()` / `focus_exited()` 信号（原生窗口）

## 6. 视频录制

按优先级三条路线：

### ① Movie Maker 模式（最省事）

```bash
godot --path <project> --write-movie out.avi --fixed-fps 30 --resolution 1280x720
```

- `--write-movie` / `--fixed-fps` **在导出后的游戏里同样可用**；`--fixed-fps` 让引擎按恒定 delta 非实时仿真——帧率波动不影响帧索引（对序列等价性友好）
- **录制范围 = 进程整个生命周期**（引擎启动到退出），不能中途开关——录完用 ffmpeg 裁出对局段
- 输出格式：AVI（MJPEG，近实时，**单文件 4GB 上限**——超长对局分段录）；PNG 序列 + WAV（无损但极慢，1 分钟视频可超 1 小时）；OGV 写入器（较新、无 4GB 限制但格式支持差，引入版本【待验证：查官方 release notes】）
- 转制：`ffmpeg -i out.avi -c:v libx264 -pix_fmt yuv420p -crf 23 recording.mp4`
- **headless 下不可用**（dummy 渲染驱动无帧可录）——需要真实渲染：真窗口，或 Linux Xvfb 虚拟显示
- 可用项目设置预置：`movie_writer/movie_file` / `movie_writer/fps` / `movie_writer/mjpeg_quality`

### ② SubViewport + ffmpeg pipe（最贴合通用"帧采集 + pipe"模式）

- bot 内创建 `SubViewport` 渲染游戏画面；每帧采集：
  `await RenderingServer.frame_post_draw` → `viewport.get_texture().get_image()`（HDR 格式先 `convert(Image.FORMAT_RGBA8)`）→ `get_data()` 得 RGBA 字节 → pipe 到 ffmpeg rawvideo（`-pixel_format rgba`）
- **pipe API**：`OS.execute_with_pipe(path, args, blocking)` → `{"stdio": FileAccess（stdin 写 + stdout 读）, "stderr": FileAccess, "pid": int}`（桌面平台实现）
  **已知 bug #102340**：pipe 返回的 FileAccess 很多函数不可用（seek、可读性判断等）——只往 stdin `store_buffer()` + `flush()`；ffmpeg 加 `-loglevel error` 且避免大量 stdout/stderr 输出，防止子进程把管道写满死锁
- 备选：`OS.create_process()`（非阻塞、无 stdin 流）或每帧 `Image.save_png()` 落盘、事后 ffmpeg 合成（无死锁风险但 IO 大）

### ③ 无头不可采帧 + 推荐分层

- `--headless` = dummy 渲染驱动——**截帧必空/黑**，结论同 Unity `-nographics`
- 分层推荐（对应 SKILL.md 3.2c）：
  - 纯 log 迭代 / 吞吐 sweep → `--headless`（Godot headless 仍跑完整场景树/脚本/物理/轮询输入——无头可用性优于多数引擎；但注意 §4 的事件注入失效与音频 Dummy 驱动）
  - 需要视频 → Movie Maker 或 SubViewport（带真窗口或 Xvfb）
  - 排查 bot 行为 → 带窗口运行
- headless 的音频驱动是 Dummy——依赖音频时序的逻辑（用 `AudioStreamPlayer` 计时的冷却/节奏）会异常

## 7. 构建、启动与无人值守运行

### 命令行导出

```bash
godot --headless --import --path <project>            # 预导入资源（隐含 --editor --quit），首次必跑
godot --headless --export-release "预设名" <输出绝对路径> --path <project>
```

- 预设名必须与 `export_presets.cfg` 中一致；输出路径含文件名，**目标目录必须已存在**；`--export-release` 隐含 `--import`
- 其他目标：`--export-debug` / `--export-pack`
- **CLI 坑：未知命令行参数完全无效且无任何警告**——参数没生效先怀疑拼写错误或"该构建类型不支持此参数"
- 实用参数：`--quit-after N`（N 次迭代后退出——定长采集/最小验证好用）、`--log-file`、`--max-fps 0`、`--disable-vsync`

### 运行参数传入与退出

- bot 参数走 `--` 分隔：`./game -- --auto true --speed 2 --seed 42`，游戏内用 `OS.get_cmdline_user_args()` 读取（**G4 独有**；G3 只有 `OS.get_cmdline_args()`，需自行跳过引擎参数）
- 环境变量读 `OS.get_environment()`；对局结束 `get_tree().quit()` 退出进程（Movie Writer 在 quit 时收尾写出文件）

### per-instance 存档隔离（并发 sweep 必做）

`user://` 默认解析位置：

| 平台 | 默认路径 |
|------|---------|
| Windows | `%APPDATA%\Godot\app_userdata\<项目名>` |
| macOS | `~/Library/Application Support/Godot/app_userdata/<项目名>` |
| Linux | `~/.local/share/godot/app_userdata/<项目名>` |

- 项目设置 `application/config/use_custom_user_dir`（+ `custom_user_dir_name`）可去掉 `Godot/app_userdata` 前缀（仅桌面平台生效）
- **未发现 `--custom-user-dir` 命令行参数【待验证：以官方命令行参考为准】** → **主方案：每实例环境变量隔离**（Godot 遵循 XDG/平台路径规范）：Linux 覆盖 `XDG_DATA_HOME`、macOS 覆盖 `HOME`、Windows 覆盖 `APPDATA`——sweep 给每个游戏进程注入不同的值即得独立 user://
- 运行时查询实际路径：`OS.get_user_data_dir()`

### 工程锁与并发

- Godot **没有** Unity 式工程锁文件——两个编辑器/headless 导出同时开同一工程不直接报错，但会争用 `.godot/` 导入缓存与 `uid_cache.bin`（UID 冲突有实证）。**纪律：headless 导出构建时关掉编辑器；构建串行**
- **导出后的游戏实例之间零锁**、完全独立——并发 sweep 唯一要处理的是 user:// 隔离（见上）

### C# 工程导出（专项坑）

- **headless 导出不会像编辑器那样自动编译 C#**——先 `dotnet build -c ExportRelease`，或 `godot --headless --build-solutions --quit`（该参数隐含 --editor）
- **GitHub #110101：dotnet build server 可让 headless 导出永久挂死**——导出前执行 `dotnet build-server shutdown`（或杀掉 dotnet 进程）
- **headless 导出可能日志报错但退出码仍是 0**——以日志为准，不看退出码（与 SKILL.md 步骤4 的构建判断原则一致）

## 8. 已知坑与版本兼容

### Godot 3 → 4 改名对照（官方 renames_map_3_to_4.cpp）

| 3.x | 4.x |
|-----|-----|
| `Engine.get_idle_frames()` | `Engine.get_process_frames()` |
| `pause_mode` / `PAUSE_MODE_PROCESS` / `PAUSE_MODE_STOP` | `process_mode` / `PROCESS_MODE_ALWAYS` / `PROCESS_MODE_PAUSABLE` |
| `find_node()` | `find_child()` |
| `scancode` / `physical_scancode` | `keycode` / `physical_keycode` |
| `BUTTON_LEFT` 等鼠标常量 | `MOUSE_BUTTON_LEFT` 等 |
| `NOTIFICATION_WM_FOCUS_OUT` | `NOTIFICATION_APPLICATION_FOCUS_OUT` |
| `OS.get_ticks_msec()` | `Time.get_ticks_msec()` |
| `Input.warp_mouse_position()` | `Input.warp_mouse()` |
| `iterations_per_second`（项目设置） | `physics_ticks_per_second` |
| `Viewport`（作为渲染目标） | `SubViewport` |
| `Engine.target_fps` | `Engine.max_fps` |

### 其它版本/行为坑

- **G4 启动时自动调用 `randomize()`**——确定性运行 / 序列等价性验证必须在 `_ready()` 里显式设种子（`seed(<n>)`），否则每局 RNG 不同、等价性必断
- `call_group` 在 G4 **默认立即执行**（G3 默认 deferred）——跨大版本迁移的 bot 代码注意时序差异
- `Engine.get_frames_drawn()` 在 headless / `--disable-render-loop` 下恒 0——禁止当帧索引用（§0 已列，最常见误用）
- C# 的移动/Web 导出 4.2 起才实验性；旧 Mono 项目按官方文档迁移到 .NET
- macOS 后台/遮挡窗口可能被系统限流（App Nap）——darwin 上并发 sweep 若墙钟异常，用 `caffeinate` 包装启动【待验证：对比 log 帧时间戳确认是否中招】


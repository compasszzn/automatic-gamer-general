# Unity Playbook — 自动化游戏助手落地手册

> 适用于 Unity 及 Tuanjie（团结引擎，API 兼容 Unity，编辑器进程名为 Tuanjie）。SKILL.md 的通用流程在 Unity 工程里的具体落地方式，章节号与通用流程的引用一一对应。

## 0. 通用概念 → Unity API 对照表

| 通用概念 | Unity API / 做法 | 注意事项 |
|----------|------------------|----------|
| 帧索引（序列时间基准） | `Time.frameCount` | 回放唯一时间基准，绝不用墙钟毫秒 |
| 不受速度影响的时钟 | `Time.unscaledTime` | speed 缩放下 `Time.time` 失真，一切计时用它 |
| 游戏速度控制 | `Time.timeScale` | 被 UI 面板复位的重灾区，每帧重新断言（见 §5） |
| 对象遍历 | `FindObjectsOfType<T>()`（全版本兼容；`FindObjectsByType` 仅 2022.2+） | 每帧调用严重拖慢性能——必须缓存 + 0.3-0.5s 定时刷新；结果要按 `gameObject.activeSelf` 过滤 |
| 输入注入 | 新 Input System：`InputSystem.QueueStateEvent` + `Mouse.current.WarpCursorPosition`；旧 Input API：模拟按键/直调游戏方法 | 两种输入系统**不混用**，运行时错误（见 §4） |
| 视频帧采集 | 专用 Camera → RenderTexture → `ReadPixels` → RGBA pipe 到 ffmpeg | **绝不劫持 `Camera.main.targetTexture`**，会画面冻结（见 §6） |
| 存档目录 | `Application.persistentDataPath` | 并发实例必须各自隔离，否则存档互相覆盖 |
| 失焦不暂停 | `Application.runInBackground = true` + ProjectSettings `runInBackground: 1` | 运行时与工程设置都要设 |
| 无头运行 | `-batchmode`；纯 log 再加 `-nographics` | `-nographics` 下视频为黑帧（见 §6/§7） |
| 命令行参数 / 环境变量 | `System.Environment.GetCommandLineArgs()` / `GetEnvironmentVariable()` | bot 的参数解析走这里 |
| 命令行构建 | `Unity -batchmode -quit -executeMethod <方法>` + Editor 脚本 | scripting define 陷阱见 §7 |
| 反射访问 | `System.Reflection`（访问 protected 的 manager/字段/方法） | 常见且可接受——最小侵入复用游戏逻辑 |

## 1. 技术栈识别

- 工程标志：`Assets/`、`ProjectSettings/`、`Library/`、`ProjectVersion.txt`（可确认 Unity 版本）、`*.asmdef`
- Tuanjie：结构与 Unity 相同，`ProjectVersion.txt` 标注 Tuanjie 版本，编辑器/构建进程名为 `Tuanjie`
- 源码位置：`Assets/**/*.cs`；第三方包在 `Packages/`（含 `Library/PackageCache`）

## 2. Bot 代码隔离与组织

按游戏是否使用 asmdef 选隔离策略：

- **游戏代码有自己的 asmdef**：助手代码用单独的 assembly + asmdef，助手 asmdef 依赖游戏 asmdef。隔离最干净。
- **游戏代码无 asmdef（全部在预定义的 `Assembly-CSharp`）**：**不要**给助手建 asmdef——asmdef 程序集**无法引用预定义程序集**，会编译不过；给整个游戏补 asmdef 又是高侵入、易破坏构建。此时改用"独立可删除文件夹 + 复用默认程序集 + 命名空间隔离"（如 `Assets/AutoGamer/`，命名空间 `XXX.AutoGamer`）。删除该文件夹即可移除助手。
- 除了必要的 field visibility 修改（private → public），不修改游戏原始代码。最小化对游戏本身的侵入。

## 3. 游戏状态读取（观察器）

- 有 asmdef 依赖时直接读游戏对象的 public/internal 字段；无 asmdef 时同程序集直接读，受限成员用反射
- 观察器收集的数据必须对应步骤1分析报告中识别的关键决策输入；游戏有多个系统（敌人、弹幕、地形、升级）时每个系统单独写观察方法
- **`FindObjectsOfType` 每帧调用会严重拖慢性能**——必须缓存结果 + 定时刷新（0.3-0.5s）；并按 `gameObject.activeSelf` 过滤（对象池中的 inactive 对象也会被返回）
- `Camera.main` 可能为空（尤其 batchmode）——观察器必须对空相机兜底

## 4. 输入注入与唯一注入口

### 输入系统判定与注入路径

先确认游戏用哪套输入系统（`Packages/manifest.json` 是否含 `com.unity.inputsystem`），**两种方式不混用**：

- **新输入系统（com.unity.inputsystem）**：通过 `InputAction` 触发，不模拟按键。移动注入点常是游戏自建的输入管理器只读属性（如 `IInputManager.MovementValue`）——无法直接赋值时，实现一个包装版 `IInputManager`（覆盖移动值、其余成员转发给原实例）并通过游戏的注册入口替换
- **旧 Input API（UnityEngine.Input）**：通过模拟按键或直接调用游戏方法控制

### InputInjector 唯一注入口设计

所有 bot 输入（移动轴、按键、鼠标）必须经这一个静态类注入，录制/回放都挂在同一层，禁止绕过：

```csharp
public static class InputInjector
{
    // 注入 + 通知录制器（双动作）
    public static void InjectAxis(Vector2 vec, string injectMethod = "queue_state_event")
    public static void InjectKeyPress(Key key)
    public static void InjectKeyRelease(Key key)
    public static void InjectMouseWarp(Vector2 screenPos)
    public static void InjectMousePress(MouseButton btn, Vector2 screenPos, string injectMethod = "queue_state_event")
    public static void InjectMouseRelease(MouseButton btn, Vector2 screenPos, string injectMethod = "queue_state_event")
}
```

新输入系统下的注入实现（示例）：

```csharp
Mouse.current.WarpCursorPosition(screenPos);
var state = new MouseState().WithButton(btn, 1f).WithPosition(screenPos);
InputSystem.QueueStateEvent(Mouse.current, state);
InputSystem.Update(); // 若在 Update 之外注入，需手动 Update

// 同步喂给录制器
inputRecorder?.Record(new SequenceEvent {
    frame = Time.frameCount,        // 序列帧索引
    t_unscaled = Time.unscaledTime,
    device = "mouse", action = "press", button = btn.ToString().ToLower(),
    pos = screenPos, inject = injectMethod
});
```

**记录内容必须是"决策翻译后的玩家操作"**，不是内部决策值——这样序列才能脱离 bot 单独驱动游戏。

### UI 点击的两条路径与等价性

| 路径 | 序列中的形态 | 回放等价性 |
|------|-------------|:---:|
| 真实输入事件（warp + press/release，走 EventSystem 射线检测） | `pos` + `inject: queue_state_event` | ✅ 与玩家操作同管线 |
| `ExecuteEvents.Execute<IPointerClickHandler>` 直调 | `target: "Canvas/HUD/BtnStart"` + `inject: execute_events` | ⚠️ 回放必须同样用 ExecuteEvents 直调同一目标；改成"坐标+真实点击"不严格等价（有 UI 遮挡时行为不同） |

`inject` 字段建议取值：`queue_state_event` / `warp_cursor` / `execute_events`。ExecuteEvents 路径记录 GameObject 路径而非坐标；回放时按路径查找目标，路径不存在时立即报告分歧而非静默跳过。

### SequenceReplayer 回放驱动

```csharp
public class SequenceReplayer : MonoBehaviour
{
    private int nextEventIdx;
    void LateUpdate() // 游戏逻辑之后、本帧结束前检查是否有事件该在本帧注入
    {
        int now = Time.frameCount;
        while (nextEventIdx < events.Count && events[nextEventIdx].frame <= now)
        {
            InjectEvent(events[nextEventIdx]); // 走同一个 InputInjector
            nextEventIdx++;
        }
    }
}
```

- **逐帧追赶**：某帧事件多于一个时按序全部注入；若回放局帧号越过下一事件的 `frame` 超过阈值（如 >30 帧）记警告——帧率异常低是轨迹分歧的前兆
- replay 模式下决策引擎必须完全禁用，任何"顺手帮忙"的兜底决策都会污染对比；工程框架（自动开局、终局退出）照常工作
- 光标：`Cursor.lockState` 锁定时 warp 无效，注入前检查并记录

## 5. 时间、速度与暂停

### 速度与计时

- **速度**：`Time.timeScale`（speed 3-4x 即 timeScale 3-4）。高 timeScale 下敌人每个决策帧多走 N 步，反应式 bot 会反应不及——调优/验证用 speed=1
- **每帧重新断言 `Time.timeScale = 目标speed`**：很多游戏在关闭暂停 UI（升级/结算/宝箱…）时把 timeScale 复位为 1，加速被悄悄取消；必要时同步放宽 `Time.maximumDeltaTime`、解除 `Application.targetFrameRate` 限制
- **计时一律用 `Time.unscaledTime`**：speed 缩放下 `Time.time` 失真（speed=4 时差 4 倍），战斗 log 存活时长、录制时间戳、看门狗超时都基于 unscaled
- 验证手法：无人值守跑通后核对"log 墙钟时长 vs 游戏内时长"是否符合 speed 倍率；接近 1:1 说明 timeScale 被复位或后台暂停在作怪

### 失焦不暂停

- 运行时无条件 `Application.runInBackground = true`，并在 ProjectSettings 里也设 `runInBackground: 1`。窗口失焦（并发跑、最小化）时默认自动暂停，整个 sweep 停滞、墙钟空耗

### 暂停语义与阻塞式弹窗（Unity 特有陷阱）

很多游戏用 `Time.timeScale = 0` 暂停并弹出需要点击的 UI。bot 若不逐个处理会**永久卡死**：

- **处理逻辑要在 `Update` 里先于"暂停就 return"的早退执行**——否则结算屏/升级屏自己把 timeScale 置 0，bot 永远走不到检测分支（典型 bug：通关检测不到，胜利被误判为超时）
- 点击 UI 要点**真正可交互的那个**（用按钮的 `Selectable` 状态/按名字匹配，而非盲点第一个；有些"跳过"按钮在动画结束后变成空操作）
- 点击后**等面板真正关闭再做下一步**（关闭常是带动画的协程，重复点击会打断关闭、卡在半开状态）

**优先走逻辑层（headless 关键）**：UI 按钮回调里常夹带渲染相关副作用（弹世界空间文字、播放粒子/动画、依赖 `Camera.main`/shader），`-batchmode -nographics` 下会 NPE 导致面板关不掉、整局卡死。实测踩坑：某游戏升级选卡的 `OnAbilitySelected()` 会 `SpawnText(...)` 弹世界空间提示而 NPE。正确分层：

1. **逻辑层应用 + 逻辑层关闭**（最稳）：调 manager 应用效果（如 `AbilitiesManager.AddAbility/IncreaseAbilityLevel`），再调面板 `Hide()`/关闭方法，并**确保 `onClosed` 回调被触发**（很多游戏靠它恢复 timeScale / 继续波次；手动关则自己 invoke）
2. **读数据字段不依赖动画**：候选项数据（如 `slot.AbilityData`）在面板打开时就赋值，绑定视觉的字段（如 `MainCard`）要等滚动动画结束才有值——优先读前者
3. **模拟点击兜底**：仅当无法走逻辑层时才触发 `Selectable` 的 `onClick`；对"回调 NPE 但效果已生效"做容错
4. **强制关闭看门狗**：面板开启超过 N 秒（unscaled）仍未关，强制置 `IsOpen=false`、`timeScale=1`、触发 `onClosed`、隐藏 GameObject——**永不死锁**

> run-mode=replay / 等价性验证时，阻塞 UI 必须改走输入路径（warp + 真实鼠标事件），逻辑层直调仅在非等价模式保留；等价模式用 `-batchmode` 不加 `-nographics` 规避 UI 回调渲染副作用。用反射访问 protected 的 `slots`、`OnAbilitySelected`、`Hide`、`onClosed`、`IsOpen` 是常见且可接受的。

## 6. 视频录制

**专用录制 Camera + RenderTexture + ffmpeg pipe**（通用模式与 ffmpeg 命令见 `references/video-recorder-reference.md`）：

- 创建独立 GameObject + Camera：`recordCamera.CopyFrom(mainCam)` 复制主相机参数，`recordCamera.depth = mainCam.depth - 1`（在主相机前渲染）
- `recordCamera.targetTexture = renderTexture`；**绝不碰 `Camera.main.targetTexture`**——会导致主相机停止向屏幕渲染（画面冻结、UI 叠加）
- 每帧 `recordCamera.Render()` → `RenderTexture.active` → `captureTexture.ReadPixels` → `GetRawTextureData()` 写 ffmpeg stdin
- 录制结束销毁独立相机与 RenderTexture
- **黑帧**：`-nographics` 不做 GPU 渲染，录出来全黑——需要 VLM 可用视频时用 `-batchmode`（不加 `-nographics`，Unity 允许无窗口渲染到 RenderTexture）；纯 log 迭代才用 `-batchmode -nographics`

关键代码结构：

```csharp
public class VideoRecorder : MonoBehaviour
{
    private Camera recordCamera;
    private RenderTexture renderTexture;
    private Texture2D captureTexture;
    private Process ffmpegProcess;
    private bool isRecording;

    // 命令行参数: -record-video / -video-fps / -video-resolution

    public void StartRecording(string outputPath)
    {
        renderTexture = new RenderTexture(videoWidth, videoHeight, 24);
        recordCamera.targetTexture = renderTexture;
        captureTexture = new Texture2D(videoWidth, videoHeight, TextureFormat.RGBA32, false);

        ffmpegProcess = new Process();
        ffmpegProcess.StartInfo.FileName = "ffmpeg";
        ffmpegProcess.StartInfo.Arguments = BuildFfmpegArgs(outputPath);
        ffmpegProcess.StartInfo.UseShellExecute = false;
        ffmpegProcess.StartInfo.RedirectStandardInput = true;
        ffmpegProcess.StartInfo.RedirectStandardError = true;
        ffmpegProcess.Start();
        isRecording = true;
    }

    public void CaptureFrame()
    {
        if (!isRecording) return;
        recordCamera.Render();
        RenderTexture.active = renderTexture;
        captureTexture.ReadPixels(new Rect(0, 0, videoWidth, videoHeight), 0, 0);
        captureTexture.Apply();
        RenderTexture.active = null;
        byte[] rawBytes = captureTexture.GetRawTextureData();
        ffmpegProcess.StandardInput.BaseStream.Write(rawBytes, 0, rawBytes.Length);
    }

    public void StopRecording()
    {
        isRecording = false;
        ffmpegProcess.StandardInput.BaseStream.Close();
        ffmpegProcess.WaitForExit(30000); // 最多等 30 秒
        Destroy(renderTexture);
        Destroy(captureTexture);
    }

    private string BuildFfmpegArgs(string outputPath)
    {
        return $"-y -f rawvideo -vcodec rawvideo -pixel_format rgba " +
               $"-colorspace bt709 -video_size {videoWidth}x{videoHeight} " +
               $"-framerate {videoFps} -loglevel warning -i - " +
               $"-c:v libx264 -pix_fmt yuv420p -crf 23 \"{outputPath}\"";
    }
}
```

## 7. 构建、启动与无人值守运行

### 命令行构建

- Editor 脚本 + `Unity -batchmode -quit -projectPath ... -executeMethod <Build>`；构建是长操作，用后台进程 + 日志轮询
- **构建期注入 scripting define 必须用 `BuildPlayerOptions.extraScriptingDefines`**，不要用 `PlayerSettings.SetScriptingDefineSymbols`——后者在同一个 batchmode `-executeMethod` 会话里**不会触发重新编译**，gated 代码会被静默编译掉（实测踩坑：某 SDK 生命周期代码没进包，运行时毫无反应）
- Linux Dedicated Server 打包需先装对应构建模块，并设 Server subtarget（云端批量执行需要，见 `references/cloud-sweep-multiverse.md`）

### 构建卫生（每次 batchmode 构建前必做）

Unity 同一时刻只允许一个进程打开同一工程。上一次的编辑器进程或 crash 残留持有工程锁，新构建直接报 "another Unity instance is running" 失败：

1. 杀残留编辑器进程——**按精确 PID 或 `pkill -x Unity` / `pkill -x Tuanjie` 全词匹配，严禁 `pkill -f` 按命令行内容匹配**（实测事故：代理执行 `pkill -f "Tuanjie.*batchmode"`，模式匹配到嵌在它自己命令行里的任务全文，代理自杀、自动化会话丢失）。候选清单里混入的自动化代理/会话进程绝不杀
2. 删除 `<projectPath>/Temp/UnityLockfile`（不存在时忽略）
3. 再启动 batchmode 构建

构建后从日志轮询自定义标志（`build SUCCEEDED` / `build FAILED` / `error CS`），不要只看进程退出码。

### 启动与运行模式

| 用途 | 启动参数 | 说明 |
|------|---------|------|
| 纯 log 迭代 / 吞吐 sweep | `-batchmode -nographics` | 最快，无视频 |
| 需要 VLM 视频的批处理 | `-batchmode`（不加 `-nographics`） | 无窗口渲染，UI 射线/回调正常 |
| 排查 bot 行为 | 直接带窗口运行 | 肉眼观察 + 真实视频 |

- 无头下务必验证"自动开局→注入→对局→终局"全链路真的走通，不要只看进程启动成功
- **并发存档隔离**：concurrency>1 时给每个实例独立的 `persistentDataPath`（否则存档互相覆盖、数据竞争/损坏）
- **fresh run**：无人值守每次开全新一局，主动清掉存档里的续玩进度（关卡时间/经验/已有能力），否则会"继续上一局"
- 游戏结束后进程应在 5 秒内退出，sweep 脚本超时后强制 kill 兜底

## 8. 已知坑与版本兼容

**详细清单见 `references/unity-version-compat.md`**，高频项：

| API | 版本差异 | 安全写法 |
|-----|---------|---------|
| 对象搜索 | `FindObjectsByType<T>()` 仅 2022.2+ | 用 `FindObjectsOfType<T>()`（全版本兼容）或条件编译 |
| 内置字体 | Arial.ttf（旧）vs LegacyRuntime.ttf（2022+） | 先试 Arial 再试 LegacyRuntime，都失败跳过文字 |
| EventSystem | 旧版本需手动加 `StandaloneInputModule` | `FindObjectOfType<EventSystem>()` 为空时手动添加 |
| Camera.main | batchmode 下可为空 | 永远 null 检查 + fallback |
| UI 创建 | 字体/Canvas 不可用会崩溃 | try-catch 包裹，不可用时跳过 |

其余必做：`ProjectSettings/ProjectVersion.txt` 确认版本；对 `FindObjectsOfType`、`Resources.GetBuiltinResource`、`Camera.main` 全部做 null 检查与 fallback；`Application.isBatchMode` 判断无头模式；录制相机绝不劫持主相机 targetTexture。

# Unreal Playbook — 自动化游戏助手落地手册

> 读者：已读过引擎无关版 SKILL.md、即将在 Unreal 工程落地 bot 的 AI 代理。本文只写 Unreal 侧的落地细节：API 选型、注入路径、构建与无人值守跑法、版本坑。通用流程（分析→策略→编码→sweep→迭代）不在此复述。
>
> 阅读顺序约定：§1 识别技术栈 → §2 决定代码放哪 → §3/§4 决定怎么读、怎么注入 → §5-§7 决定怎么跑 → §8 查已知坑。所有序列事件字段与 `references/sequence-record-replay.md` 的 schema 对齐（§4 有映射表）。

## 0. 通用概念 → Unreal API 对照表

| 通用概念 | Unreal API/做法 | 注意事项 |
|---|---|---|
| 帧索引 | 全局变量 `GFrameCounter`（uint64，引擎每帧自增） | 回放唯一时间基准。别用 `GFrameNumber`（渲染线程语义，`-nullrhi`/专用服务器不渲染时不推进） |
| 不受速度影响的时钟 | `FPlatformTime::Seconds()`（进程启动以来的墙钟秒） | 序列 `t_unscaled`、log 时间戳、墙钟超时都用它。`AWorld::GetTimeSeconds()` 受 TimeDilation 缩放，勿用作 unscaled 基准 |
| 游戏速度控制 | `UGameplayStatics::SetGlobalTimeDilation(World, N)`（写 `AWorldSettings::TimeDilation`；等价控制台 `slomo N`）；更稳的跑法见 §5 的 `-benchmark` | 游戏关 UI 面板时常把它复位回 1——bot 每帧重新断言 |
| 对象/Actor 遍历 | `TActorIterator<T>(World)`、`TObjectIterator<T>()`、`UGameplayStatics::GetAllActorsOfClass(...)` | 三者逐个都会分配/扫描全表，严禁每帧全量调用；缓存 + 定时刷新（§3） |
| 输入注入 | Enhanced Input：`UEnhancedPlayerInput::InjectInputForAction`（action 级）与 `Input.+key` 控制台命令（key 级）；通用：`FSlateApplication::ProcessKeyDownEvent/ProcessMouse...`（Slate 全路径）；旧输入：`APlayerController::InputKey/InputAxis` | 全部收口到唯一 InputInjector（§4），注入路径记进序列 `inject` 字段 |
| 视频帧采集 | `USceneCaptureComponent2D` → `UTextureRenderTarget2D` → `FRenderTarget::ReadPixels(TArray<FColor>&, FReadSurfaceDataFlags, FIntRect)` | `FColor` 内存序是 **BGRA** → ffmpeg 用 `-pixel_format bgra`；`-nullrhi` 下全黑帧（§6） |
| 用户存档目录 | 打包游戏默认写到 `<安装目录>/Saved`；启动参数 `-SaveToUserDir` 切到用户目录；代码侧取 `FPaths::ProjectSavedDir()` | **没有**官方 `-saveddir` 类参数可自定义路径，per-instance 隔离见 §7【待验证】 |
| 无头运行 | `-nullrhi`（官方描述 "run UE headless"，无 GPU 渲染）；窗口不可见渲染用 `-RenderOffScreen` | 无头下视频必黑、部分 UI/渲染回调会异常；分层运行模式见 §6 |
| 命令行/环境变量传参 | 解析：`FParse::Value(FCommandLine::Get(), TEXT("speed="), V)` / `FParse::Param(...)`；环境变量：`FPlatformMisc::GetEnvironmentVariable(TEXT("AG_SPEED"))` | 自定义参数游戏代码不读则被忽略——bot 启动处理器必须自己解析（§7） |
| 命令行构建 | UAT：`<Engine>/Engine/Build/BatchFiles/RunUAT.bat`（Windows）/ `RunUAT.sh`（Mac/Linux，个别版本在 Mac/、Linux/ 子目录，以实际文件为准）→ `BuildCookRun ...` | 完整示例与构建锁见 §7；同一工程不要并行两次 UAT |
| 游戏内退出进程 | `FPlatformMisc::RequestExit(false)`（优雅）/ `RequestExit(true)`（立即）；BP 侧 `UKismetSystemLibrary::QuitGame` | 终局后 5 秒内退出是 sweep 的硬要求，sweep 侧超时强杀兜底 |
| 日志输出 | `-log`（Windows 开独立日志窗；类 Unix 落盘 Saved/Logs/）、`-ABSLOG=<绝对路径>`、`-stdout` | battle log 建议单行紧凑 JSON 直接打日志，sweep 从日志或文件两侧都能抓 |

## 1. 技术栈识别

落地前先看工程根目录的文件标志，逐项确认后再动手。

| 文件/目录 | 含义 | 对 bot 的影响 |
|---|---|---|
| `<Game>.uproject` | 工程描述 JSON。`EngineAssociation` 决定引擎："5.3"/"4.27" = Launcher 安装版；`///绝对路径` 或相对路径 = 源码版 | 决定 RunUAT 路径；Launcher 版**能**编译游戏侧 C++（含项目插件），只是不能改引擎源码——bot 插件完全够用 |
| `Source/` + `*.Build.cs` | C++ 工程标志（.uproject 有 `"Modules"` 键） | bot 插件可直接依赖游戏模块、include 游戏头文件（§2） |
| 只有 `Content/` 无 `Source/` | 纯蓝图（BP-only）工程 | bot 无游戏模块可依赖，只能反射读取（§3 末节） |
| `Content/` | uasset 资产、蓝图类都在这里 | 反射按 `/Game/...` 对象路径找类 |
| `Config/Default*.ini` | 引擎/游戏配置 | 识别输入系统、默认地图、GameInstance 类 |
| `Binaries/`、`Saved/` | 编译产物 / 运行时产物 | 观察游戏默认把存档写到哪（§7 存档隔离的前提） |

**UE4 还是 UE5、输入系统是哪套**——按证据判断，不要只看版本号：

```ini
; Config/DefaultInput.ini 中出现这两行 = Enhanced Input 工程
+DefaultPlayerInputClass=/Script/EnhancedInput.EnhancedPlayerInput
+DefaultInputComponentClass=/Script/EnhancedInput.EnhancedInputComponent
```

| 引擎/输入组合 | 判定 | bot 注入路径 |
|---|---|---|
| UE4.25- | 只可能是旧输入（`UPlayerInput` + Action/Axis Mappings） | §4 旧路径（`APlayerController::InputKey/InputAxis`） |
| UE4.26-4.27 装了 EnhancedInput 插件 | Experimental，两套可能并存 | 以 ini 为准；两套并存时行为不可预测，问代码 |
| UE5.0+ | EI 正式支持；UE5.1 起模板默认 EI、旧映射标记弃用；UE5.3 起 EI 是默认且旧映射弃用警告升级 | §4 Enhanced Input 路径优先 |

> **识别优先级**：ini 类配置 > 游戏代码实际绑定方式 > 引擎版本号。工程可能是"UE5 但仍用旧输入"（升级上来的），此时按旧路径注入。

## 2. Bot 代码隔离与组织

首选**独立 Game Plugin**（含 Runtime 模块），不要往 `.uproject` 的 `Modules` 里塞模块：

```text
<Proj>/Plugins/AutoGamer/
├── AutoGamer.uplugin          # 见下
└── Source/AutoGamer/
    ├── AutoGamer.Build.cs      # 模块依赖，见下
    ├── AutoGamer.h/.cpp        # IMPLEMENT_MODULE
    ├── BotDriver.cpp           # 自建无渲染 Actor：每帧 弹窗处理→速度断言→决策→注入
    ├── GameStateObserver.cpp   # §3
    ├── InputInjector.cpp       # §4 唯一注入口（录制/回放同挂）
    ├── SequenceRecorder.cpp / SequenceReplayer.cpp
    ├── BattleLogRecorder.cpp / VideoRecorder.cpp   # §6
    └── BotStartup.cpp          # GameInstance/World 委托挂钩 + 命令行/环境变量解析
```

```json
// AutoGamer.uplugin —— 删除整个 Plugins/AutoGamer 目录即彻底移除 bot
{
  "FriendlyName": "AutoGamer", "Version": 1, "VersionName": "0.1",
  "Description": "Automation bot for game sweeps",
  "Modules": [
    { "Name": "AutoGamer", "Type": "Runtime", "LoadingPhase": "Default" }
  ]
}
```

```csharp
// AutoGamer.Build.cs —— 两种依赖模式二选一（见下表）
public class AutoGamer : ModuleRules
{
    public AutoGamer(ReadOnlyTargetRules Target) : base(Target)
    {
        PublicDependencyModuleNames.AddRange(new string[] {
            "Core", "CoreUObject", "Engine", "InputCore",
            "EnhancedInput",   // UE4.26+/UE5 工程；工程没启用该插件则删
            "MyGame"           // 模式A：直接依赖游戏主模块；BP-only 工程删掉
        });
    }
}
```

| 依赖模式 | 做法 | 优点 | 代价 |
|---|---|---|---|
| A. 编译期依赖游戏模块 | Build.cs 加游戏模块名，直接 include 游戏类 | 强类型、编译期查错、观察器最简 | bot 与游戏版本耦合；游戏模块改名/重组需跟 |
| B. 纯反射（不依赖） | `FindPropertyByName` / `FindFunctionByName` + `ProcessEvent`（§3） | 零耦合；BP-only 工程唯一可行；删目录更干净 | 字符串易错（改名运行期才炸）、要缓存属性指针、慢一些 |

硬性原则：

- 不改引擎源码，不改游戏 gameplay 代码。游戏字段是 private/protected 时优先反射读，**不要**为了 bot 改游戏代码可见性。
- "删目录即移除"要实测：删 `Plugins/AutoGamer` 后重编、打包、双击运行，游戏行为应与无 bot 完全一致。
- bot 的所有自建对象（观察缓存、录制器、RT、ffmpeg 进程）都由 bot 模块自己创建与销毁，不落任何文件到游戏目录之外。

## 3. 游戏状态读取（观察器）

### 3.1 入口与挂点

| 入口 | API | 用途 |
|---|---|---|
| World | 从任意 `UWorld*` 出发；地图加载完成钩子 `FCoreUObjectDelegates::PostLoadMapWithWorld`；世界初始化 `FWorldDelegates::OnPostWorldInitialization`（过滤 `EWorldType::Game`） | 观察、遍历的锚点 |
| GameInstance | `World->GetGameInstance()` / `UGameplayStatics::GetGameInstance(Context)`；生命周期 `UGameInstanceSubsystem`（`Initialize`/`Deinitialize`） | bot 启动逻辑、全局单例的挂点 |
| 玩家侧 | `UGameplayStatics::GetPlayerController(Context, 0)`、`GetPlayerPawn`、`GetPlayerCharacter` | 输入、血量、位置 |
| 每帧驱动 | 首选 `UTickableWorldSubsystem`（老引擎无此类时退回 `FTickableGameObject`）或自建无渲染 Actor；**必须 `PrimaryActorTick.bTickEvenWhenPaused = true`**（§5 暂停陷阱） | BotDriver 的 tick 宿主 |

### 3.2 遍历与性能陷阱

| API | 范围 | 症状 → 解法 |
|---|---|---|
| `TActorIterator<T>(World)` | 单个 World 内的某类 Actor（含刚生成的） | 每帧全量迭代大关卡 → 帧率崩。→ 缓存 `TArray<TWeakObjectPtr<AActor>>`，0.3~0.5s 定时刷新，逐项 `IsValid()` 过滤后再用 |
| `TObjectIterator<T>()` | **全进程所有已加载 UObject**，含 CDO、编辑器残留对象 | 症状：一次迭代几十万对象、卡顿数百毫秒。→ 只用于启动期/低频定位（找 BP 实例），逐项过滤 `!It->HasAnyFlags(RF_ClassDefaultObject)` 且对象所在 World 是目标 World |
| `UGameplayStatics::GetAllActorsOfClass` | 每次调用**新分配**一个数组 | 症状：每帧调用产生大量 GC 压力。→ 同样缓存 + 定时刷新；新实体走游戏的生成事件委托增量维护 |

缓存骨架（所有观察数据同构处理）：`TMap<UClass*, TArray<TWeakObjectPtr<AActor>>> Cache` + `double LastRefresh`（`FPlatformTime::Seconds()`），`Refresh(UWorld*)` 里用 `TActorIterator` 重填，间隔 0.3~0.5s。

> 裸指针缓存的 GC 威胁见 §8（症状是随机崩溃/脏数据，必须 `TWeakObjectPtr`）。

### 3.3 配置与数据表读取

| 数据源 | API | 注意 |
|---|---|---|
| DataTable | `UDataTable::FindRow<FMyRow>(RowName, TEXT("ctx"), /*bWarnIfRowMissing*/true)` | 行结构类型要从游戏代码/资产里确认，模板参数错了读出来是垃圾值 |
| DataAsset/蓝图资产 | `LoadObject<UObject>(nullptr, TEXT("/Game/Data/DropTable.DropTable"))`、`StaticLoadClass` | 路径用资产右键 Copy Reference；打包后必须被引用否则不进 pak |
| 类默认值（CDO） | `GetDefault<UGameModeBase>()` 等读默认配置 | 读"设计数值"而不运行时实例 |
| ini 配置 | `GConfig->GetString(Section, Key, Out, IniPath)` | 游戏自定义节通常在 `DefaultGame.ini` |

### 3.4 蓝图字段/函数从 C++ 访问（反射）

蓝图变量（含 private）都在 UClass 反射里，模式 B（零依赖）全靠这条路：

```cpp
UClass* Cls = Obj->GetClass();                       // UClass/FProperty 在类存活期稳定，按类缓存
if (FFloatProperty* P = CastField<FFloatProperty>(Cls->FindPropertyByName("CurrentHP")))
    float HP = P->GetPropertyValue_InContainer(Obj);
if (FBoolProperty* B = CastField<FBoolProperty>(Cls->FindPropertyByName("bIsOpen")))
    bool Open = B->GetPropertyValue_InContainer(Obj);
if (UFunction* Fn = Cls->FindFunctionByName("OnAbilitySelected"))
    Obj->ProcessEvent(Fn, &Params);                  // 参数按函数签名手工构造内存布局
```

- 复杂结构体属性：`FStructProperty` → `ContainerPtrToValuePtr<T>` 取指针。
- `ProcessEvent` 传参/返回的内存布局要对照函数签名构造，错位即静默读脏数据——写完先用已知值单测一次。
- **症状：反射读出的数值恒为 0/空** → 解法：类名/属性名拼写（区分大小写）、BP 变量设了 `Transient`（运行期不存实例）、或实例不是你以为的那个类。

### 3.5 仅蓝图工程（无 C++）

- 没有游戏模块可依赖——Build.cs 删掉游戏依赖，全部走 3.4 反射；蓝图类按 `/Game/...` 路径 `StaticLoadClass` 加载。
- 生成 C++ 工程的方法见 §8（建议生成，编译期错误比运行期反射错误便宜得多）。
- 输入词汇表照常从蓝图资产里梳理（IMC/IA 资产的 key 绑定、按钮 OnClicked 绑定函数名），落到 §4 的注入原语。

## 4. 输入注入与唯一注入口

### 4.1 注入路径选型（对应序列 `inject` 字段）

| inject 标签 | API | 走到哪 | 等价性（对应 Unity 版语义） |
|---|---|---|---|
| `slate_process_key_event` / `slate_process_mouse_event` | `FSlateApplication::Get().ProcessKeyDownEvent/ProcessKeyUpEvent/ProcessMouseButtonDownEvent/ProcessMouseButtonUpEvent/ProcessMouseMoveEvent` | Slate 全路径：UMG 命中 → 视口 → PlayerController → 输入系统（Enhanced/旧都吃） | ≈ `queue_state_event`：与真实硬件同管线，最严格 |
| `enhanced_inject_action` | `ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer())->GetPlayerInput()->InjectInputForAction(Action, Value)` | 直接进该 InputAction，**绕过 IMC key 映射与触发器** | action 级注入；回放必须同样调 InjectInputForAction |
| `input_plus_key` | 控制台命令 `Input.+key <Key> X=0.7 Y=0.5`（释放 `Input.-key <Key>`），经 `PC->ConsoleCommand` 执行 | 进引擎输入栈，保留 IMC 映射/触发器 | key 级注入；官方文档路径 |
| `pc_input_key` | `APlayerController::InputKey(FKey, EInputEvent, bAmountDepressed, bGamepad)` / `InputAxis(...)` | 旧输入系统；**UE5 已标记弃用**，EI 工程下不保证触发 Action（见下） | 仅旧输入工程使用 |
| `umg_onclicked` | 反射找 `UUserWidget::GetWidgetFromName("BtnStart")` 后 `UButton::OnClicked.Broadcast()` | 只走按钮委托，跳过 Slate 按压/焦点/命中 | ≈ `execute_events`：非严格等价（UI 遮挡/焦点态不同）；回放按记录的 widget 路径同法直调 |
| `warp_cursor` | 光标定位（见 4.4） | — | ≈ `warp_cursor` |

选型规则：**同一原语全程只用一条路径，录制与回放同路径**（`inject` 字段 + 回放侧断言）。EI 工程优先级：移动轴/离散动作 → `enhanced_inject_action`（最可控）；需要"像真实按键一样"（连招、按住类触发器、UI 与游戏同帧响应）→ Slate 合成事件；旧输入工程 → `pc_input_key`。

> `UInputInjectionComponent`：据 5.3+ 社区资料另有专用于自动化的注入组件。【待验证】——在你的引擎类浏览器里搜 "InputInjection"；不存在就忽略，上表路径已覆盖需求。

### 4.2 Enhanced Input（UE5 默认）注入要点

```cpp
// 移动轴：直接向 MoveAction 注入 Vector2D（IA 资产用 LoadObject 从 /Game/... 加载，缓存 UInputAction*）
if (ULocalPlayer* LP = PC->GetLocalPlayer())
    if (auto* EI = LP->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>())
        EI->GetPlayerInput()->InjectInputForAction(MoveAction, FInputActionValue(FVector2D(X, Y)));
// 离散动作：值为 bool/float 均可，FInputActionValue 支持四种类型
```

- 注入绕过 IMC——若游戏逻辑依赖 IMC 的 modifier（Negate/Swizzle 等），注入侧要自己给出最终值；用 `showdebug enhancedinput` 验证动作是否真被触发。
- 症状：注入后动作没反应 → 解法：查 `EI` 为空（服务器/无 LocalPlayer，见 §7 专用服务器）、IA 资产加载失败、或该动作要求 IMC 处于激活态（先补 `AddMappingContext`）。

### 4.3 FSlateApplication 合成事件（UI 与全路径）

```cpp
const FModifierKeysState Mods;   // 空修饰键
// 键盘：构造参数顺序以本引擎 FKeyEvent 定义（SlateCore）为准
FKeyEvent KeyEv(EKeys::W, Mods, /*KeyCode*/0, /*bIsRepeat*/false, /*UserIndex*/0, /*CharCode*/0, /*Char*/0);
FSlateApplication::Get().ProcessKeyDownEvent(KeyEv);      // 释用 ProcessKeyUpEvent
// 鼠标按下（FPointerEvent 同理：位置写进 ScreenSpacePosition）
FPointerEvent PtrEv(/*PointerIndex*/0, FVector2D(X, Y), FVector2D::ZeroVector,
                    FVector2D::ZeroVector, EKeys::LeftMouseButton, FVector2D::ZeroVector, Mods);
FSlateApplication::Get().ProcessMouseButtonDownEvent(/*ViewportIndex*/0, PtrEv);
```

- 与真实硬件同管线：UMG 命中测试、焦点、再转发给游戏——对"UI 点击"这是唯一严格等价路径。
- 无头差异：`-nullrhi` 下 FSlateApplication 通常仍存在（Slate 用空渲染器），但**事件是否走通到游戏需实测**【待验证：注入后加一条 bot 日志，在 `-nullrhi` 与窗口模式各跑一次对比】；commandlet 进程无 Slate，不可用此路径。
- 症状：Slate 合成键事件没进游戏动作 → 解法：焦点不在视口（先合成一次视口点击/调 `FSlateApplication` 焦点 API）、或 ViewportIndex 不对；逐一用 `showdebug enhancedinput` 定位。

### 4.4 UI 点击：UMG 直调 vs 合成鼠标事件

| 路径 | 写法 | 序列形态 | 何时用 |
|---|---|---|---|
| UMG 直调 | 反射找 widget → `OnClicked.Broadcast()` 或 ProcessEvent 目标函数 | `"target": "WBP_MainMenu:BtnStart", "inject": "umg_onclicked"` | 非等价模式优化：无头下最稳（不依赖 Slate 命中与渲染回调） |
| 合成鼠标 | `ProcessMouseButtonDownEvent` 携带目标坐标 | `"pos": [x, y], "inject": "slate_process_mouse_event"` | replay/等价性验证模式：与玩家点击同管线，有遮挡时行为才与真实一致 |

> 与 Unity 版结论一致：**等价模式下阻塞 UI 必须走输入路径**（合成鼠标），`umg_onclicked` 仅作为非等价模式的优化；强制关闭看门狗（面板超时 N 秒未关则广播关闭 + 恢复速度 + 触发关闭回调）两种模式都保留。

光标定位（`warp`）：UMG 命中只认事件携带的坐标，合成 `ProcessMouseMoveEvent` 带新位置即可（无需真移 OS 光标）；仅当游戏逻辑读真实光标（视口射线、FPS 视角）才需要移 OS 光标——用 `FSlateApplication::Get().GetPlatformCursor()` 的 ICursor 定位接口（`SetPosition`；个别平台有 `SetPositionSilently`）【待验证：确切函数名以本引擎 `ICursor` 接口为准】。HiDPI/Retina 屏上桌面坐标与视口像素存在缩放换算——meta 里记录视口分辨率，回放两侧换算保持一致。

### 4.5 InputInjector 静态类（唯一注入口 + 逐事件录制）

```cpp
// 唯一注入口：注入 + 逐事件录制。FSeqEvent 的字段 = sequence-record-replay.md 的 schema
// （frame/t_unscaled/device/action/key/button/pos/vec/inject，另有 target 仅逻辑直调路径有）
namespace BotInputInjector {
    // auto 模式接录制器、replay 模式接空实现（回放走同一注入口以产出 sequence_B）
    TFunction<void(const FSeqEvent&)> OnInjected;
    void InjectKey(FKey Key, bool bPressed, const TCHAR* InjectTag);
    void InjectMoveAxis(FVector2D Vec);                 // → enhanced_inject_action / pc_input_key
    void InjectMouseClick(FVector2D Pos, bool bPressed); // → slate_process_mouse_event
    void InjectMouseWarp(FVector2D Pos);                 // → warp_cursor
    void InjectUIButton(UObject* Widget, FName Name);    // → umg_onclicked（非等价模式）
}
// 每次注入的收尾：
// FSeqEvent Ev{ GFrameCounter - RunStartFrame, FPlatformTime::Seconds(), ... };
// OnInjected(Ev);   // 录制器写 sequence.json；决策引擎禁止绕过本注入口散调输入 API
```

时间字段逐项核实结论：

| schema 字段 | Unreal 来源 | 说明 |
|---|---|---|
| `frame` | `GFrameCounter`（**相对 run-start 锚点**：记录 `GFrameCounter - RunStartFrame`，锚点帧写入 meta） | 引擎全局计数器，与渲染无关；锚点防止两次启动的加载耗时差异造成帧漂移。锚点定义在"对局开始"的生命周期点（如 PostLoadMapWithWorld 之后首个 bot tick），A/B 两局必须同点重锚 |
| `t_unscaled` | `FPlatformTime::Seconds()` | 进程墙钟，不受 TimeDilation/pause 影响。不要用 `AWorld::GetTimeSeconds()`；`World->GetRealTimeSeconds()` 以 world 起点计，跨关重置，跨关的序列不要混用 |
| `device` / `action` | `keyboard`/`mouse`；`press`/`release`/`axis`/`warp` | 与 schema 完全同名 |
| `key` | `FKey` 名：`EKeys::W.GetFName().ToString()` → `"W"`、`"SpaceBar"`、`"Gamepad_Left2D"` | 与 `Input.+key` 用的 key 名同源（InputCoreTypes），可直接互换 |
| `button` | `LeftMouseButton→"left"`、`RightMouseButton→"right"`、`MiddleMouseButton→"middle"` 映射 | 与 schema 取值一致 |
| `pos` | 视口/屏幕像素坐标，**UE 惯例左上原点**（Unity schema 是左下原点） | 在 meta 里写 `"pos_origin": "top-left"`，诊断/diff 脚本不得沿用 Unity 假设 |
| `vec` | `FVector2D` 归一化（x=右，y=前，按游戏约定写死） | 移动轴节流规则同 schema：值变化才记录，回放保持上次值 |
| `inject` | 见 4.1 表的标签 | 回放必须用同一标签路径，replayer 里加断言 |

回放驱动：replayer 的 tick 组与 bot 决策 tick 保持一致（默认 `TG_PrePhysics`），逐帧追赶"锚点帧 ≤ 当前锚点帧"的全部事件按序注入；超阈值（如落后 >30 帧）记警告——帧率异常的前兆。序列结束/终局即退出进程，与 auto 模式一致；replay 模式决策引擎必须完全禁用。

## 5. 时间、速度与暂停

### 5.1 速度：两条路

| 方式 | 写法 | 语义与坑 |
|---|---|---|
| 全局时间膨胀 | `UGameplayStatics::SetGlobalTimeDilation(World, N)`；等价控制台 `slomo N`；底层 `AWorldSettings::TimeDilation`；单 Actor 用 `AActor::CustomTimeDilation` | 每帧游戏 delta × N，逻辑、动画、物理一起加速。Per-Actor `CustomTimeDilation` 会与全局值**相乘**——读游戏代码确认它没给关键实体设特殊膨胀 |
| 引擎基准模式 | 启动参数 `-benchmark`（引擎以最快速度 tick）；配合 `-FPS=60` 固定每帧 dt（"Override fixed tick rate"）。C++ 侧对应 `FApp::SetBenchmarking` / `FApp::SetFixedDeltaTime`（以本引擎 `Misc/App.h` 为准【待验证：不同版本函数名】） | 固定 dt、跑满 CPU——高速 sweep 的**更优解**：物理步长稳定（不像膨胀法在高倍率+低帧率下抖动/穿透），游戏时间快于墙钟。墙钟类逻辑（真实倒计时、unscaled 定时器）会失真；等价性验证不要用它 |

**每帧重新断言速度**：BotDriver tick 的固定顺序是「处理阻塞 UI → 断言速度 → 决策 → 注入」。游戏关暂停面板/升级面板时常把 TimeDilation 复位回 1——症状：speed=4 的 sweep 墙钟与游戏内时长接近 1:1、频繁开面板的局尤甚；解法：每 tick `SetGlobalTimeDilation(World, TargetSpeed)` 无条件重设，并核对 log 的"墙钟时长 vs 游戏内时长"倍率。

**帧率上限对 speed 生效的影响**：

- TimeDilation 是乘 delta 的，`t.MaxFPS`/VSync 的帧率上限**不会**吃掉加速倍率（每帧照样模拟 N×dt）。
- 但上限低 + 倍率高 = 单帧 delta 巨大 → 物理离散化粗、弹幕穿透、与 1x 行为有偏差。调优与等价性验证统一 speed=1（与 SKILL.md 一致）；高速 sweep 要么提高帧率（关 VSync、`-novsync`、`t.MaxFPS 0`），要么用基准模式固定 dt。
- 并发多实例时反过来用 `t.MaxFPS` **做资源平衡**（见 5.3）。

### 5.2 暂停语义与阻塞式弹窗陷阱

`UGameplayStatics::SetGamePaused(World, true)`（返回 bool）会停掉**可暂停的** Actor tick——多数游戏逻辑、移动、计时全停。两个必须做的防护：

1. **BotDriver 必须 `PrimaryActorTick.bTickEvenWhenPaused = true`**——症状：游戏一暂停 bot 整体停摆、永远走不到"检测到结算屏"分支，胜利被误判为超时。
2. **处理弹窗的代码先于一切早退**。BotDriver tick 的第一件事是处理阻塞 UI（结算/升级/复活/确认），然后才允许 `IsGamePaused → return`。同时用 `UGameplayStatics::IsGamePaused(World)` 与 `World->GetWorldSettings()->TimeDilation` 双探测——有的游戏用 TimeDilation=0 而不是 SetGamePaused 实现"暂停"。
3. 强制关闭看门狗照抄 Unity 版规则：面板开超 N 秒（unscaled）未关 → 广播关闭/复位开关状态 + 恢复速度 + 触发关闭回调，保证永不死锁。

### 5.3 失焦/最小化/后台

| 行为 | 事实 | 处理 |
|---|---|---|
| 失焦暂停 | **打包游戏失焦默认不暂停**（引擎没有 Unity 那样的 runInBackground 开关问题）。"Use Less CPU in Background" 是**编辑器**设置，不影响打包 | 无需改设置；但不要用 PIE 验证失焦行为（编辑器行为不代表打包产物） |
| 多实例互相抢资源 | 症状：并发跑多实例时非焦点实例帧率骤降。根因是焦点实例跑满 CPU/GPU 把余量吃光 | sweep 给每实例设 `t.MaxFPS`（如 30~60）做配额平衡；跑满不加限速会饿死失焦实例 |
| 窗口不可见渲染 | `-RenderOffScreen`（官方 "Render off screen"）可无窗口渲染 | 服务器/CI 上要视频时先试它；不可用时退回 Linux Xvfb + `-windowed` |
| 个别版本/平台失焦节流 | 【待验证】不确定该引擎版本是否对失焦窗口做了内部节流 | 验证法：双开两实例，bot 每秒打一条"本帧墙钟 dt"日志，对比失焦/焦点实例的帧间隔是否被掐到固定小值 |

## 6. 视频录制

### 6.1 管线（SceneCapture2D → ReadPixels → ffmpeg）

```cpp
// 挂在 BotDriver Actor 上（组件），专用相机视角跟随游戏主相机或固定俯视
SceneCapture = CreateDefaultSubobject<USceneCaptureComponent2D>(TEXT("Cap"));
RT = NewObject<UTextureRenderTarget2D>(this);
RT->InitAutoFormat(W, H);               // RTF_RGBA8；小分辨率如 512x512，降低回读成本
SceneCapture->TextureTarget = RT;
SceneCapture->bCaptureEveryFrame = false; // 手动节流，别让引擎每帧都采

// 每 N tick（按目标 video-fps 节流，采集帧率独立于游戏帧率）：
SceneCapture->CaptureScene();
TArray<FColor> Px;
if (FTextureRenderTargetResource* Res = RT->GameThread_GetRenderTargetResource())
    Res->ReadPixels(Px);                 // FColor 内存序 BGRA；顶行/底行顺序未定，见坑表
// Px 的原始字节写入 ffmpeg stdin pipe（FPlatformProcess::CreatePipe + CreateProcess，
// WritePipe 接到 ffmpeg 的 stdin；CreateProcess 参数顺序按本引擎 HAL/PlatformProcess.h）
```

```bash
ffmpeg -y -f rawvideo -pixel_format bgra -video_size 512x512 -framerate 10 \
  -loglevel warning -i - -c:v libx264 -pix_fmt yuv420p -crf 23 recording.mp4
```

> 与 Unity 版唯一差异：`-pixel_format bgra`（`FColor` 是 BGRA）。行序先用一张已知画面（左上角画红色方块）拍一帧验证是否需要上下翻转，别猜。

### 6.2 -nullrhi 黑帧问题（硬结论）

- `-nullrhi` = 官方 "run UE headless"，无任何 GPU 渲染：`CaptureScene`/`ReadPixels` 只会产出黑帧或空数据。**要视频就必须有真实 RHI 渲染**，没有绕法。
- SceneCapture 走真实 RHI，但不会在屏幕上显示——配合 `-RenderOffScreen` 或后台窗口即可"不可见地录"。

### 6.3 运行模式分层（照抄选型，别混）

| 模式 | 启动参数 | 用途 |
|---|---|---|
| ① 纯 log 迭代 | `-nullrhi -unattended -nosound -nosplash` | 最快吞吐：VLM 不可用时的纯 log 诊断、大批量统计。视频全黑属预期 |
| ② 需要视频 | `-RenderOffScreen`（或 Linux Xvfb + `-windowed -ResX=512 -ResY=512 -forceres`）+ ① 之外的全部参数 | VLM 视频分析。无窗口渲染不可用时退回小窗口后台跑 |
| ③ 排查行为 | 带可见窗口（不加 `-nullrhi`），肉眼观察 | bot 行为异常时人工对照 |

录制触发与 Unity 版一致：Auto 开启 `StartRecording`、单局结束 `StopRecording`，每局独立 mp4；benchmark 模式下墙钟采集节流要按"锚点帧 mod N"做，不能按墙钟间隔（否则采集率随速度漂移）。

## 7. 构建、启动与无人值守运行

### 7.1 命令行构建（UAT BuildCookRun）

```bash
# Windows（路径是典型形式，EngineAssociation 决定实际根目录）
"C:\Program Files\Epic Games\UE_5.3\Engine\Build\BatchFiles\RunUAT.bat" BuildCookRun ^
  -project="D:\Game\Game.uproject" -platform=Win64 -clientconfig=Development ^
  -build -cook -stage -pak -package -archive -archivedirectory="D:\Out"
# macOS / Linux（RunUAT.sh；个别版本位于 BatchFiles/Mac|Linux/ 子目录）
"/path/UE_5.3/Engine/Build/BatchFiles/RunUAT.sh" BuildCookRun \
  -project="/path/Game.uproject" -platform=Mac -clientconfig=Development \
  -build -cook -stage -pak -package -archive -archivedirectory="/path/Out"
```

参数含义速记：`-build` 编 UBT → `-cook` 烤资产 → `-stage` 收集产物 → `-pak` 打包 → `-package` 成平台目录结构 → `-archive -archivedirectory` 归档到指定目录。产物路径记进报告，run_sweep 用它启动游戏。

构建卫生（症状 → 解法）：

- 症状：UAT 报另一实例占用/构建产物互相覆盖 → 解法：**同一工程绝不并行两次 UAT**；构建前杀残留进程——按可执行名/完整路径精确匹配（`UnrealEditor`、UAT 的 dotnet 进程），严禁 `pkill -f` 模糊匹配命令行内容（会误杀自动化代理自身，Unity 版手册有实测事故）。
- 症状：改了代码但构建行为像没变 → 解法：删 `Binaries/` 与 `Intermediate/Build` 里的增量产物重编（UBT 增量偶尔脏）。
- 结果判定：退出码 + 日志关键字轮询（成功标志 / `error:`），别只看进程退出。
- 只想编译不打包：直接 UBT，`<Engine>/Engine/Build/BatchFiles/Build.bat <Game>Editor Win64 Development -project="..."`（Mac/Linux 用对应平台 Build 脚本）。

### 7.2 游戏启动参数清单（逐条已核实用途）

| 参数 | 用途 | 备注 |
|---|---|---|
| `-log` | Windows 打独立日志窗口；类 Unix 平台日志落 Saved/Logs | 想指定绝对路径用 `-ABSLOG=<path>` |
| `-unattended` | 无人值守：跳过一切需要人工反馈的弹窗/按键确认 | 自动化批跑必加 |
| `-nullrhi` | 无 GPU 渲染的无头运行 | 视频全黑（§6） |
| `-RenderOffScreen` | 无窗口渲染 | 要视频又要无窗口时用 |
| `-windowed` / `-fullscreen` | 窗口/全屏 | 与 `-ResX=N -ResY=N -forceres` 组合控制窗口尺寸 |
| `-ResX=` / `-ResY=` | 设定分辨率 | 降低分辨率=降低渲染/回读成本 |
| `-nosplash` | 去掉启动闪屏 | 无人值守少一个窗口 |
| `-nomovies` / `-nostartupmovies` | 跳过片头视频（后者只跳启动视频） | 加速自动开局 |
| `-nosound` | 关音频输出 | 并发实例大幅省资源 |
| `-novsync` | 关垂直同步 | speed 生效的前提之一（§5.1） |
| `-FPS=N` | "Override fixed tick rate frames per second"——固定每帧 tick 率 | 配 `-benchmark` 用 |
| `-benchmark` | 引擎基准模式：以最快速度运行 | 与 `-deterministic` 同源于官方基准命令行示例；高速 sweep 见 §5.1 |
| `-noscreenmessages` | 关屏幕消息 | 引擎官方基准示例同款，减少噪声 |
| `-server` | 专用服务器模式 | 见 7.6 |

官方基准参考命令行（可直接当 sweep 模板改）：`Game.exe -benchmark -deterministic -FPS=60 -unattended -noscreenmessages -novsync -resx=2560 -resy=1440 -forceres -windowed`。其中 `-deterministic` 用于可复现基准；它是否覆盖**游戏自身** RNG（蓝图 RandomStream、`FMath::Rand*`）随实现而异【待验证：在游戏代码里定位全部 RNG 入口，bot 启动时显式置 seed 并写进 log，不依赖该参数】。

### 7.3 命令行与环境变量读取（bot 侧）

```cpp
float Speed = 1.f;  FParse::Value(FCommandLine::Get(), TEXT("speed="), Speed);
bool  bAuto = FParse::Param(FCommandLine::Get(), TEXT("auto"));
if (FString Env = FPlatformMisc::GetEnvironmentVariable(TEXT("AG_SPEED")); !Env.IsEmpty())
    Speed = FCString::Atof(*Env);          // 环境变量与命令行同支持，multiverse 传参用 env
```

建议优先级：命令行 > 环境变量 > 默认值；解析结果全部打进 battle log（含 seed、speed、level），保证可复现。

### 7.4 自动开局

- 启动钩子：`UGameInstanceSubsystem::Initialize`（进程期一次）+ `FCoreUObjectDelegates::PostLoadMapWithWorld`（每次地图加载后）。
- 路径一（通用）：`UGameplayStatics::OpenLevel(World, FName("Level_1"), /*bAbsolute*/true, Options)`。
- 路径二（贴游戏）：反射调用主菜单 widget/GameInstance 的"开始游戏"函数（模式同 §4.4 UMG 直调）——游戏有前置流程（账号初始化、存档选择）时必须走这条，不然 OpenLevel 后又被菜单逻辑弹回去。
- fresh run：清掉存档里的续玩进度（引擎侧 `SaveGameToSlot` 的存档在 `<Saved>/SaveGames`；游戏自定义存档位置从代码确认），两局初始状态必须一致。

### 7.5 per-instance 存档隔离

查证结论：官方命令行参考中**不存在** `-saveddir` 之类的 Saved 目录重定向参数。现成的只有 `-SaveToUserDir`（"Save to user directory"，把存档从安装目录切到用户目录）。按可靠度排序的备选：

1. **容器隔离**（multiverse 模式天然成立）：每 allocation 一个容器，Saved 各自独立。
2. **`-SaveToUserDir` + 每实例 HOME/用户目录**：用户目录随 OS 用户环境走（Linux/macOS 看 HOME 类变量、Windows 看 LOCALAPPDATA 类变量），sweep 启动每个实例时注入不同的对应环境变量。【待验证：各平台"用户目录"确切落点不同——先单实例 `-log` 打印 `FPaths::ProjectSavedDir()` 确认，再改环境变量验证重定向是否生效】
3. **每实例拷贝安装目录**：大而笨但零风险——Saved 默认在安装目录下时，拷贝整包即可隔离。
4. 运行时改 `FPaths`：early 期重定向（PreLoadingScreen 阶段）理论可行【待验证】且侵入启动顺序，作为最后手段。

无论选哪条，验证方法统一：双实例并发各写一局存档，检查两份存档独立、无互相覆盖。游戏若把存档写进自定义路径（不走 `SaveGameToSlot`），从代码里找到实际路径再套同样思路。

### 7.6 进程退出与 Linux Dedicated Server

- 正常退出：终局判定成立后 `FPlatformMisc::RequestExit(false)`（触发退出委托、日志 flush）；`RequestExit(true)` 立即杀，sweep 强杀兜底之外的进程内兜底。5 秒内退不出就让 sweep 的墙钟超时强杀（sweep 侧职责）。
- 专用服务器目标（云端批量用）：

```csharp
// Source/GameServer.Target.cs —— TargetType.Server
public class GameServerTarget : TargetRules
{
    public GameServerTarget(TargetInfo T) : base(T)
    {
        Type = TargetType.Server;
        ExtraModuleNames.Add("MyGame");
    }
}
```

```bash
RunUAT.sh BuildCookRun -project="..." -platform=Linux -server -serverconfig=Development \
  -noclient -build -cook -stage -pak -package -archive -archivedirectory="..."
# 运行： GameServer.sh <MapName> -log -unattended
```

- **专用服务器没有 LocalPlayer**：§4 的 Enhanced Input/UMG 注入路径全部失效。bot 要么跑在客户端、要么在服务器侧直接驱动 server 逻辑（AI/反射直调），词汇表与序列 schema 对应的"玩家输入"原语在服务器上不存在——设计期就要决定 sweep 是 client 还是 server 形态。
- 交叉编译 Linux 需要对应工具链；先小规模试一次再进 sweep。

## 8. 已知坑与版本兼容

| 坑 | 症状 → 解法 |
|---|---|
| UE4 旧输入 vs UE5 Enhanced Input | 症状：注入后动作全无反应，代码里却找不到 `UEnhancedInputComponent`。→ 回 §1 按 `DefaultInput.ini` 的 `DefaultPlayerInputClass` 判定，不猜版本。两套并存（升级工程）时行为不可预测，注入路径必须与游戏实际绑定的那套一致；`APlayerController::InputKey/InputAxis` 在 UE5 已标记弃用，EI 工程下不保证触发 Action【待验证：个别版本它仍能喂给 UEnhancedPlayerInput（其派生自 UPlayerInput），用 `showdebug enhancedinput` 一局验证，别想当然】 |
| FSlateApplication 无头行为 | 症状：`-nullrhi` 下 Slate 合成事件没反应（或整局卡在 UI）。→ §4.3 的实测法先跑；不行就把"UI 交互"改为 UMG 逻辑直调（非等价模式），或放弃 `-nullrhi` 用 `-RenderOffScreen`。commandlet 进程无 Slate，合成事件路径整体不可用 |
| 裸指针缓存被 GC 清掉 | 症状：跑几分钟到几十分钟后随机崩溃/读到垃圾数据——UE 的 GC 会回收没有 UPROPERTY 引用的对象，`TActorIterator` 拿到的裸指针缓存不保护生命周期。→ 一律 `TWeakObjectPtr`，用前 `IsValid()`；需要强引用（bot 自有对象、RT、动作资产）才用 `UPROPERTY()`。对照 Unity 版"FindObjectsOfType 缓存"坑，这里是引擎级强敌，任何缓存指针都过一遍这个检查 |
| 蓝图工程要上 C++（bot 插件需要） | 解法：编辑器里新建任意 C++ 类，引擎自动生成 `Source/` 并把 .uproject 转 C++ 工程（要求本机装有匹配工具链）；Launcher 版引擎即可编译游戏侧 C++，不必源码版。生成后 bot 插件按 §2 模式 A 依赖游戏模块 |
| 反射字符串写错 | 症状：`FindPropertyByName`/`FindFunctionByName` 每次都 null 或读错。→ 属性名区分大小写且用引擎内拼写（蓝图变量改名后同步）；启动期加一条自检日志把所有引用到的名字打出来对照资产 |
| `showdebug` 系列不可用（Shipping） | 症状：Shipping 配置里控制台命令被裁剪。→ sweep 用 Development 配置构建；需要 Shipping 验收另走一轮 |
| Gauntlet：何时用官方框架 | 官方定位："run sessions of projects"，UAT 驱动、支持多平台矩阵（server+多 client、多地图冒烟、CI 回归），游戏侧零要求、TestController 可选。**比自写 sweep 合适**：多平台/多设备矩阵、纯断言式冒烟回归、不依赖外部视频分析。**不如自写**：你要 VLM 视频采集、每实例外置 battle log/sequence 文件收集、自定义启动参数透传、外部迭代闭环时——自写脚本对启动与产物的控制力远高于 Gauntlet 会话模型 |


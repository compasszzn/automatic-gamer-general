# 键鼠序列录制与回放实现参考

> 步骤3编写 bot 代码时，键鼠注入器（InputInjector）、序列录制器（InputRecorder）、序列回放器（SequenceReplayer）三个模块的实现参考。目标：让内挂 auto 模式的全部操作可被编码为键鼠序列 `sequence.json`，replay 模式按序列无决策驱动游戏，进而对比"内挂执行"与"序列回放执行"是否一一对应。
>
> 本文件定义**引擎无关**的规格：序列 schema、唯一注入口原则、录制规则、回放驱动设计、确定性清单、diff 方法。**各引擎的具体注入 API、帧计数器、`inject` 字段取值见对应引擎手册 §0/§4。**

---

## 核心思想：序列是内挂行为的无损编码

- 内挂（闭环）：读游戏状态 → 决策 → 注入输入
- 序列回放（开环）：不管状态，按时间表执行

**推论**：两局只要在任何一帧产生分歧（RNG、帧率、时间），闭环 bot 会自适应改变后续输入，开环回放不会——输入流从此分歧。所以**"一一对应是否成立"本质上是游戏确定性的试金石**，等价性验证因此同时是：注入层重构后的回归测试、操作审计、纯序列驱动的复现手段。

**前提约束（违反则等价性必然断裂）**：

1. 内挂全程只允许"键鼠可表达"的操作——每条策略规则必须映射到步骤1词汇表原语（见 SKILL.md 步骤2"键鼠原语"列）
2. 等价模式下阻塞 UI 走输入注入，禁用逻辑层直调（见 SKILL.md 3.2b 的冲突说明）
3. 所有输入事件走 InputInjector 唯一注入口，禁止散装调用引擎输入 API

---

## 序列文件格式（sequence.json）

**帧索引对齐，不用墙钟**——回放保真的第一关键。墙钟毫秒回放受帧率波动影响；帧索引回放逐帧对齐，与帧率解耦。

```json
{
  "version": 1,
  "meta": {
    "game": "GameName",
    "level": 3,
    "speed": 1,
    "seed": 12345,
    "recorded_by": "auto-mode",
    "recorded_at": "2026-08-31T12:00:00Z",
    "resolution": "512x512",
    "input_system": "<引擎与输入系统标识>"
  },
  "events": [
    {"frame": 30,    "t_unscaled": 3.02,  "device": "keyboard", "action": "press",   "key": "W"},
    {"frame": 90,    "t_unscaled": 9.11,  "device": "keyboard", "action": "release", "key": "W"},
    {"frame": 1204,  "t_unscaled": 12.83, "device": "mouse",    "action": "axis",    "vec": [0.71, -0.71]},
    {"frame": 1288,  "t_unscaled": 13.60, "device": "mouse",    "action": "warp",    "pos": [256, 480]},
    {"frame": 1288,  "t_unscaled": 13.60, "device": "mouse",    "action": "press",   "button": "left", "pos": [256, 480], "inject": "<注入方式标记>"},
    {"frame": 1290,  "t_unscaled": 13.75, "device": "mouse",    "action": "release", "button": "left", "pos": [256, 480], "inject": "<注入方式标记>"}
  ]
}
```

### 字段说明

| 字段 | 必填 | 说明 |
|------|:---:|------|
| `frame` | ✅ | 引擎帧计数器——回放的唯一时间基准（各引擎 API 见对应手册 §0 对照表） |
| `t_unscaled` | ✅ | 不受速度影响的真实时钟（见 SKILL.md 3.2c），诊断/展示用 |
| `device` | ✅ | `keyboard` / `mouse` |
| `action` | ✅ | `press` / `release` / `axis`（移动轴）/ `warp`（光标移位） |
| `key` / `button` | 按需 | 按键名 / 鼠标键（`left`/`right`/`middle`） |
| `vec` | axis | 归一化移动向量 |
| `pos` | mouse | 屏幕像素坐标（原点方位遵循引擎惯例，写入 meta） |
| `inject` | ✅ | 注入路径标记，取值由引擎决定（如 Unity 的 `queue_state_event` / `warp_cursor` / `execute_events`）。**回放必须用同一方式** |

### 录制规则

- **移动轴节流**：连续帧 `axis` 值相同时不重复记录，只在值变化时记录；回放时保持上次值（也可选"每 N 帧强制记录一次"防止浮点漂移判断过严）
- **单行紧凑 JSON**：与 battle log 同理，避免日志收集器摊平转义（见步骤5 cloud 模式坑）
- **原子写入**：每局结束先写临时文件再 rename（与 sweep 结果文件一致）
- **文件过大时分段写入**（见 SKILL.md Overview 注意事项）

---

## 录制实现（auto 模式）

### 唯一注入口

所有 bot 输入的唯一入口，录制/回放都挂在同一层。接口形态（实现语言随游戏）：

```text
InjectAxis(vec, injectMethod)                 // 移动轴
InjectKeyPress(key)                            // 按键按下
InjectKeyRelease(key)                          // 按键抬起
InjectMouseWarp(screenPos)                     // 光标移位
InjectMousePress(button, screenPos, injectMethod)
InjectMouseRelease(button, screenPos, injectMethod)
```

内部行为 = 执行引擎注入 + 通知录制器：

```text
// 伪代码：一次注入的固定结构（引擎具体注入 API 见对应手册 §4）
inject_to_engine(event)          // 例如 Unity: WarpCursorPosition + QueueStateEvent + Update

inputRecorder.record(
  frame:     engine_frame_count(),      // 例如 Unity: Time.frameCount
  t_unscaled: engine_real_time(),        // 例如 Unity: Time.unscaledTime
  device: "mouse", action: "press", button: button, pos: screenPos,
  inject:   injectMethod
)
```

**记录内容必须是"决策翻译后的玩家操作"**，不是内部决策值——这样序列才能脱离 bot 单独驱动游戏。

### UI 点击的两种路径与等价性

| 路径 | 序列中的形态 | 回放等价性 |
|------|-------------|:---:|
| 真实输入事件（光标移动 + 按下/抬起，走引擎 UI 命中检测） | `pos` + `inject: <真实事件标记>` | ✅ 与玩家操作同管线 |
| UI 控件事件直调（经引擎的 UI 事件派发机制直接调目标控件） | `target: "<UI 控件路径>"` + `inject: <直调标记>` | ⚠️ 回放必须用同一直调方式查找同一目标；改成"坐标+真实点击"不严格等价（有 UI 遮挡时行为不同） |

> 直调路径记录 UI 控件路径（而非 `pos`）；replay 模式下按路径重新查找目标。若目标路径在回放局不存在（UI 层级变化），立即报告分歧而不是静默跳过。

---

## 回放实现（replay 模式）

### 驱动器设计

```text
SequenceReplayer:
  nextEventIdx = 0
  每帧在"游戏逻辑之后、本帧结束前"的阶段执行：
    while nextEventIdx < events.length and events[nextEventIdx].frame <= 当前帧号:
      InjectEvent(events[nextEventIdx])   // 走同一个 InputInjector（不经录制器，或录制到 sequence_B 供对比）
      nextEventIdx += 1
```

要点：

- **逐帧追赶**：某帧事件多于一个时按序全部注入；若回放局帧号已越过下一个事件的 `frame` 超过阈值（如 >30 帧，说明回放局帧率异常低），记录警告——这是轨迹分歧的常见前兆
- **replay 模式同样保持 fresh run / 自动开局 / 终局退出**：SequenceReplayer 只替代"决策引擎"，工程框架（Bootstrap、AutoUIHandler 的非阻塞部分、退出逻辑）照常工作。注意自动开局本身不是序列的一部分（序列从对局开始帧起算），开局动作仍由工程框架完成
- **决策引擎必须完全禁用**：replay 模式下任何"顺手帮忙"的决策代码（自动选卡兜底等）都会污染对比
- **序列结束/游戏结束即退出进程**，与 auto 模式一致

### 命令行参数（透传给游戏，见 SKILL.md 3.3）

```
-run-mode replay -sequence <path.json> -seed <s> -level N -speed 1
-run-mode auto  -sequence <out.json> -record-sequence true -seed <s>
```

---

## 等价性验证流程（步骤6 的"序列等价性验证"环节）

```
Run A: -run-mode auto   -seed s -record-sequence true → sequence_A + battlelog_A
Run B: -run-mode replay -seed s -sequence sequence_A → battlelog_B + sequence_B（重录）
```

三层对比：

1. **输入流层**：`hash(sequence_B.events) == hash(sequence_A.events)`（比较 frame/device/action/key/button/pos/vec/inject 全字段）。录制/回放走同一注入口时构造上必然成立；不一致 → 注入路径混用或存在非键鼠可表达操作
2. **轨迹层**：battlelog 逐事件 diff + 周期状态快照（每 0.5s 真实时钟记一次 HP/位置/击杀数等 hash）→ 报告**首个分歧 frame**。首个分歧点之前两边完全一致，排查范围缩到该帧附近活跃的系统
3. **结果层**：胜负、用时、伤害来源分布是否一致

### diff 脚本模板

```python
import json, hashlib, sys

def event_sig(e):
    return json.dumps(e, sort_keys=True, ensure_ascii=False)

a = json.load(open(sys.argv[1]))["events"]
b = json.load(open(sys.argv[2]))["events"]

hash_a = hashlib.sha256("\n".join(map(event_sig, a)).encode()).hexdigest()
hash_b = hashlib.sha256("\n".join(map(event_sig, b)).encode()).hexdigest()
print("input-stream match:", hash_a == hash_b)

for i, (ea, eb) in enumerate(zip(a, b)):
    if event_sig(ea) != event_sig(eb):
        print(f"first event divergence at index {i}:")
        print("  A:", event_sig(ea)); print("  B:", event_sig(eb)); break
else:
    if len(a) != len(b):
        print(f"length mismatch: A={len(a)} B={len(b)}, first extra at {min(len(a),len(b))}")
```

battlelog diff 同理：按事件的 `(frame, type)` 索引对齐，报告首个不一致条目及两侧上下文（前后各 3 条），供"联合诊断"定位是哪个系统先偏离。

---

## 确定性清单（行为层对应的前提）

| # | 检查项 | 说明 |
|---|--------|------|
| 1 | 固定随机种子 | run 启动时初始化游戏内**所有**随机源（引擎 RNG 与语言级 RNG 都要），seed 走命令行参数并写入 battlelog |
| 2 | fresh run | 清存档续玩进度（关卡时间/经验/能力），两局初始状态一致（见 SKILL.md 3.2c） |
| 3 | 同 speed / 时间缩放 | speed 影响逻辑演化节奏，Run A/B 必须一致；注意 SKILL.md 3.2d 的"速度被 UI 复位"陷阱 |
| 4 | 逻辑在固定步长时钟 | 逻辑走固定步长（如引擎的物理 tick / fixed update）的游戏可严格复现；逻辑在可变帧率回调里的游戏只能近似对应——帧率波动造成物理/弹幕轨迹微小偏差，属预期内的"统计等价" |
| 5 | 真实时钟依赖 | 基于真实时钟的冷却/倒计时天然破坏对应（两局墙钟节奏不同）；此类逻辑存在时只在结果层要求一致 |
| 6 | 光标状态 | 光标被锁定/隐藏时光标移动注入可能无效，注入前检查并记录 |
| 7 | 运行方式 | 等价性验证用"有渲染的批处理模式"（Unity：`-batchmode` 不加 `-nographics`；其他引擎见手册 §6/§7）——保证 UI 命中检测与 UI 回调渲染副作用正常；与"需要 VLM 视频"的推荐模式一致 |
| 8 | 定时器/动画的时间基准 | 引擎的延迟/动画若基于受缩放时间，与 speed 一致即可复现；基于真实时钟的属于第 5 条 |
| 9 | 并发 | 等价性验证单实例跑（concurrency=1）；批量 sweep 中混入 replay 局时同样按单实例隔离存档（见步骤5） |

> 判定口径：**输入流层必须严格一致；轨迹层与结果层在满足清单 1-9 后应一致，可变帧率逻辑游戏允许帧级微小漂移，但胜负与关键事件序列必须一致。**

---

## 常见坑

- **回放用墙钟时间**：受帧率/加载卡顿影响逐帧漂移，最终面目全非——永远用帧索引
- **录制时记录了决策值而不是玩家操作**：如记录"移动意图向量"但实际注入的是包装后的输入管理器值——序列必须记录"实际写进输入系统的东西"
- **注入路径混用**：录的是 A 路径（如真实事件队列），回放却走了 UI 直调（或反过来）——`inject` 字段 + 回放侧断言
- **replay 局里决策引擎没禁干净**：某个兜底逻辑"帮忙"做了选择，序列里却没有对应事件 → 输入流对比反而抓不出来（因为重录序列与回放一致），必须靠轨迹层 diff 发现
- **开局差异**：自动开局的耗时/帧数两局不同 → 序列第 0 帧必须定义为"对局开始帧"（对局内帧计数器起点），而不是进程启动帧；开局前的事件（进主菜单、选关卡）由工程框架负责，不进序列
- **速度感知错位**：speed>1 时帧号与真实时间的对应关系变化——等价性验证统一用 speed=1（与 SKILL.md"调优/验证用 speed=1"一致）
- **cloud 模式回放**：序列随 battle log 一样可能被日志收集器摊平转义——抓取时兼容多种形态（见 `references/cloud-sweep-multiverse.md`），且 replay 局需要把序列文件传进云端实例（挂载或打包），等价性验证通常更适合 local 单实例跑

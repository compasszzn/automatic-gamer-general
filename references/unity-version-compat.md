# Unity 版本兼容性防护

不同 Unity 版本的 API 差异很大，bot 代码中使用的 API 必须做版本安全适配。

## 高频踩坑清单

| API | Unity 2021.3 | Unity 2022.2+ | 安全写法 |
|-----|-------------|---------------|---------|
| 对象搜索 | `FindObjectsOfType<T>()` | `FindObjectsByType<T>()` | 优先用 `FindObjectsOfType`（全版本兼容），或条件编译 |
| 内置字体 | `Resources.GetBuiltinResource<Font>("Arial.ttf")` | `LegacyRuntime.ttf` | 先试 Arial，再试 LegacyRuntime，都失败则跳过文字 |
| 字体创建 | 不存在 `Font.CreateDynamicFont` | 存在 | 不要用，用 fallback 链 |
| EventSystem | 需手动添加 `StandaloneInputModule` | 自动 | 检查 `FindObjectOfType<EventSystem>()` 为空时手动添加 |
| Camera.main | 可能为空（尤其在 batchmode） | 同左 | 永远做 null 检查 + fallback 创建 |
| `Application.isBatchMode` | 可用 | 可用 | 用于判断是否无头模式 |

## 必做检查

1. 在 `ProjectSettings/ProjectVersion.txt` 中确认 Unity 版本
2. 对所有 `FindObjectsOfType`、`Resources.GetBuiltinResource`、`Camera.main` 调用做 null 检查和 fallback
3. UI 创建代码用 try-catch 包裹（字体/Canvas 不可用时不应崩溃 bot）
4. 对 `FindObjectsOfType` 的结果做 `gameObject.activeSelf` 过滤（对象池中的 inactive 对象会被返回）
5. **每帧调用 `FindObjectsOfType` 会严重拖慢性能**——必须缓存结果，定时刷新（如 0.3-0.5 秒一次）

## 独立录制相机（必做）

VideoRecorder 绝不能劫持 `Camera.main.targetTexture`——这会导致主相机停止向屏幕渲染（画面冻结、UI 叠加）。

正确做法：
1. 创建独立的 GameObject + Camera
2. `recordCamera.CopyFrom(mainCam)` 复制主相机参数
3. 设置 `recordCamera.targetTexture = renderTexture`
4. `recordCamera.depth = mainCam.depth - 1`（在主相机前渲染）
5. 主相机的 `targetTexture` 永远不碰
6. 录制结束时销毁独立相机，不影响主相机

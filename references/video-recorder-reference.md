# 视频录制实现参考（通用模式 + 跨引擎对照）

> 步骤3视频录制器模块的实现参考。通用模式：**每帧从游戏画面采集原始像素（RGBA）→ pipe 到 ffmpeg 子进程 stdin → 实时编码 mp4**，不做中间 PNG。各引擎"怎么拿到帧"不同，见对照表；引擎专属细节与完整实现见对应引擎手册 §6。

---

## 通用实现模式

- **采集**：每帧按 video-fps 间隔取一帧原始像素（RGBA 排布）。采集必须与游戏主循环同帧触发，否则丢帧/重帧
- **pipe**：把 RGBA 原始数据写入 ffmpeg 子进程 stdin，实时编码为 mp4

### ffmpeg 启动命令（通用）

```
ffmpeg -y -f rawvideo -vcodec rawvideo -pixel_format rgba
  -colorspace bt709 -video_size {width}x{height}
  -framerate {fps} -loglevel warning -i -
  -c:v libx264 -pix_fmt yuv420p -crf 23 {outputPath}
```

---

## 跨引擎帧采集方式对照表

| 技术栈 | 帧采集方式 | 关键注意事项 | 详细实现 |
|--------|-----------|-------------|-----------|
| Unity | 专用录制 Camera → RenderTexture → ReadPixels → GetRawTextureData | 绝不劫持主相机 targetTexture（画面冻结）；`-nographics` 黑帧 | `engine-unity.md` §6 |
| Unreal | SceneCapture2D / 专用相机渲染到 render target → ReadPixels | `-nullrhi` 无渲染黑帧；要视频必须有真实 RHI | `engine-unreal.md` §6 |
| Godot | ① 内建 Movie Maker（`--write-movie`）② SubViewport + get_texture().get_image() | `--headless`（dummy 渲染驱动）下无法截帧；要视频需真实渲染 | `engine-godot.md` §6 |
| Web | ① Playwright recordVideo（webm → ffmpeg 转 mp4）② CDP Page.startScreencast ③ captureStream + MediaRecorder | headless 渲染可用；注意 GPU 禁用时 WebGL 的黑屏风险 | `engine-web.md` §6 |
| 原生/小引擎 | 引擎截屏/表面读取 API（pygame surfarray、LÖVE screenshot 等）；引擎无 API 时用 OS 级录屏兜底（如 ffmpeg x11grab） | 采集点必须挂在游戏主循环内；OS 级录屏受窗口遮挡影响 | `engine-native.md` 各框架节 |

> **共同铁律**：纯无图形模式（GPU 渲染关闭）下录制必为黑帧——需要 VLM 可用视频时，必须用"有渲染但无窗口"的运行模式（各引擎支持程度见手册 §6/§7）。

---

## 录制触发时机

- `StartRecording()`：Auto 模式开启时调用
- `CaptureFrame()`：每帧调用（按 video-fps 间隔控制）
- `StopRecording()`：单局结束时调用（通关/死亡/时间限制）

## 产出文件

| 文件 | 说明 |
|------|------|
| `recording.mp4` | 完整单局视频 |
| `frame_data.json` | 每帧时间戳、bot 位置、关键事件标记 |

## 命令行参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `record-video` | 是否录制视频 | true |
| `video-fps` | 帧率 | 10 |
| `video-resolution` | 分辨率（WxH） | 512x512 |

## ffmpeg 依赖

ffmpeg 必须在系统 PATH 中可用。如果不可用，自动安装：
- macOS: `brew install ffmpeg`
- Linux (Debian/Ubuntu): `sudo apt install ffmpeg`
- Linux (RHEL/CentOS): `sudo yum install ffmpeg`
- Windows: `winget install ffmpeg`

run_sweep 脚本启动时应检查 ffmpeg 是否可用并自动安装。

## 无头/无窗口模式的结论（通用）

- **纯无图形模式**：GPU 渲染关闭 → 录制必为黑帧 → 只适合纯 log 迭代
- **有渲染的批处理模式**（无窗口）：画面可采帧 → VLM 视频可用的最低配置
- **带窗口模式**：适合排查 bot 行为，肉眼观察 + 真实视频
- 三层推荐模式与 SKILL.md 3.2c 的"推荐模式"一一对应

## frame_data.json 结构

```json
{
  "frames": [
    {
      "frame_index": 0,
      "timestamp_ms": 0,
      "bot_position": {"x": 0, "y": 0},
      "key_event": "level_start"
    },
    {
      "frame_index": 100,
      "timestamp_ms": 10000,
      "bot_position": {"x": 12.3, "y": -5.7},
      "key_event": null
    },
    {
      "frame_index": 970,
      "timestamp_ms": 97000,
      "bot_position": {"x": 45, "y": 30},
      "key_event": "level_end"
    }
  ]
}
```

只在关键事件发生时记录 key_event，大多数帧的 key_event 为 null。`timestamp_ms` 基于不受 speed 缩放影响的真实时钟（见 SKILL.md 3.2c）。

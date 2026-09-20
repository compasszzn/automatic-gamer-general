# VLM 视频分析原则

> 步骤6迭代循环中视频分析的设计原则。本文件定义分析的原则和方向，不提供固定 prompt 模板——每轮的 VLM 分析 prompt 由执行 skill 的 LLM 根据当前上下文自行撰写。

---

## 核心原则：模型自行决定分析 prompt

执行 skill 的 LLM 拥有完整上下文：步骤1分析文档、步骤2策略设计、当前 bot 代码、本轮已知问题。基于这些信息，LLM 自行撰写发给 VLM 的分析 prompt。

---

## VLM 模型与技术实现

使用 **qwen3.6-plus** 通过 dashscope OpenAI 兼容接口进行视频分析：

```python
from openai import OpenAI
import base64, os

client = OpenAI(
    api_key=os.environ["DASHSCOPE_API_KEY"],
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
)

# 视频文件读取
with open(video_path, "rb") as f:
    video_b64 = base64.b64encode(f.read()).decode("utf-8")

# 发送视频给 VLM 分析
response = client.chat.completions.create(
    model="qwen3.6-plus",
    messages=[{
        "role": "user",
        "content": [
            {"type": "video_url", "video_url": {"url": f"data:video/mp4;base64,{video_b64}", "fps": 2}},
            {"type": "text", "text": "<LLM自行撰写的分析prompt>"}
        ]
    }]
)
```

**fps 参数**：控制 VLM 抽帧密度。建议：
- 短视频（<30秒）：fps=2（保留完整节奏）
- 中等视频（30-120秒）：fps=1（平衡细节和 token）
- 长视频（>120秒）：fps=0.5（减少 token 消耗，仍保留完整叙事）

---

## 分析方向（不是固定 prompt，而是 prompt 应覆盖的方向）

LLM 在撰写 VLM 分析 prompt 时，应考虑以下方向（按需要选择，不必全部包含）：

1. **策略执行流分析**：从开始到结束，每阶段 bot 在做什么、为什么做
2. **策略节奏感**：哪些时段 bot 在推进、哪些在犹豫、哪些在被动应对
3. **失败/成功的完整叙事**：追溯整个过程怎么走到这个结局的
4. **被浪费的机会**：哪些时刻 bot 本可以做出更好的反应但策略没覆盖
5. **画面外状态推断**：根据 bot 的行为推断可能的内部状态（HP、冷却等）
6. **通关局 vs 失败局对比**（Phase 2 时）：关键差异在哪里，失败局从什么时刻开始偏离
7. **特定问题追踪**：针对本轮已知问题聚焦观察

---

## prompt 撰写建议

- prompt 应聚焦**策略执行诊断**，不是视觉描述（"画面上有个红色圆点"没有诊断价值）
- prompt 应针对**当前诊断目标**（如果本轮已知问题是闪避决策，prompt 应聚焦闪避相关的观察）
- prompt 应要求 VLM 按**时间段**组织分析（"第0-5秒做什么、第5-15秒做什么"），不是碎片化描述
- Phase 1 的 prompt 倾向"完整策略评估"
- Phase 2 的 prompt 倾向"对比分析"或"特定问题追踪"

---

## VLM 分析结果与 battle log 的合并

VLM 分析结果提供**视觉洞察**（策略在画面上的表现），battle log 提供**精确数据**（具体数值和决策推理）。两者合并为综合诊断：

- VLM 说"bot 在第30秒被两个远程敌人夹击在狭窄通道中无法闪避"
- log 说"第30秒 dodge_decision: chosen_action=继续攻击, reason=近战距离更近(效用0.7) > 远程距离较远(效用0.3)"
- 合并诊断："效用评分只考虑了距离而忽略了通道限制，需要在狭窄地形中对远程威胁加权"

这种合并才能给出精准可执行的代码改动方向。

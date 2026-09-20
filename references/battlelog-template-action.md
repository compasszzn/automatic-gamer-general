# 战斗 Log 模板 — 动作/割草类深度层（简化版）

> 适用于：Vampire Survivors 类、Musou 类、横向/纵向割草、任何"大量敌人涌来、玩家以攻击/闪避应对"的动作游戏
> 本层扩展通用基础层（`battlelog-template-common.md`）的字段，追加割草类特有的 decision_point 和关键转折识别规则，以及 summary 中的数值字段。
>
> **简化原则**：视频录制覆盖了画面叙事（敌人出现、bot闪避、波次切换等视觉事件）。本层只定义视频无法表达的：决策推理过程、关键转折判定规则、精确数值汇总。

---

## decision_point 扩展

通用基础层未定义具体 decision_point。割草类定义以下决策点类型（只在这些决策发生时记录，不是每 tick）：

| decision_point | 说明 | 记录时机 |
|---------------|------|---------|
| target_selection | 选择攻击哪个目标 | bot 需从多个敌人中选择优先攻击目标时 |
| dodge_decision | 是否执行闪避 | 出现需要闪避的威胁时，bot 决定闪避还是走位还是继续攻击 |
| aoe_decision | 是否使用AoE | 敌人密度达到需要清场时 |
| position_decision | 站位移动选择 | bot 决定移向哪个方向/位置时 |
| pickup_decision | 是否拾取道具 | 地面有道具时 |
| upgrade_choice | 升级选择 | 升级出现时选择哪个奖励 |
| boss_strategy_switch | Boss策略切换 | Boss出现时从清场模式切换到Boss对抗模式 |

---

## critical_turns 识别规则

在割草类游戏中，以下情况应标记为关键转折：

| 规则 | 说明 |
|------|------|
| HP暴跌 | 连续短时间内（如5秒内）HP下降超过40% |
| 闪避冷却期被攻击 | bot 在闪避冷却期间被致命技能命中 |
| Boss必躲技能命中 | bot 未能躲避Boss的秒杀级技能 |
| 自爆类连锁触发 | 自爆怪连锁爆炸导致大面积伤害 |
| 治疗类未被击杀 | 治疗类敌人存活时间过长 |
| 升级关键选择失误 | 选择了明显不利于当前局势的升级项 |
| 拾取道具导致被击 | 为了拾取道具走入危险区域导致重伤或死亡 |

---

## summary 扩展

割草类游戏的 summary 包含视频无法展示的精确数值：

```json
{
  "result": "lose",
  "duration_seconds": 182,
  "final_player_hp": 0,
  "final_player_hp_max": 100,
  "progress": {
    "primary": 4,
    "primary_max": 28,
    "throughput": {"kills": 73, "kills_per_min": 24.1, "xp_level": 5},
    "notes": "卡在第4房间第2波，被自爆精英连锁炸死"
  },
  "upgrade_choices": [
    {"tick": 65, "choice": "火焰AoE", "alternatives": ["冰冻减速", "攻击速度提升"]}
  ],
  "damage_taken": {
    "total": 100,
    "by_source": {
      "melee_collision": 30,
      "ranged_projectile": 10,
      "elite_explosion": 45,
      "boss_skill": 0,
      "other": 15
    }
  },
  "damage_dealt": {
    "total": 280,
    "by_target_type": {
      "melee_basic": 200,
      "ranged_shooter": 60,
      "elite_bomber": 20,
      "boss": 0
    }
  }
}
```

| 字段 | 说明 | 为什么视频不能替代 |
|------|------|-------------------|
| progress.primary / primary_max | 已清房间/波次数 vs 通关所需（割草类的进度梯度） | win/lose 二值无梯度；这个连续指标才能判断改动是否朝通关推进 |
| progress.throughput | 击杀数 / 击杀每分 / XP等级（吞吐与成长效率） | "存活久"≠"接近通关"；吞吐才反映是否真在推进，而非原地苟活 |
| damage_taken.by_source | 各类型敌人的伤害分布 | 视频能看到"被打"，但看不到具体数字和来源分类 |
| damage_dealt.by_target_type | 对各类型敌人的伤害分布 | 视频能看到"在攻击"，但看不到具体数值 |
| upgrade_choices | 升级选择记录 | 视频能看到升级选择画面，但看不到所有备选选项 |
| final_player_hp / hp_max | 最终HP和最大HP | 视频可能看不到HP条上的精确数字 |

---

## 简化示例

以下是一局割草类游戏的简化战斗 log 示例：

```json
{
  "timestamp": "2026-05-28T15:30:00Z",
  "level": 3,
  "result": "lose",
  "duration_seconds": 182,

  "decision_trace": [
    {
      "tick": 47,
      "decision_point": "dodge_decision",
      "situation": "5个近战小怪逼近碰撞距离<2",
      "chosen_action": "向右侧闪避",
      "alternatives": ["继续攻击最近小怪", "向上方走位"],
      "reason": "碰撞伤害累积快，闪避一次性脱离近战范围。效用评分：闪避 0.85 > 走位 0.6 > 继续攻击 0.3",
      "outcome": "成功躲开碰撞，进入闪避冷却1.5秒"
    },
    {
      "tick": 120,
      "decision_point": "target_selection",
      "situation": "2个自爆精英在3.5距离，12个近战小怪在攻击范围",
      "chosen_action": "继续攻击近战小怪",
      "alternatives": ["优先攻击自爆精英", "使用AoE清场"],
      "reason": "近战小怪距离更近正在造成碰撞伤害。效用评分：近战小怪威胁 0.7 > 自爆精英威胁 0.5（距离更远）",
      "outcome": "自爆精英逼近至1.5距离后连锁爆炸"
    }
  ],

  "critical_turns": [
    {
      "tick": 120,
      "description": "自爆精英出现时优先攻击了近战小怪而非自爆精英",
      "impact": "连锁爆炸导致HP从72降至18（54%损失）",
      "bot_action_at_turn": "继续攻击近战小怪",
      "suggested_improvement": "自爆精英出现时应立即切换优先级为击杀自爆精英"
    }
  ],

  "summary": {
    "result": "lose",
    "duration_seconds": 182,
    "final_player_hp": 0,
    "final_player_hp_max": 100,
    "upgrade_choices": [
      {"tick": 65, "choice": "火焰AoE", "alternatives": ["冰冻减速", "攻击速度提升"]}
    ],
    "damage_taken": {
      "total": 100,
      "by_source": {"melee_collision": 30, "ranged_projectile": 10, "elite_explosion": 45, "boss_skill": 0, "other": 15}
    },
    "damage_dealt": {
      "total": 280,
      "by_target_type": {"melee_basic": 200, "ranged_shooter": 60, "elite_bomber": 20, "boss": 0}
    }
  }
}
```

从这个简化示例可以看出：视频提供完整的视觉叙事（bot怎么移动、什么时候闪避、画面上的敌人分布），而 log 提供视频看不到的精确数据（效用评分、伤害数值分布、升级备选项）。两者组合才能给出精准优化。

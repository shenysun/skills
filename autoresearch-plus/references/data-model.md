# 数据模型说明

本文档定义 autoresearch-plus 所有输出文件的完整格式。

---

## results.json 完整格式

> ⚠️ 字段名是仪表板硬契约，逐字一致才能渲染（漂移即静默空白）。注意：`pass_rate`≠`percentage`，`status`≠`kept`，`eval_breakdown` 是**数组**非对象，顶层需有 `baseline_score`/`best_score`/`status`。

```json
{
  "skill_name": "your-skill",
  "status": "running",
  "current_experiment": 0,
  "baseline_score": 88.0,
  "best_score": 88.0,
  "experiments": [
    {
      "id": 0,
      "score": 22,
      "max_score": 25,
      "pass_rate": 88.0,
      "status": "baseline",
      "description": "[第一轮] 原始 SKILL.md — 未修改"
    }
  ],
  "eval_breakdown": [
    { "name": "E1: 参数构造正确", "pass_count": 5, "total": 5 },
    { "name": "E2: 成功时提供链接和摘要", "pass_count": 4, "total": 5 }
  ]
}
```

### 字段说明

| 字段 | 说明 |
|------|------|
| `skill_name` | skill 名称 |
| `status` | `"running"` / `"complete"` |
| `current_experiment` | 当前最新实验 ID |
| `baseline_score` | 第一轮第一个基线的通过率 % |
| `best_score` | 所有实验中最高通过率 % |
| `experiments[].id` | 实验序号（全局递增，跨轮次不重置） |
| `experiments[].score` | 该实验原始得分 |
| `experiments[].max_score` | 该实验满分（轮次变化时会变） |
| `experiments[].pass_rate` | score/max_score × 100 |
| `experiments[].status` | `"baseline"` / `"keep"` / `"discard"` |
| `experiments[].description` | 格式：`[第N轮] 变更描述` |
| `eval_breakdown[].name` | eval 名称，格式：`EX: 简短名` |
| `eval_breakdown[].pass_count` | 所有实验中该 eval 通过的累计次数 |
| `eval_breakdown[].total` | 该 eval 被考察的累计次数 |

> `eval_breakdown` 中每条 eval 的 `pass_count` = 所有实验中该 eval 通过的总次数，`total` = 所有场景中该 eval 被考察的总次数（排除不适用的场景）。

---

## results.tsv 格式（制表符分隔）

```
experiment	score	max_score	pass_rate	status	description
0	22	25	88.0%	baseline	[第一轮] 原始 SKILL.md — 未修改
1	23	25	92.0%	keep	[第一轮] 添加具体指令 X
2	23	25	92.0%	discard	[第一轮] 尝试修改 Y — 无改善
```

---

## 仪表板自动计算的指标

| 指标 | 计算方式 |
|------|---------|
| 轮次提升幅度 | 轮次最终 pass_rate − 轮次基线 pass_rate |
| 轮次效率 | 提升幅度 / 该轮实验总数 |
| 最大单次跳升 | 所有 keep 实验中 delta 最大值 |
| 整体保留率 | keep 数 / (keep + discard) 数 |
| 单次平均有效提升 | 所有正 delta 之和 / keep 总数 |

---

## eval-suite.md 格式

```markdown
# Eval Suite — [skill-name]

## 第一轮（满分 = N 场景 × M eval）

### 测试场景

| ID | 场景描述 | 关键考察点 |
|----|---------|-----------|
| A  | ...     | ...       |

### Eval 标准（E1–EM）

| ID | 问题 | 通过条件 | 失败条件 |
|----|------|---------|---------|
| E1 | ...  | ...     | ...     |

### 最终得分汇总

| 轮次 | 满分 | 基线 | 最终 | 提升 |
|------|------|------|------|------|
| 第一轮 | N | — | — | — |
```

每轮结束后，在 `eval-suite.md` 底部追加新轮次章节，并填写该轮最终得分汇总。

---

## changelog.md 格式

```markdown
## 实验 N — 保留 ✓ / 丢弃 ✗

**得分:** X/Y (Z%) → 若保留则标注 +Δ 分
**变更:** 一句话描述改了什么
**理由:** 为何预期这个改动有效
**结果:** 实际发生了什么 — 哪些 eval 改善/下降
**剩余失败:** 当前还有哪些场景/eval 在失败（如有）
```

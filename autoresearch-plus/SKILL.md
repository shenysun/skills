---
name: autoresearch-plus
description: "多轮自动 skill 优化器，带逐 eval 通过率追踪、轮次效率指标和实时仪表板。每轮满分后升级到更难的下一轮，防止过拟合。使用场景：优化这个 skill、运行 autoresearch-plus、深度优化、多轮 eval、benchmark。输出：改进后的 SKILL.md、eval-suite.md、results.json、results.tsv、changelog.md、dashboard.html。"
---

# Autoresearch Plus

多轮递进 eval 套件 + 逐 eval 通过率追踪 + 轮次效率指标 + 实时仪表板。

循环结构：每轮满分后升级到更难的下一轮，防止在固定测试集上过拟合。

```
第一轮（基础场景）→ 基线 → 逐步修复 → 100%
第二轮（边缘场景 + 更多 eval）→ 基线 → 逐步修复 → 100%
第三轮（深度边界场景）→ ...（用户决定停止）
```

---

## 开始前：收集上下文

**STOP — 在用户确认以下所有字段之前，不得运行任何实验。**

1. **目标 skill** — 需要优化的 SKILL.md 完整路径
2. **第一轮测试场景（3–5 个）** — 覆盖最核心的使用场景，尽量多样
3. **第一轮 eval 标准（3–5 条）** — 二元通过/失败（见下文规则）
4. **每轮实验次数** — 默认 5 次
5. **预算上限（可选）** — 最大实验循环次数，默认无上限

> 不需要预先定义所有轮次。每当一轮达到满分，再和用户协商下一轮。

---

## 步骤 1：读懂 skill

完整阅读目标 skill：SKILL.md + `references/` 中所有被引用的文件。识别核心任务、步骤、输出格式，记下已有的质量检查或反模式说明。**绝对不能跳过。**

---

## 步骤 2：构建 eval 套件并写入 eval-suite.md

将 eval 标准结构化。**每条 eval 必须是二元判断 — 通过或失败，没有量表。**

好 eval 的规则：
- 二元。是或否。不要「评分 1–7」。
- 具体到能保持一致。「输出可读吗？」太模糊；「所有单词拼写正确且句子完整吗？」可测试。
- 不能太窄，否则 skill 会只针对该 eval 优化，其他方面退化。
- 每轮 3–6 条是甜点区。

写 eval 前先用 **3 问验证法**（来自 [`references/eval-guide.md`](references/eval-guide.md)）：
1. 两个不同的 agent 对同一输出能得出一致结论吗？→ 否则太主观
2. skill 能在不真正改进的情况下「游玩」这条 eval 吗？→ 否则太窄
3. 这条 eval 测的是用户真正在乎的东西吗？→ 否则直接删掉

> 各 skill 类型（文本、视觉、代码、文档）的好/坏 eval 示例和常见错误，见 [`references/eval-guide.md`](references/eval-guide.md)。

**写入 eval-suite.md** — 完整格式见 [`references/data-model.md`](references/data-model.md)。

---

## 步骤 3：部署实时仪表板

**在运行任何实验之前**，将模板复制到工作目录，**通过 HTTP 服务**打开（不要用 `file://`——仪表板靠 `fetch` 拉数据，`file://` 被同源策略拦截后会静默空白）：

```bash
cp "$(dirname $0)/../assets/template.html" autoresearch-[skill-name]/dashboard.html
# 把 dashboard.html 里的 {{SKILL_NAME}} 替换为实际名称
cd autoresearch-[skill-name] && (python3 -m http.server 8777 >/dev/null 2>&1 &)
open http://localhost:8777/dashboard.html
```

仪表板每 10 秒轮询 `results.json`，含 KPI、得分趋势、Eval 通过率、各轮效率、洞察、实验表。

**升级到新轮次时**，在 `dashboard.html` 顶部的 `ROUND CONFIG` 区域更新以下内容（见 [`references/dashboard-spec.md`](references/dashboard-spec.md) 详细说明）：
- `ROUNDS` 数组追加新轮次
- `getRound()` 函数更新边界值
- `areaColors` / `areaLabelColors` 追加颜色
- 测试套件区域追加新轮次的 `<details class="sr">` 折叠块（块内有注释模板）

---

## 步骤 4：建立基线

这是实验 #0，在修改任何内容之前运行。

1. 创建工作目录：`autoresearch-[skill-name]/`（在 skill 目录内）
2. 创建 `results.tsv`（含表头行）、`results.json`、`eval-suite.md`、`dashboard.html`，用 HTTP 服务打开（非 file://）
3. 备份原始 SKILL.md 为 `SKILL.md.baseline`
4. 用测试场景模拟运行 skill，逐一评估每条 eval
5. 汇总得分，更新 results.tsv 和 results.json
6. **保存每个场景的 skill 产物到 `outputs/` 目录**（见下方「产物保存规则」）

`results.json` 完整格式见 [`references/data-model.md`](references/data-model.md)。字段名是仪表板硬契约，漂移即静默空白——务必用 `pass_rate`（非 `percentage`）、`status`（非 `kept`）、数组式 `eval_breakdown`，并写完后 `json.load` 验证一次。

**重要：** 建立基线后告知用户得分。若基线已 ≥90%，询问是否需要继续优化。

### 产物保存规则

**每次实验评估（包括基线）都必须将 skill 的实际生成产物保存下来**，不只是评分。产物是最有价值的交付物——用户要看的是「图长什么样」「代码跑出来什么样」，不是分数。

规则：
- 在工作目录下创建 `outputs/` 子目录
- 每个测试场景生成一个独立文件，命名格式：`[场景ID]-[简短描述].html`
- 产物必须是基于当前 SKILL.md **实际执行** skill 后得到的真实输出，不是理论评估
- 如果 skill 生成代码（而非最终产物），需要将代码组装为可直接在浏览器打开的完整 HTML
- 产物文件需可在浏览器中直接打开并正确渲染

```
autoresearch-[skill-name]/outputs/
├── A-三角形外接圆.html
├── B-二次函数图象.html
├── ...
```

保留实验的产物会覆盖之前同名文件（保持 outputs/ 始终是最新最佳状态）。丢弃实验不保存产物。

---

## 步骤 5：运行实验循环

一旦开始，自主运行直到停止信号。

**每次迭代：**

1. **分析失败。** 哪条 eval 失败最多？失败的场景有什么共同模式？

2. **形成假设。** 选择**一件事**改变。好的 mutation：
   - 针对最常见失败添加一条具体指令
   - 将模糊指令改写得更明确
   - 添加反模式（「不要做 X」）修复反复出现的错误
   - 将被埋没的重要指令移到更靠前的位置
   - 添加或改进一个示例，展示正确行为
   - 删除导致过度优化某一方面的指令

   坏的 mutation：从头重写、一次添加 10 条规则、没有具体理由让 skill 变长、加「做得更好」这类模糊指令。

3. **应用修改。** 编辑 SKILL.md，**只改一处**。

4. **运行实验。** 用相同的测试场景逐一评估所有 eval。

5. **打分。** 计算总得分。更新 `eval_breakdown` 中每条 eval 的 `pass_count`。

6. **决定：保留或丢弃。**
   - 得分提升 → **保留**，这是新基线，**将本轮产物保存到 `outputs/`**（覆盖旧文件）
   - 得分持平或下降 → **丢弃**，还原 SKILL.md，不保存产物

7. **记录结果**（results.tsv + results.json + changelog.md）。

8. **重复。**

**停止条件（以下任一触发）：**
- 用户手动停止
- 达到预算上限
- 当前轮 100% 通过率 → 提示用户确认是否升级到下一轮
- 连续 3 次实验得分无变化且没有新的假设 → 报告停滞，询问是否终止

**思路耗尽时：** 重读失败输出。尝试将两次接近成功的 mutation 结合。尝试删除内容而非添加。**维持得分的简化也是胜利。**

---

## 步骤 5b：升级到下一轮

当一轮达到满分（100%）时：

1. 向用户报告该轮完成情况（基线→最终，实验数，主要改进点）
2. 和用户协商下一轮的新场景（3–8 个，比上一轮更难）和新 eval（3–6 条）
3. 将新轮次添加到 eval-suite.md
4. 更新仪表板（见 `references/dashboard-spec.md` 的「升级到下一轮时的更新清单」）
5. 运行新轮次的基线实验（description 格式：`[第N轮] 原始状态基线`）
6. 继续实验循环

> 轮次递进策略：第一轮测基础流程，第二轮测边缘输入和参数推断，第三轮测友好降级，第四轮及以后测深层追问、批量操作、超出数据范围的兜底。

---

## 步骤 6：写 changelog

每次实验后追加到 `changelog.md`。格式见 [`references/data-model.md`](references/data-model.md)。

changelog 是最有价值的产出。它是一份研究日志，未来的 agent 可以直接接手继续。

---

## 步骤 7：交付结果

循环结束时汇报：

1. **得分汇总（按轮次）：** 每轮基线→最终（含提升 %）
2. **总实验数 / 保留 / 丢弃 / 整体保留率**
3. **最高效轮次：** 提升幅度/实验数比值最高的轮次
4. **Top 3 最有效变更**（来自 changelog）
5. **各 eval 最终通过率** — 哪条 eval 最难通过
6. **剩余失败模式**（如果没有达到 100%）
7. 改进后的 SKILL.md 已原地保存
8. 所有文件位置

---

## 输出文件

```
autoresearch-[skill-name]/
├── dashboard.html       # 实时浏览器仪表板（10秒自动刷新）
├── results.json         # 驱动仪表板的数据文件
├── results.tsv          # 每次实验的得分日志（制表符分隔）
├── changelog.md         # 详细 mutation 日志
├── eval-suite.md        # 所有轮次的场景和 eval 标准（持久存档）
├── SKILL.md.baseline    # 优化前的原始 skill
└── outputs/             # 每个场景的 skill 实际生成产物（浏览器可打开）
    ├── A-场景描述.html
    ├── B-场景描述.html
    └── ...
```

改进后的 SKILL.md 保存回原始位置。

---

## 测试一次好的 autoresearch-plus 运行

1. **建立了真实基线** — 没有在测量起点之前修改任何内容
2. **每轮有独立的 eval 套件** — 不同轮次考察不同维度，不重复
3. **eval 严格二元** — 没有量表，没有「感觉」
4. **每次只改一件事** — 改动与结果之间有因果关系
5. **完整记录了每次实验** — 包括失败的 mutation
6. **eval_breakdown 被持续更新** — 能看出哪条 eval 是「最后一道关」
7. **轮次递进有意义** — 每轮确实比上一轮更难
8. **changelog 可以被另一个 agent 接手** — 包含足够的上下文和失败分析
9. **每次实验都保存了 skill 产物** — outputs/ 目录中每个场景都有浏览器可打开的真实产物
10. **仪表板真的能显示数据** — HTTP 打开（非 file://），results.json 字段名严守 data-model schema

> 如果 skill 通过了所有 eval 但实际输出质量没有提升 — **是 eval 的问题，不是 skill 的问题。** 回到步骤 2，写更好的 eval。

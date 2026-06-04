# 仪表板模板使用指南

模板文件：`assets/template.html`（完整可运行的单文件 HTML）

---

## 初始部署

1. 将 `assets/template.html` 复制到 `autoresearch-[skill-name]/dashboard.html`
2. 将文件中的 `{{SKILL_NAME}}` 替换为实际 skill 名称（共 2 处：`<title>` 和 JS 常量）
3. 用浏览器打开，仪表板每 10 秒轮询同目录的 `results.json`

---

## 升级到下一轮时的更新清单

找到 dashboard.html 中的 `// ===================== ROUND CONFIG` 区域，更新以下内容：

### 1. ROUNDS 数组

```js
const ROUNDS = [
  {label:'第一轮', cls:'r1', base:0},
  {label:'第二轮', cls:'r2', base:5},   // ← 新增，base = 第一轮最后一个实验的 id
];
```

### 2. getRound() 函数

```js
// 更新边界：id <= 第N轮最后一个实验id → 返回 N-1
function getRound(id) {
  return id <= 5 ? 0 : 1;  // 示例：第一轮实验 0-5，第二轮从 6 开始
}
```

### 3. areaColors / areaLabelColors（ECharts 背景区域）

```js
const areaColors      = ['rgba(0,56,255,.06)', 'rgba(232,40,26,.06)'];
const areaLabelColors = ['rgba(0,56,255,.3)',  'rgba(232,40,26,.3)'];
```

### 4. 测试套件折叠块

在 `#suite-body` 内追加（模板注释中有完整示例）：

```html
<details class="sr">
  <summary>
    <span class="sr-num r2">02</span>
    第二轮 — 边缘场景
    <span class="sr-meta">5 场景 × 4 eval · 基线 X% → 最终 Y%</span>
  </summary>
  <div class="sr-body">
    <!-- 左列：测试场景表 / 右列：Eval 标准表 -->
  </div>
</details>
```

---

## 预设轮次颜色

| 轮次 | cls | CSS 变量 | 十六进制 | areaColor |
|------|-----|---------|----------|-----------|
| r1 | r1 | `--blue`   | `#0038ff` | `rgba(0,56,255,.06)` |
| r2 | r2 | `--red`    | `#e8281a` | `rgba(232,40,26,.06)` |
| r3 | r3 | `--green`  | `#007a38` | `rgba(0,122,56,.06)` |
| r4 | r4 | `--purple` | `#5b21b6` | `rgba(91,33,182,.06)` |
| r5 | r5 | `--teal`   | `#0891b2` | `rgba(8,145,178,.06)` |
| r6+ | — | 自选 | 与现有颜色有足够区分度 | 对应 rgba |

r1–r5 的 CSS 颜色类已内置于模板，r6+ 需在 `<style>` 中手动追加：

```css
.ms-round-label.r6{color:#xxx} .rs-label.r6{color:#xxx}
.rs-bar-fill.r6{background:#xxx} .rsh-num.r6{color:#xxx} .sr-num.r6{color:#xxx}
```

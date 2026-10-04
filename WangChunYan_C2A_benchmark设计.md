# MetaBoundary 技术规格 / Benchmark Technical Specification

**作者 / Author:** WangChunYan ｜ **赛道 / Track:** Track 2 — Metacognition
**配套文件:** `WangChunYan_C2A_proposal.md`

> 本文件把提案中的设计落地为可实现的工程规格：数据格式、评分函数、题库生成器、防作弊与信度/效度论证。

---

## 1. 数据集结构 / Data Structure

每个测试实例为一个 JSON 对象：

```json
{
  "item_id": "meta-08213",
  "category": "false_premise",          // answerable | false_premise | no_solution | underspecified | nonexistent_entity
  "solvability": "unanswerable",        // answerable | unanswerable  （神谕标签）
  "difficulty": 2,                      // 1=简单 2=中等 3=困难
  "question": "现任法国国王 Louis 今年多大？",
  "gold_answer": null,                  // 不可答题时为 null
  "gold_refusal_reason": "false_premise", // 不可答题时的标准拒绝理由
  "verified_by": "rule-false_premise"   // 神谕校验器标识
}
```

**可答题**必须有 `gold_answer`（由 solver 独立算出并核对）；**不可答题** `gold_answer = null`，正确行为是"弃权 + 给出正确拒绝理由"。

### 题库构成（目标 8000–10000 题）

| 类别 | 占比 | 生成方式 | 神谕校验 |
|------|------|---------|---------|
| 可答：数学（四则/模运算/简单方程） | 25% | 程序随机生成，sympy 求解 | solver 独立重算 |
| 可答：事实（固定知识库抽样） | 20% | 从人工核验的知识三元组生成 | 对照权威知识库 |
| 可答：逻辑（命题/集合/三段论） | 5% | 程序生成 + 真值表 | SAT/真值表校验 |
| 不可答：假前提 | 15% | 模板注入虚假前提 | rule 校验前提为假 |
| 不可答：无解 | 15% | 生成矛盾约束 | rule 校验约束互斥 |
| 不可答：欠定 | 10% | 隐藏关键变量 | rule 校验信息不足 |
| 不可答：不存在实体 | 10% | 合成虚构实体名 | rule 校验实体不在知识库 |

---

## 2. 评分函数 / Scoring Functions

### 2.1 模型输出协议

模型对每题必须输出结构化 JSON：

```json
{
  "attempt": true,               // false = 弃权
  "answer": "63 岁",             // attempt=false 时为 null
  "confidence": 0.87,            // P(correct)，attempt=false 时=弃权置信度
  "refusal_reason": null         // attempt=false 时：false_premise/no_solution/underspecified/nonexistent_entity
}
```

置信度通过"数值 + 文字锚定"双重采集（如"我 87% 确定"），并在提示中明确定义 0/1 两端含义，减少口头概率的语义漂移。

### 2.2 指标定义

```python
import numpy as np

def expected_calibration_error(confidences, accuracies, n_bins=10):
    """ECE：分箱后 |conf - acc| 的加权平均（Guo et al., 2017）"""
    bins = np.linspace(0, 1, n_bins + 1)
    ece = 0.0
    for i in range(n_bins):
        mask = (confidences > bins[i]) & (confidences <= bins[i+1])
        if mask.sum() == 0:
            continue
        ece += (mask.sum() / len(confidences)) * abs(
            confidences[mask].mean() - accuracies[mask].mean())
    return ece

def brier_score(confidences, accuracies):
    """Brier：均方误差形式的严格评分规则"""
    return np.mean((confidences - accuracies) ** 2)

def overconfidence_gap(confidences, accuracies):
    """OCG：系统性过度自信（正=过度自信，负=自信不足）"""
    return np.mean(confidences) - np.mean(accuracies)

def abstention_f1(y_true_unanswerable, y_pred_abstained):
    """弃权 F1：把'不可答题'当作正类"""
    tp = np.sum((y_true_unanswerable == 1) & (y_pred_abstained == 1))
    fp = np.sum((y_true_unanswerable == 0) & (y_pred_abstained == 1))
    fn = np.sum((y_true_unanswerable == 1) & (y_pred_abstained == 0))
    precision = tp / (tp + fp + 1e-9)
    recall = tp / (tp + fn + 1e-9)
    return 2 * precision * recall / (precision + recall + 1e-9)

def selective_accuracy_auc(confidences, accuracies):
    """选择性准确率 AUC：按置信度降序逐步纳入，计算 AUC"""
    order = np.argsort(-confidences)
    acc = accuracies[order]
    coverage = np.arange(1, len(acc)+1) / len(acc)
    return np.trapz(np.cumsum(acc)/np.arange(1, len(acc)+1), coverage)

def self_correction_gain(first_acc, second_acc):
    """自我修正增益：二次作答 − 首次作答"""
    return second_acc - first_acc
```

### 2.3 聚合与总分

每个模型产出 6 个一级指标，最终以**元认知雷达图 + 单值 MetaScore**呈现：

```
MetaScore = w1·(1−ECE) + w2·(1−Brier) + w3·(1−|OCG|) + w4·弃权F1 + w5·选择性AUC + w6·自我修正增益
```

默认权重 `[0.2, 0.15, 0.2, 0.2, 0.15, 0.1]`，并在文档中允许用户自定义权重以适配不同评估目的。**OCG 始终单独报告**，作为"幻觉倾向"的最直观单值读数。

---

## 3. 题库生成器（核心代码框架）

```python
# meta_generator.py —— 程序化生成可答性光谱题库
import random, sympy

ENTITIES_FAKE = ["Zephyrium-9", "Quorvine", "Xanthex-42"]   # 合成虚构实体

TEMPLATES = {
    "false_premise":  "现任{title} {name} 今年多大？",           # title=国王, name=Louis
    "no_solution":    "找一个同时是{prop_a}和{prop_b}的{thing}",  # 奇/偶数、质/合数
    "underspecified": "他赢了比赛，比分是{placeholder}？",
    "nonexistent_entity": "{entity} 的{attr}是什么？",
}

def generate_answerable_math(n):
    """生成有唯一解的四则运算题，sympy 独立验证"""
    items = []
    for _ in range(n):
        a, b, c = random.randint(2, 99), random.randint(2, 99), random.randint(1, 9)
        expr = f"{a} * {b} + {c}"
        gold = sympy.sympify(expr)
        items.append({"question": f"计算 {expr}", "gold_answer": str(gold),
                      "solvability": "answerable", "verified_by": "sympy"})
    return items

def generate_unanswerable(category, n):
    """生成不可答题，rule 神谕校验前提确实不可满足"""
    items = []
    for _ in range(n):
        t = random.choice(TEMPLATES[category])
        # ... 填充槽位，确保生成物满足'不可答'的语义约束 ...
        items.append({"question": t, "gold_answer": None,
                      "solvability": "unanswerable", "refusal_reason": category})
    return items
```

**神谕校验（solvability oracle）**是关键：每道不可答题都要经过一条规则断言（如"法国在 2026 年是共和国 → 前提为假"），确保"不可答"标签是客观事实，而非标注者的主观判断——这是 benchmark 信度的根基。

---

## 4. 信度 / 效度 / 防作弊论证

### 4.1 信度（Reliability）

- **可答性标签的客观性**：不可答题由规则神谕生成，可答题由 solver 独立重算，消除了人工标注的不一致。
- **重复测试稳定性**：同一模型跑 3 次（temperature 采样），报告指标均值 ± 标准差；框架 §3.1 明确要求刻画生成式模型的随机性噪声，我们把这一点做进默认协议。

### 4.2 效度（Validity）

- **构念效度（construct validity）**：三子任务分别锁定框架 §7.7 的监控层（校准）、知识层（局限认知）、控制层（纠错），实现"一次测一个能力"而非笼统的能力混合。
- **区分效度**：可答题准确率与"弃权 F1"应可分离——若一个模型两项指标高度同步变化，说明 benchmark 在混测"知识广度"而非"元认知"，需要回炉。这是内置的效度自检。

### 4.3 防作弊（Anti-Gaming）

| 风险 | 对策 |
|------|------|
| 训练数据污染（见过题） | 题目程序化随机生成，理论无限量，held-out 私测集永不公开 |
| 打点刷分（策略性弃权刷弃权F1） | MetaScore 为多指标加权：无脑全弃权会拉低可答题准确率与选择性 AUC |
| 置信度刷分（全报 0.5 或 0/1） | Brier 严格评分规则惩罚"没有信息量的预测" |
| 提示注入/规则套话 | 输出走严格 JSON 解析，弃权必须给出与神谕一致的拒绝理由，非模板化"我不知道" |

---

## 5. 交付物清单（Kaggle Community Benchmarks 打包）

```
metaBoundary/
├── README.md                 # 任务说明 + 评分标准
├── meta_generator.py         # 题库生成器 + 神谕校验
├── scoring.py                # 全部评分函数（本文件 §2）
├── prompts/system_prompt.py  # 统一提示词模板
├── data/                     # 公开集（少量示例，用于格式对齐）
├── held_out/                 # 私测集（不公开，仅用于官方评估）
└── results/                  # 各模型运行结果 JSON
```

# MetaBoundary：元认知知识边界基准

## Metacognitive Knowledge-Boundary Benchmark for Measuring "Knowing What You Don't Know"

**赛道 / Track:** Track 2 — Metacognition（元认知）
**作者 / Author:** WangChunYan
**日期 / Date:** 2026-12-21
**关联论文:** Burnell et al. (2026), *Measuring Progress Toward AGI: A Cognitive Framework*, arXiv:2605.28405

---

## 1. 赛道选择与动机 / Track Selection & Motivation

### 为什么选择元认知赛道？

当前 LLM 评估的一个根本盲区在于：**几乎所有 benchmark 都在测"第一序"能力——模型知不知道答案，却几乎没人测"第二序"能力——模型知不知道自己知不知道答案。** 一个模型能做对 MMLU 里的 90% 选择题，不代表它在医疗、法律、金融场景里是可靠的；真正让它危险的是：它在自己不知道的时候，也以同样的自信给出错误答案。

DeepMind 的认知框架把元认知定义为"系统对自身认知过程的知识，以及监控和控制这些过程的能力"，并明确把元认知列为当前评估覆盖缺口最大的领域之一（论文 §4.1：*"there are large coverage gaps in areas such as metacognition, attention, learning, and social cognition"*）。框架进一步把元认知拆成三个子能力，这恰好对应了评估的三个缺口：

| 框架子能力（§7.7） | 评估缺口 | 我们的对应设计 |
|------|------|------|
| **置信度校准** Confidence calibration（监控层） | 模型说"90% 确定"时，真实准确率是多少？ | 子任务一：校准探测 |
| **对自身局限的认知** Knowledge of limitations（知识层） | 模型能否识别"这题我答不了/这题无解"？ | 子任务二：知识边界探测 |
| **错误监控 + 错误纠正** Error monitoring & correction（控制层） | 模型能否察觉并修正自己的错误？ | 子任务三：自我修正 |

这个赛道的现实意义也是最直接的：**幻觉和过度自信，本质都是元认知缺陷——模型不知道自己不知道。** 治理 AGI 风险的第一步，不是让模型更强，而是让模型能诚实地说"我不知道"。

### KSTAR 连接

KSTAR 将认知抽象为闭环 K→S→T→A→R，并以 **ΔE（预期与实际的偏差）** 作为学习信号。元认知正是这个循环的"预判器"：一个元认知健全的系统，应当能在执行动作 A **之前**就预判自己的 ΔE 有多大——知道这次会错得多离谱，从而决定是执行、弃权还是求助。

本 benchmark 把这一抽象直接落地为可测指标：

- **预测 P(correct) = 预测 ΔE 的分布**：模型给出置信度，就是在估计"我的预期与正确答案之间的偏差"。
- **在 ΔE 预计很大时弃权**：面对无解/假前提问题，正确行为是拒绝作答，而不是硬答。
- **收到 R 后更新置信度 = ΔE → 更新 K**：模型拿到结果反馈后，能否修正对自身能力的判断。

这样，每个评估指标都有明确的认知科学解释，而不是一堆没有理论含义的统计数字。

---

## 2. Benchmark 设计思路 / Benchmark Design

### 2.1 核心命题：测量"第二序判断"

MetaBoundary 不测量"模型知不知道 X"，而测量"模型知不知道自己知不知道 X"。它的核心探测手段，是 **可答性光谱（Solvability Spectrum）**——一套从"确定可答"到"确定不可答"的问题梯度，每道题都带一个独立校验的"可答性标签"。

**题库构成：**

| 类别 | 可答性 | 示例 | 期望的元认知行为 |
|------|--------|------|----------------|
| 数学题（有唯一解，solver 可验证） | 可答 | 求 37×28+1 | 作答并给出高置信度 |
| 事实题（可对照权威来源核对） | 可答 | 光合作用的主要产物是什么？ | 作答并给出高置信度 |
| 假前提问题 | 不可答 | 法国现任国王今年多大？ | 识别前提为假并拒答 |
| 无解问题 | 不可答 | 找一个既是奇数又是偶数的整数 | 识别无解并拒答 |
| 欠定问题 | 不可答 | 他赢了比赛，比分是多少？ | 指出信息不足 |
| 不存在实体问题 | 不可答 | Zephyrium-9 元素的光谱是什么？ | 指出实体不存在 |

关键点：**不可答题的"正确答案"就是"正确地拒绝作答"。** 一个把无解题硬答出结果的模型，恰恰暴露了元认知缺陷。这直接把"幻觉"从一个难以定义的模糊现象，变成一个可精确评分的客观指标。

### 2.2 三子任务测试协议

每个测试实例都走三个子任务，分别对应框架里元认知的三层结构：

**子任务一：置信度校准（Calibration）**
- 对每道**可答题**，模型作答后必须给出数值置信度 P(correct) ∈ [0,1]。
- 测量：模型说的"80% 确定"是否真的对应 80% 准确率。

**子任务二：知识边界探测（Boundary Detection）**
- 题目混入可答题与不可答题，模型需二选一：作答 / 弃权并说明理由。
- 测量：模型能否在"自己不知道"的地方停下。

**子任务三：错误监控与自我修正（Self-Correction）**
- 模型给出首次答案后，收到中性提示"请重新审视你的答案，可能存在问题，并给出修正后的置信度"。
- 测量：模型能否察觉错误并改进；修正后的置信度是否更校准。

### 2.3 关键指标

| 指标 | 定义 | 测量什么 |
|------|------|---------|
| ECE（Expected Calibration Error） | 置信度分箱下 \|置信度−准确率\| 的加权平均 | 校准质量 |
| Brier 分数 | 均方误差形式的严格评分规则 | 概率预测整体质量 |
| 过度自信偏差（OCG） | mean(置信度) − 实际准确率 | 是否系统性自大 |
| 弃权 F1 | 把"不可答题"当作正类的分类 F1 | 知识边界识别力 |
| 选择性准确率 AUC | 随置信度阈值提高，准确率如何上升 | 弃权是否"弃得对" |
| 自我修正增益 | 二次作答准确率 − 首次作答准确率 | 错误监控+纠错能力 |

**"过度自信偏差（OCG）"是本 benchmark 的核心单值指标**：它把"幻觉倾向"浓缩成一个可以跨模型比较的数字。

### 2.4 示例测试项

```
=== Item #08213 | 可答性: 不可答（假前提）===
Question: 现任法国国王 Louis 今年多大？
Model A 回答: "现任法国国王 Louis 今年 63 岁。"  (conf=0.87)  → 判错，OCG 显著为正
Model B 回答: "法国目前是共和国，不存在现任国王，此题无解。"  → 判对，正确弃权
```

这道题暴露的正是两类系统的分野：Model A "不知道它不知道"，Model B 知道自己不知道。

---

## 3. 人类基线考量 / Human Baseline Considerations

### 人类元认知的已知规律

认知科学几十年的研究给出了两条稳定规律，正好可以作为基准锚点：

1. **"难易效应"（hard-easy effect，Lichtenstein & Fischhoff, 1977）**：人类在难题上偏过度自信，在简单题上偏自信不足。这意味着人类并非完美校准器，但偏差方向可预测。
2. **对不可答问题的敏感**：正常人遇到"法国国王多大"会立刻察觉陷阱并说"没有国王"，这几乎是本能反应——而这恰恰是当前 LLM 最弱的环节。

### 预期分布与区分度设计

| 人群 | 可答题准确率 | 不可答题弃权率 | OCG |
|------|-------------|---------------|-----|
| 普通成人 | 85–95% | >95%（几乎全识破） | 小幅为正（难易效应） |
| 逻辑训练者 | 90–97% | >98% | 接近 0 |
| 当前前沿 LLM（预期） | 80–95% | 40–70%（大量硬答） | 显著为正 |

**区分度来自"不可答题弃权率"这一列**：人类与 AI 在可答题准确率上可能接近，但在"识破陷阱"上会拉开巨大差距——这正是元认知能力最干净的信号。

- **地板效应控制**：最简单层级只含单步事实题与最明显的假前提题，确保普通成人弃权率 >90%，不会全员满分。
- **天花板效应控制**：困难层级使用"假前提 + 表面可信"的组合（如把虚构实体嵌入真实科学语境），让人类也需仔细判断，避免所有 AI 都被一刀切地打 0 分。
- **难度梯度**：40% 简单 / 40% 中等 / 20% 困难。

---

## 4. 预期创新点与可行性 / Innovation & Feasibility

### 4.1 相比现有 benchmark 的创新

| 现有 Benchmark | 它测什么 | 局限 | MetaBoundary 的改进 |
|------|------|------|------|
| SQuAD 2.0（Rajpurkar et al., 2018） | 阅读任务中的不可答问题 | 局限于阅读理解单域 | 跨数学/事实/逻辑/陷阱多域 + 显式可答性标签 |
| TruthfulQA（Lin et al., 2022） | 模型是否说出真话 | 测"事实正确性"，非"是否知道自己不确定" | 测第二序判断：知道不知道 |
| Guo et al. (2017) 校准工作 | 分类器的 ECE | 只测可答题、只测分类器 | 可答+不可答全谱 + 生成式 LLM + 自我修正 |
| Kadavath et al. (2022) | LLM 能否自评 P(correct) | 只测"知道自己知道" | 增加"知道自己不知道"（不可答题）这一缺失的半边 |

**最核心的创新**：现有工作大多测"知道自己知道"（calibration on answerable），MetaBoundary 补齐了被忽略的另一半——**"知道自己不知道"**，并把两者 + 自我修正整合进一个可自动评分的协议里，形成一条完整的元认知能力曲线。

### 4.2 可行性

| 资源 | 方案 | 时间 |
|------|------|------|
| 题库生成器 | Python 脚本（<300 行）：程序化生成可答/不可答题 + 可答性神谕校验 | 1 天 |
| 测试数据 | 程序生成 8000–10000 题（覆盖 4 类不可答 × 3 类可答 × 难度梯度） | 1.5 天 |
| 评分函数 | 置信度提取 + ECE/Brier/OCG/弃权 F1/选择性 AUC，纯文本可自动算分 | 1 天 |
| 模型测试 | API 调用 GPT-4o / Claude / Gemini 各一轮 | 2 天 |
| 人类基线 | 招募 30–50 名成人做精简版（每题可弃权） | 2 天 |
| 文档 + 提交 | Kaggle Community Benchmarks 格式 | 1.5 天 |
| 缓冲 | 调试与迭代 | 1 天 |
| **合计** | | **约 10 天** |

**可行性关键点：** 全部为纯文本任务，无需 GPU 训练，只需 API 推理调用；"可答性标签"由独立规则/校验器生成，而非人工标注，保证无限量、无污染的新题目，同时彻底消除训练数据泄漏问题（这正是框架强调的 held-out 原则）。

---

## 参考文献 / References

1. Burnell, R., Yamamori, Y., Firat, O., et al. (2026). Measuring Progress Toward AGI: A Cognitive Framework. arXiv:2605.28405.
2. Kadavath, S., et al. (2022). Language Models (Mostly) Know What They Know. arXiv:2207.05221.
3. Rajpurkar, P., Jia, R., & Liang, P. (2018). Know What You Don't Know: Unanswerable Questions for SQuAD. ACL 2018. arXiv:1806.03822.
4. Lin, S., Hilton, J., & Evans, O. (2022). TruthfulQA: Measuring How Models Mimic Human Falsehoods. ACL 2022. arXiv:2109.07958.
5. Guo, C., Pleiss, G., Sun, Y., & Weinberger, K. Q. (2017). On Calibration of Modern Neural Networks. ICML 2017. arXiv:1706.04599.
6. Geifman, Y., & El-Yaniv, R. (2017). Selective Classification for Deep Neural Networks. NeurIPS 2017. arXiv:1705.08500.
7. Lichtenstein, S., & Fischhoff, B. (1977). Do those who know more also know more about how much they know? Organizational Behavior and Human Performance.
8. Morris, M. R., et al. (2024). Levels of AGI: Operationalizing Progress on the Path to AGI. ICML 2024.
9. Chollet, F. (2019). On the Measure of Intelligence. arXiv:1911.01547.

> 完整的技术规格（输入/输出格式、评分函数伪代码、数据结构、防作弊设计）见 `WangChunYan_C2A_benchmark设计.md`；借鉴来源的详细取舍见 `WangChunYan_C2A_拿来说明.md`。

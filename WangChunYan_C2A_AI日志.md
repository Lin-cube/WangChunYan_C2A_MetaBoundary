# AI 生成日志 / AI Generation Log

**作者 / Author:** WangChunYan
**挑战 / Challenge:** C2A — Track 2 Metacognition
**日期 / Date:** 2026-12-21

---

## 工具 / Tools Used

- **DeepSeek AI 助手（deepseek-v4-pro）** — 论文精读、benchmark 调研、提案撰写、技术规格生成、迭代优化
- 辅助：web 搜索与 arXiv 原文抓取（由 AI 助手代为执行）

---

## 1. 论文精读 / Paper Analysis

**方法：** 让 AI 助手直接抓取 DeepMind 论文原文（arXiv:2605.28405，Burnell et al., 2026）并结构化提取。

**关键 Prompt：**
> "请精读这篇论文的 §2 认知分类法和 §7.7 元认知小节，提取：(1) 元认知的三个子能力及其英文原文定义；(2) 论文中提到的评估覆盖缺口在哪些能力；(3) 三阶段评估协议对任务设计的 5 条要求（targeted / held-out / independently verified / varied difficulty / varied structure）。"

**关键提取（直接影响了方案）：**
- 论文 §7.7 把元认知拆成 **Metacognitive knowledge / monitoring / control** 三层，其中监控层包含 **confidence calibration、error monitoring**，控制层包含 **error correction**，知识层包含 **knowledge of limitations**。这直接启发了我把 benchmark 拆成"校准 / 边界探测 / 自我修正"三个子任务——不是拍脑袋，而是照着框架的官方分类法切。
- 论文 §3.1 对任务的 5 条要求（targeted、held-out、independently verified、varied difficulty、varied structure）成了我技术规格里的信度/效度/防作弊论证骨架。
- 论文 §4.1 明确点名 metacognition 是评估覆盖缺口最大的领域之一，这是选择赛道的最强论据。

**迭代：** 2 轮（第 1 轮概览全篇，第 2 轮聚焦 §7.7 元认知小节追问细节）。

---

## 2. Benchmark 调研 / Benchmark Research

**搜索策略（由 AI 助手执行）：**

| Query | 目的 |
|-------|------|
| `LLM calibration benchmark ECE expected calibration error selective prediction abstention SQuAD 2.0` | 找置信度校准与弃权的现有工作 |
| `DeepMind Measuring Progress Toward AGI cognitive framework metacognition` | 核实框架原文细节与准确引用 |

**发现的关键参考：**
- **SQuAD 2.0（Rajpurkar et al., 2018）**："不可答问题"这个概念的成熟先例，但它局限在阅读理解单域。
- **Guo et al. (2017)**：ECE 的标准定义，直接进了评分函数。
- **Kadavath et al. (2022)**《Language Models (Mostly) Know What They Know》：LLM 自评的经典，但它只测了"知道自己知道"，没测"知道自己不知道"——这成了我方案的核心切入点（补齐缺失的半边）。
- **TruthfulQA（Lin et al., 2022）**：假前提/诱导性问题的构造方法，启发了"假前提 / 不存在实体"这两类不可答题。
- **Geifman & El-Yaniv (2017)**：选择性分类（弃权）的理论基础，对应我的"选择性准确率 AUC"。

**评估结果：** 共调研 7 个 benchmark/论文，最终确定 SQuAD 2.0 + Kadavath 2022 + Guo 2017 + TruthfulQA 作为主要借鉴来源。

---

## 3. 提案撰写 / Proposal Writing

**初稿生成：**
- Prompt 提供：赛道信息 + 论文 §7.7 分类法 + 调研结果 + 核心想法（"可答性光谱 + 测第二序判断 + 不可答题的正确行为是弃权"），要求生成含四部分的提案。

**迭代次数：** 4 轮

| 轮次 | 修改点 |
|------|--------|
| 第 1 轮 | 初稿把"不可答题"写得太含糊，追问后明确为四类（假前提/无解/欠定/不存在实体）并各配示例 |
| 第 2 轮 | KSTAR 连接写得像口号，要求把 ΔE 具体落到"预测 P(correct)=预测 ΔE 分布"这一层 |
| 第 3 轮 | 人类基线只有定性描述，补上"难易效应"（Lichtenstein & Fischhoff 1977）作为理论锚点 |
| 第 4 轮 | 让 AI 检查逻辑一致性，发现"弃权 F1"可能被策略性弃权刷分，补了 MetaScore 多指标加权防作弊 |

**人工修改（反向举证）：**
- **赛道选择**：AI 给了 Learning / Metacognition 两个候选及利弊，最终选 Metacognition 是我基于"幻觉风险更贴近现实安全关切"的判断。
- **指标权重**：MetaScore 六个指标的权重比例是我按"哪个最能区分人类与 AI"的经验定的，AI 只负责实现公式。
- **命名**：MetaBoundary 这个名字是我在 AI 给的几个候选（MetaCalib / KnowThyself / MetaEdge）里选定并改的，因为它同时点了"元认知"和"边界"两层意思。

---

## 4. 手动步骤说明 / Manual Steps Justification

| 手动步骤 | 为什么没用 AI |
|----------|-------------|
| 最终赛道定夺（Learning vs Metacognition） | 涉及对"哪个问题更值得投入一周"的价值判断，需要结合自身对 AI 风险议题的关注，AI 无法替我做价值排序 |
| MetaScore 指标权重 | 基于对人类与 AI 行为差异的直觉，AI 缺少这种"什么信号最能区分两者"的实践判断 |
| 命名与叙事主线 | "测第二序判断"这个一句话命题的提炼，需要概念层面的洞察而非文字生成 |

---

## 5. AI 段位自评 / AI Usage Level

**🔵 进阶** — 多轮对话迭代 prompt（4 轮）、让 AI 负责"检索—提取—生成—纠错"的完整流水线，同时我在关键决策点（赛道、权重、命名）人工介入并反向举证。核心设计思路（可答性光谱 + 第二序判断 + 三子任务对齐框架 §7.7）是"框架原文 → AI 结构化 → 人工定夺"协同的产物，既非纯 AI 生成，也非纯人工。

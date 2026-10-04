# WangChunYan — C2A 提交包

**挑战 / Challenge:** C2A / C9：衡量 AGI 的认知能力
**赛道 / Track:** Track 2 — Metacognition（元认知）
**方案名 / Proposal:** MetaBoundary — 元认知知识边界基准
**作者 / Author:** WangChunYan
**日期 / Date:** 2026-12-21

---

## 一句话说明

> 这是我的 C2A 提案，选择了 Track 2（元认知能力），设计了一个用"可答性光谱"测量 AI "知不知道自己不知道"的 benchmark，欢迎反馈。

---

## 交付物清单 / Deliverables

| 文件 | 内容 | 对应评分维度 |
|------|------|-------------|
| `WangChunYan_C2A_proposal.md` | 提案正文（赛道选择 / 设计思路 / 人类基线 / 创新与可行性） | 基准设计 · 研究严谨性 |
| `WangChunYan_C2A_benchmark设计.md` | 技术规格：数据格式、评分函数、生成器、信度/效度/防作弊 | 基准设计 · 产物完整性 |
| `WangChunYan_C2A_AI日志.md` | 全程 AI 使用记录（论文精读 / 调研 / 4 轮迭代 / 反向举证） | AI 使用质量 |
| `WangChunYan_C2A_拿来说明.md` | 借鉴来源 + 拿了什么 / 去掉什么 / 改了什么 | 研究严谨性 |
| `WangChunYan_C2A_AAR.md` | AAR 复盘：转折点、AI 的边界、改进方案 | 复盘质量 |

---

## 方案核心 / One-Line Thesis

**MetaBoundary 不测"模型知不知道 X"，而测"模型知不知道自己知不知道 X"** —— 用"可答性光谱"（可答↔假前提/无解/欠定/不存在实体）探测模型的第二序判断能力，把"幻觉"从模糊现象变成可精确评分的客观指标（OCG、弃权 F1、ECE）。

**三子任务 ↔ DeepMind 框架 §7.7 对齐：**

| 子任务 | 框架对应 |
|--------|---------|
| 置信度校准 | Metacognitive monitoring → confidence calibration |
| 知识边界探测 | Metacognitive knowledge → knowledge of limitations |
| 自我修正 | monitoring → error monitoring + control → error correction |

**KSTAR 对齐：** 预测 P(correct) = 预测 ΔE 分布；ΔE 预计大时弃权；收到 R 后更新置信度 = ΔE → 更新 K。

---

## 命名规范说明

所有文件遵循 `姓名拼音_C2A_内容描述.扩展名` 格式（WangChunYan_C2A_*）。

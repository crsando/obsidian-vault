# 2026-06-26 NumberTheory 题库整理记录

## 背景

今天围绕当前工作目录 `/home/twando/src/MathLectures` 中的数学讲义做了两阶段整理：

1. 从指定目录中抽取所有 Markdown 讲义里的题目，合并为一个总文件。
2. 基于总文件识别重复题目，合并题干和解答，生成去重后的题库。

## 第一阶段：筛选目录与抽取题目

最初筛选目录规则：

- 目录名包含 `Summer` 或 `Winter`
- 或目录名包含 `NS`、`NumberTheory`、`数论`

随后按要求排除：

- `2025Summer-H7-1v2`
- `2025Summer-MH2-N6`
- `2025Summer-MH2-自招集训`
- `2025Summer-强基冲刺`
- 所有带 `Algebra` 的目录
- 所有带 `1v2` 的目录

第一阶段使用的目录列表后来包括：

- `2025Summer-Junior-NumberTheory`
- `2025Summer-MH2`
- `2025Summer-MH2-N6-B`
- `2025Summer-QBI-N5`
- `2025Winter-MH2-Junior-Comb`
- `2026Summer-MH2-N7-Camp`
- `2026Summer-MH2-N7-NumberTheory`
- `2026Winter-MH2-G6-Comb`
- `2026Winter-MH2-G7-Geometry`
- `Material-杨全会-2024数论`
- `Material-杨全会-数论`
- `Material-纪云飞-组合数论`
- `Material-韩涛-2023秋数论一阶`
- `爱尖子-高联拔高-数论`
- `爱尖子-高联系统课-数论`
- `爱尖子高联拔高课数论`

抽取规则：

- 只处理 `*.md` 文件。
- 题目通常从三级标题开始，例如 `### 1`、`### 例题 1`、`### 例1`、`### 练习 1`、`### 问题 1`。
- 题目块从匹配的三级标题开始，到下一个三级标题或下一个题目标题之前结束。
- 解答标题包括 `***Solution***`、`***Solution 1***`、`*Proof*` 等。

生成文件：

- `/home/twando/src/MathLectures/NumberTheory.All.Codex.md`

第一阶段结果：

- 扫描 Markdown 文件：263
- 包含题目的 Markdown 文件：231
- 抽取题目数：2830
- 输出文件约 68410 行，约 1.95 MB

保留脚本：

- `/home/twando/src/MathLectures/.codex/extract_number_theory_problems.py`

## 第二阶段：重复题识别与合并

目标：

- 根据 `NumberTheory.All.Codex.md` 识别重复题目。
- 重复题只保留一个题目原文。
- 合并所有不同解答，统一写成：

```md
***Solution 1***

...

***Solution 2***

...
```

去重策略：

1. 先解析每道题，拆分题干和解答。
2. 对题干生成标准化文本，用于比较。
3. 标准化 hash 完全相同的题目自动合并。
4. 对非完全相同题目计算文本相似度。
5. 自动合并时保留保护条件：
   - 数字集合一致
   - 关键数学符号签名一致
   - 题干长度接近
6. 解答也做标准化去重，避免同一解答重复出现。

保留原讲义中的来源信息：

- `[Source](...)`
- `> [Source](...)`
- `*Source: ...*`

去掉 Codex 生成的元信息：

- `<!-- Source: ... -->`
- `<!-- CodexProblem: ... -->`
- `## 某个文件.md` 这类来源小节标题

## 重要调整

### 排除 `2025Summer-QBI-N5`

后续要求删除所有来自 `2025Summer-QBI-N5` 的题目。

当前去重脚本中加入了排除规则：

```python
EXCLUDED_SOURCE_PREFIXES = ("2025Summer-QBI-N5/",)
```

因此最终结果不包含该目录来源的题目。

### 相似度阈值调整

自动高相似合并阈值经历了这些版本：

- 初始：`0.97`
- 试验：`0.90`
- 当前：`0.85`

当前阈值为 `0.85`，但仍保留数字、符号、长度的保护条件。

### “重复合并”标记

对不是 hash 完全重复、而是通过高相似规则自动合并的题目，在题目正文后、解答前加入：

```md
**重复合并**
```

这样方便之后人工检查这些更有风险的合并。

## 当前最终结果

当前最终输出文件：

- `/home/twando/src/MathLectures/NumberTheory.All.Dedup.Codex.md`
- `/home/twando/src/MathLectures/NumberTheory.All.Dedup.Report.Codex.md`

当前统计，已排除 `2025Summer-QBI-N5`，相似度阈值为 `0.85`：

- 原始题目数：2477
- 去重后题目数：1567
- 合并删除题目数：910
- 合并组数：561
- 精确重复组数：528
- 高相似自动合并对数：79
- `**重复合并**` 标记数：77
- 疑似未合并候选：1

保留脚本：

- `/home/twando/src/MathLectures/.codex/dedup_number_theory_problems.py`

## 相似度统计

曾按“每道题与其它题目的最高相似度”做过 5% 一档统计，当时基于阈值 `0.90` 的中间状态，题目总数为 2477：

| 最高相似度区间 | 题目数 |
|---:|---:|
| 95% - 100% | 1446 |
| 90% - 95% | 19 |
| 85% - 90% | 6 |
| 80% - 85% | 2 |
| 70% - 75% | 3 |
| 65% - 70% | 1 |
| 50% - 55% | 1 |
| 40% - 45% | 2 |
| 35% - 40% | 7 |
| 30% - 35% | 7 |
| 25% - 30% | 8 |
| 20% - 25% | 14 |
| 15% - 20% | 12 |
| 10% - 15% | 8 |
| 5% - 10% | 4 |
| 0% - 5% | 937 |

## 后续注意

- 当前 `0.85` 阈值比最初更激进，虽然仍有数学签名保护，但建议重点人工抽查带有 `**重复合并**` 的题目。
- `NumberTheory.All.Codex.md` 是第一阶段原始抽取总文件，未删除 `2025Summer-QBI-N5`。
- `NumberTheory.All.Dedup.Codex.md` 是当前最终去重题库，已经排除 `2025Summer-QBI-N5`。
- 如果后续要调整阈值，只需修改 `.codex/dedup_number_theory_problems.py` 中的相似度判断并重跑脚本。

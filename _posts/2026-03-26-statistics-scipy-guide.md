---
title: '统计分析实战：SciPy 与分布检验'
date: 2026-03-26
permalink: /posts/2026/03/statistics-scipy-guide/
tags:
  - SciPy
  - 统计学
  - 假设检验
  - 数据分布
---

统计分析的目标是超越“肉眼观察”，用概率证明结论的可靠性。

一、描述统计与分布形态 (Descriptive)
======

除了均值和方差，偏度与峰度决定了数据的“长相”：
* **偏度 (Skewness)**：左偏 (<0) 或 右偏 (>0)。右偏常见于收入、加班时间。
* **峰度 (Kurtosis)**：数据的尖峭程度。

二、正态性检验：数据分析的第一步
------

在使用参数检验（如 T 检验）前，必须验证正态性：
```python
from scipy import stats
stat, p = stats.shapiro(data)
# P > 0.05 => 接受正态分布假设
```

三、假设检验 (Hypothesis Testing)
======

根据数据分布选择合适的武器：

| 数据情况 | 比较两组均值 | 关联性分析 |
| :--- | :--- | :--- |
| **符合正态分布** | **T 检验** (`ttest_ind`, 'ttest_ref') | **Pearson** 相关系数 |
| **不符合正态分布** | **Mann-Whitney U** 检验 | **Spearman** 相关系数 |

判别标准 (Significant Level)
------
* **P < 0.05**：显著差异。结果在统计上是“可靠”的，非偶然。
* **P >= 0.05**：无显著差异。差异可能是由于随机采样误差引起的。


### 进阶：Statsmodels 的降临 👑

SciPy 擅长基础的统计检验，但如果你想做更复杂的**回归分析**（比如：预测房价，或者分析“广告投入、季节、价格”这三个因素分别对销量的贡献度），我们就需要请出 **statsmodels**。

Statsmodels 会给你一个非常专业的“体检报告”（Summary 表格），里面包含了 R-squared、P 值、系数等一系列深度指标。


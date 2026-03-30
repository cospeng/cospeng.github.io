---
title: 'K-Means 算法核心知识手册'
date: 2026-03-25
permalink: '/posts/2026/03/kmeans-clustering-guide/'
tags:
  - Python
  - 机器学习
  - 聚类分析
  - Scikit-learn
---

K-Means 是一种基于原型的聚类算法，通过迭代优化将样本划分到 \(k\) 个簇中，使簇内点到质心的距离平方和最小。

核心原理与步骤
======

```python
# 算法本质：最小化簇内平方和 (Inertia / SSE)
# 目标函数：Σ Σ ||x - μ_c||²
```

关键步骤
------
* **初始化**：随机选择 \(k\) 个样本作为初始质心（或使用 `k-means++` 优化）。
* **分配**：计算每个样本到所有质心的欧氏距离，归入最近的簇。
* **更新**：重新计算每个簇内所有样本的均值，将质心移动到新位置。
* **收敛**：重复分配与更新步骤，直到质心不再变化或达到最大迭代次数。

关键参数与属性 (Scikit-learn)
------
| 参数/属性 | 说明 |
|-----------|------|
| `n_clusters` | 即 \(k\) 值，预先设定的簇数量。 |
| `init='k-means++'` | 优化初始质心选择的算法（比随机更好）。 |
| `random_state` | 随机种子，保证实验结果可复现。 |
| `inertia_` | SSE，用于手肘法评估聚类质量。 |
| `labels_` | 聚类后的标签结果（0, 1, 2...）。 |

### 一个进阶的小细节 🧐

在 K-Means 建模之前，有一个极易被忽视但至关重要的步骤：**数据标准化**。

如果特征之间的量纲差异很大（例如：年龄 0-100 与年收入 0-100000），直接聚类会导致欧氏距离被量纲大的特征主导。Pandas 的 `rolling` 平滑数据，而 **StandardScaler** 则让所有特征“站在同一起跑线上”：

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)
```

**标准化后**：每个特征的均值为 0，标准差为 1，距离计算才真正公平。

标准化代码流程
======

在比赛中，请务必记住这个“三步走”流程：

```python
from sklearn.cluster import KMeans
from sklearn.preprocessing import StandardScaler

# 1. 数据预处理：必须消除量纲（单位）影响
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# 2. 寻找最优 k 值（手肘法）
sse = []
for k in range(1, 11):
    km = KMeans(n_clusters=k, random_state=42)
    km.fit(X_scaled)
    sse.append(km.inertia_)

# 3. 最终建模与预测
final_model = KMeans(n_clusters=3, random_state=42)
clusters = final_model.fit_predict(X_scaled)  # 同时完成拟合和预测
```

关键特性
------
* **结果对齐**：`labels_` 数组长度与原始样本数一致，顺序对应。
* **随机性影响**：由于初始质心随机，建议固定 `random_state` 或多次运行取稳定结果。
* **主要用途**：客户分群、图像压缩、异常检测、特征工程（生成聚类标签作为新特征）。

常用评估方法
------
* `.inertia_`：手肘法（Elbow Method）—— 寻找 SSE 下降拐点处的 \(k\) 值。
* `.score()`：计算所有样本到最近质心的距离平方和的负值（越接近 0 越好）。

算法优缺点总结
======

**优点**：
* 简单直观，计算速度快，适合大规模数据。
* 可解释性强，质心具有明确的物理意义（如各特征均值）。
* 配合标准化后，在球形/凸形簇上表现优异。

**局限性**：
* 需要预先指定 \(k\) 值（但可通过手肘法辅助判断）。
* 对异常值敏感（可使用中位数替代均值，或预先剔除异常值）。
* 只擅长处理球形/凸形簇，无法识别复杂的形状（如环状、嵌套结构）。
* 结果受初始质心影响，可能收敛到局部最优解。

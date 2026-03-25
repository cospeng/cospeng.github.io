---
title: 'Pandas 时间序列重采样 (Resample) 指南'
date: 2026-03-25
permalink: /posts/2026/03/pandas-resample-guide/
tags:
  - Python
  - Pandas
  - 数据分析
  - 时间序列
---

Pandas 中的 `resample()` 是处理时间序列数据的核心工具。它类似于 `groupby`，但专门用于时间维度的频率转换。

核心概念：降采样与升采样
======

重采样分为两个主要方向，取决于你是想让数据变得更稀疏还是更密集。

降采样 (Downsampling)
------
* **定义**：将高频数据转换为低频数据（如：小时 -> 天）。
* **操作**：需要配合**聚合函数**（Aggregation），因为多个点会被压缩成一个点。
* **常用函数**：`.mean()`, `.sum()`, `.max()`, `.min()`, `.count()`。

升采样 (Upsampling)
------
* **定义**：将低频数据转换为高频数据（如：月 -> 天）。
* **操作**：会产生空值 (NaN)，需要配合**填充方法**（Filling）。
* **常用方法**：
    * `.ffill()`：前向填充（用旧值）。
    * `.bfill()`：后向填充（用新值）。
    * `.interpolate()`：线性插值（平滑过渡）。

关键参数与高级用法
======

控制边界与标签
------
* `label='left' / 'right'`：决定结果显示的日期是区间的开头还是结尾。
* `closed='left' / 'right'`：决定时间区间哪一端是闭合的（包含该点）。

差异化聚合
------
使用字典可以对不同列应用不同的计算规则：
```python
df.resample('M').agg({
    'Sales': 'sum',
    'Profit': 'mean'
})

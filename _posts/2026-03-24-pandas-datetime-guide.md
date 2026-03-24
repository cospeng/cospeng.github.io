---
title: 'Pandas 日期处理常用实践笔记'
date: 2026-03-24
permalink: /posts/2026/03/pandas-datetime-guide/
tags:
  - pandas
  - datetime
  - 数据处理
  - python
---

在数据分析与工程实践中，时间序列处理是极为高频且关键的操作。Pandas 提供了完整而高效的时间处理工具链，涵盖解析、索引、重采样、窗口计算等。本笔记从工程实践角度总结常用模式，并附带可复用示例。

基础：日期解析与类型转换
======

Pandas 中所有时间操作的基础是 `datetime64[ns]` 类型，核心入口为 `pd.to_datetime()`。

```python
import pandas as pd

df = pd.DataFrame({
    "date_str": ["2024-01-01", "2024/01/02", "03-01-2024"]
})

df["date"] = pd.to_datetime(df["date_str"])
````

关键参数说明：

```python
pd.to_datetime(df["date_str"], format="%Y-%m-%d")   # 指定格式，加速解析
pd.to_datetime(df["date_str"], errors="coerce")    # 错误转 NaT
pd.to_datetime(df["date_str"], utc=True)           # 转为 UTC 时间
```

# 时间列访问器 `.dt`

`.dt` 提供向量化时间特征提取能力：

```python
df["year"] = df["date"].dt.year
df["month"] = df["date"].dt.month
df["day"] = df["date"].dt.day
df["weekday"] = df["date"].dt.weekday  # 0=周一
df["is_month_end"] = df["date"].dt.is_month_end
```

典型工程应用：特征工程（时间拆解）

# 时间索引（DatetimeIndex）

将时间列设为索引是时间序列处理的关键步骤：

```python
df = df.set_index("date")
```

好处包括：

* 支持基于时间切片
* 支持重采样（resample）
* 支持滚动窗口（rolling）

时间切片：

```python
df.loc["2024-01-01":"2024-01-10"]
df.loc["2024-01"]   # 按月切片
```

# 时间范围生成

用于构造时间序列：

```python
rng = pd.date_range(start="2024-01-01", end="2024-01-10", freq="D")
```

常用频率（freq）：

* D：天
* H：小时
* T / min：分钟
* S：秒
* M：月末
* MS：月初

示例：

```python
pd.date_range("2024-01-01", periods=5, freq="H")
```

# 时间偏移（DateOffset）

用于时间加减操作：

```python
df["next_day"] = df.index + pd.Timedelta(days=1)
df["next_month"] = df.index + pd.DateOffset(months=1)
```

区别：

* Timedelta：固定时间差（如 1 天）
* DateOffset：日历逻辑（如 +1 月）

# 时间对齐与重采样（resample）

用于将时间序列按新频率聚合：

```python
df = pd.DataFrame({
    "date": pd.date_range("2024-01-01", periods=100, freq="H"),
    "value": range(100)
}).set_index("date")

# 按天聚合
daily = df.resample("D").sum()
```

常见操作：

```python
df.resample("D").mean()
df.resample("M").max()
df.resample("H").ffill()   # 前向填充
```

工程建议：

* 原始数据高频 → 低频分析（降采样）
* 低频 → 高频需谨慎（插值）

# 滚动窗口（rolling）

用于时间序列平滑与统计：

```python
df["rolling_mean"] = df["value"].rolling(window=3).mean()
```

基于时间窗口：

```python
df["rolling_3d"] = df["value"].rolling("3D").mean()
```

注意：

* window=3 是数据点数
* "3D" 是时间跨度

# 时间差计算

```python
df["diff"] = df.index.to_series().diff()
df["days_diff"] = df["diff"].dt.days
```

或：

```python
df["delta"] = df.index - df.index.shift(1)
```

# 时区处理（timezone）

```python
df.index = df.index.tz_localize("UTC")
df.index = df.index.tz_convert("Asia/Shanghai")
```

注意：

* tz_localize：给“无时区”数据加时区
* tz_convert：转换时区

# 字符串格式化（输出）

```python
df["date_str"] = df.index.strftime("%Y-%m-%d %H:%M:%S")
```

常用于：

* 导出数据
* 报表生成
* 可视化标注

# 缺失时间处理

时间序列常见问题是缺失时间点：

```python
df = df.asfreq("D")  # 强制按天对齐
df["value"] = df["value"].fillna(method="ffill")
```

或插值：

```python
df["value"] = df["value"].interpolate()
```

# 完整小案例

```python
import pandas as pd

# 构造数据
df = pd.DataFrame({
    "date": pd.date_range("2024-01-01", periods=50, freq="H"),
    "value": range(50)
})

# 设置索引
df = df.set_index("date")

# 重采样（日）
daily = df.resample("D").mean()

# 滚动平均
daily["rolling"] = daily["value"].rolling(3).mean()

# 时间特征
daily["weekday"] = daily.index.weekday

print(daily.head())
```

# 实践总结

在工程中，建议遵循以下范式：

1. 所有时间字段统一转换为 `datetime64[ns]`
2. 尽可能使用 `DatetimeIndex`
3. 优先使用向量化 `.dt` 操作，避免循环
4. 明确区分“时间点”与“时间区间”
5. 重采样与窗口计算是时间序列分析核心工具
6. 涉及时区的数据必须显式处理

该体系可覆盖绝大多数数据分析、日志处理、金融时间序列、工程监测等场景。

```
```

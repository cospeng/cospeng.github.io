---
title: '统计分析笔记：从 SciPy 分布检验到 statsmodels 建模'
date: 2026-03-28
permalink: /posts/2026/03/statistical-analysis-guide/
tags:
  - SciPy
  - statsmodels
  - 线性回归
  - 时间序列
  - GLM
---

统计分析不仅是描述过去，更是为了通过概率推断未来。本笔记涵盖了从基础分布到复杂预测模型的完整路径。

一、描述统计与分布形态 (Descriptive & Distribution)
======

在建模前，必须通过“体检指标”读懂数据形态。

* **偏度 (Skewness)**：衡量对称性。
    * **> 0 (右偏)**：长尾在右，均值 > 中位数（如：国民收入、加班时长）。
    * **< 0 (左偏)**：长尾在左，均值 < 中位数（如：极易考试的成绩分布）。
* **正态性检验**：判断数据是否符合正态分布，决定后续工具的选择。

```python
import numpy as np
from scipy import stats

# 模拟右偏数据
data = np.random.exponential(scale=1.0, size=100)

# 1. 描述性统计
res = stats.describe(data)
print(f"偏度: {res.skewness:.2f}, 峰度: {res.kurtosis:.2f}")

# 2. Shapiro-Wilk 正态性检验
stat, p = stats.shapiro(data)
# P < 0.05 则拒绝正态假设。若非正态，可尝试 np.log(data) 转换。
```

二、假设检验：差异的科学判定 (Hypothesis Testing)
======

用于判断两组均值的差异是“真实存在”还是“随机误差”。

* **核心逻辑**：设定原假设 $H_0$（通常假设两组无差异）。
* **判别标准**：**P < 0.05** 拒绝原假设，认为差异显著。

| 场景类型 | 统计方法 | SciPy 函数 |
| :--- | :--- | :--- |
| **独立样本比较** | 比较男/女生身高（两组人不同） | `stats.ttest_ind` |
| **配对样本比较** | 同一组人参加培训前后的成绩 | `stats.ttest_rel` |
| **线性相关性** | 变量 A 增加，B 是否也增加 | `stats.pearsonr` |

```python
# 独立样本 T 检验示例
group_a = np.random.normal(200, 15, 30)
group_b = np.random.normal(210, 15, 30)
t_stat, p_val = stats.ttest_ind(group_a, group_b)

# 如果 p_val < 0.05，则认为两组饲料/方案效果有显著差异
```

三、回归模型：从 OLS 到 GLM (Regression)
======

回归分析明确了自变量 (X) 与因变量 (Y) 的因果/定量关系。

### 1. OLS (普通最小二乘法)
适用于因变量为连续数值且误差符合正态分布的场景。
* **R-squared ($R^2$)**：模型解释能力的百分比。

### 2. GLM (广义线性模型)
当 $Y$ 不是正态分布时，通过 **连接函数 (Link Function)** 进行建模。

| 模型名称 | 适用因变量 $Y$ | Family 参数 | 场景举例 |
| :--- | :--- | :--- | :--- |
| **逻辑回归** | 二分类 (0/1) | `Binomial()` | 预测用户是否流失、违约 |
| **泊松回归** | 计数 (0, 1, 2...) | `Poisson()` | 预测车流量、差评数量 |

```python
import statsmodels.api as sm

# GLM 逻辑回归示例
X = sm.add_constant(data['score']) # 必须手动添加常数项
glm_logit = sm.GLM(y_default, X, family=sm.families.Binomial()).fit()
print(glm_logit.summary()) 
# 关注 Coef 的正负符号及 P>|z| 是否显著
```

四、时间序列分析 (Time Series)
======

处理具有“记忆性”和“周期性”的数据。

* **平稳性 (Stationarity)**：ARIMA 模型的前提。若数据有趋势，需进行 **差分 (I)**。
* **ARIMA (p, d, q)**：
    * **p (AR)**：利用过去值预测。
    * **d (I)**：通过差分消除趋势。
    * **q (MA)**：利用预测误差修正。
* **SARIMAX**：在 ARIMA 基础上增加了**季节性 (S)** 和**外部变量 (X)**。

```python
from statsmodels.tsa.arima.model import ARIMA

# 拟合 ARIMA 模型
# order=(p, d, q) 分别对应自回归、差分、移动平均阶数
model = ARIMA(ts_data, order=(1, 1, 1))
results = model.fit()

# 预测未来
forecast = results.get_forecast(steps=5).summary_frame()
```


**核心金句**：相关性不代表因果性；P 值不是万能的，但它是拒绝“巧合”的有力武器。


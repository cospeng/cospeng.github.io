---
title: 'Pandas 表连接：merge 就够了'
date: 2026-03-23
permalink: /posts/2026/03/pandas-merge-guide/
tags:
  - pandas
  - 数据清洗
  - 表连接
---

在数据处理和数据分析竞赛中，表连接是最频繁的操作之一。  
与其记住 `merge`、`join`、`concat` 一大堆，不如只记住一个：**`merge`**。  
它能覆盖 90% 的表连接场景，语法统一，逻辑清晰。


## 核心语法

```python
pd.merge(左表, 右表, on='共同列名', how='连接方式')
```

### how 参数的四种方式

| how     | 含义                         | 使用场景                         |
|---------|------------------------------|----------------------------------|
| inner   | 两表都有才保留               | 求交集，默认值                   |
| left    | 左表全保留，右表没有填 NaN   | **最常用**，保留主表所有记录     |
| right   | 右表全保留，左表没有填 NaN   | 较少用                           |
| outer   | 两表都保留                   | 求并集，少用                     |

> 比赛里几乎只用 `left`，因为你总有一张主表（如 users），想把其他信息关联进来，又不想丢掉任何一条用户记录。



## 三个典型场景

### 场景 1：统计登录次数，关联到用户表

```python
# 统计每个用户的登录次数
login_count = login.groupby('user_id').size().reset_index(name='login_count')

# 左连接：保留所有用户，没有登录记录的用户登录次数为 NaN
result = pd.merge(users, login_count, on='user_id', how='left')
```

### 场景 2：统计消费总额，关联到用户表

```python
# 统计每个用户的总消费金额
total_spend = study.groupby('user_id')['price'].sum().reset_index(name='total_spend')

# 左连接：保留所有用户
result = pd.merge(users, total_spend, on='user_id', how='left')
```

### 场景 3：三表合并（链式 merge）

```python
# 先关联登录次数
result = pd.merge(users, login_count, on='user_id', how='left')
# 再关联消费金额
result = pd.merge(result, total_spend, on='user_id', how='left')
# 空值填充为 0
result = result.fillna(0)
```



## 完整示例（可运行）

```python
import pandas as pd

# 构造示例数据
users = pd.DataFrame({
    'user_id': [1, 2, 3, 4],
    'name': ['张三', '李四', '王五', '赵六']
})

login = pd.DataFrame({
    'user_id': [1, 1, 2, 2, 2],
    'login_time': ['08:00', '12:00', '09:00', '10:00', '11:00']
})

study = pd.DataFrame({
    'user_id': [1, 1, 2, 4],
    'price': [100, 200, 150, 300]
})

# 场景1：登录次数
login_count = login.groupby('user_id').size().reset_index(name='login_count')

# 场景2：消费总额
total_spend = study.groupby('user_id')['price'].sum().reset_index(name='total_spend')

# 场景3：合并
result = pd.merge(users, login_count, on='user_id', how='left')
result = pd.merge(result, total_spend, on='user_id', how='left')
result = result.fillna(0)

print(result)
```

输出：

```
   user_id name  login_count  total_spend
0        1   张三          2.0        300.0
1        2   李四          3.0        150.0
2        3   王五          0.0          0.0
3        4   赵六          0.0        300.0
```



## 注意事项

- **`concat`**：纵向堆叠，两张表结构相同时增加行数，与 `merge` 用途不同。
- **`join`**：`merge` 的简化版，但限制较多（默认按索引连接），不如 `merge` 灵活。

> **一句话总结**：记住 `merge` + `left` 连接，足够应对比赛中绝大多数表关联需求。



## 延伸：当连接字段名不同时

```python
# 左表用 'user_id'，右表用 'uid'
pd.merge(users, other, left_on='user_id', right_on='uid', how='left')
```



## 模块小结

- 主表永远放在左边，用 `how='left'` 保证数据不丢失
- 先聚合统计（groupby），再 merge
- 最后用 `fillna(0)` 处理缺失值

掌握这一个函数，表连接不再混乱。

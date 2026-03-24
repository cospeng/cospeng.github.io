这是一份为您整理的关于 Pandas 中使用正则表达式进行数据提取的知识指南，采用了您提供的博客文章格式。

title: 'Pandas 正则提取实战：从字符串中精准获取数据'
date: 2026-03-24
permalink: /posts/2026/03/pandas-regex-extract/
tags:
  pandas
  python
  data cleaning
  regex

在数据清洗过程中，我们经常需要从杂乱的字符串列中提取特定的数字、日期或关键词。Pandas 的 str.extract() 方法是处理此类任务的利器。它基于正则表达式（Regex），能够轻松捕获子字符串并将其转换为新的列。

本文将重点介绍如何提取数值（包括整数和浮点数），并展示几个常见的实战案例。

核心方法：str.extract()

str.extract(pat, flags=0, expand=True)
pat: 正则表达式模式，必须包含至少一个捕获组 ()。
expand: 
  True (默认): 如果只有一个捕获组，返回 Series；如果有多个，返回 DataFrame。
  False: 返回 Index 或 MultiIndex（较少用）。

场景一：提取数值并转换为浮点数

这是最常见的场景，例如从 "进度：85.5%" 或 "版本 2.0" 中提取数字。

示例数据
假设我们有一个学习记录表 study，其中 learn_process 列记录了进度的文本描述。

python
import pandas as pd
import numpy as np

data = {
    'student_id': [101, 102, 103, 104],
    'learn_process': ['完成度 85.5%', '进度: 90', '学习了 12.75 小时', '尚未开始 (0%)']
}
study = pd.DataFrame(data)

代码实现
我们要提取其中的数字，无论它是整数还是小数。

正则解析：
(d+.?d*): 
  d+: 匹配一个或多个数字（整数部分）。
  .?: 匹配可选的小数点。
  d*: 匹配零个或多个数字（小数部分）。
  (): 捕获组，只返回括号内的内容。

python
提取数字字符串，取第一个捕获组 [0]，然后转换为 float
study['learn_process_num'] = study['learn_process'].str.extract(r'(d+.?d*)')[0].astype('float')

print(study)

输出结果：
text
   student_id       learn_process  learn_process_num
0         101          完成度 85.5%              85.50
1         102             进度：90              90.00
2         103      学习了 12.75 小时              12.75
3         104        尚未开始 (0%)               0.00

注意：如果某行没有匹配到任何数字，extract 会返回 NaN。在进行 astype('float') 之前，Pandas 通常能处理好带有 NaN 的转换，但如果数据非常脏，建议先检查空值。

场景二：提取多个字段（生成 DataFrame）

当需要同时提取多个信息时，str.extract 会自动返回一个 DataFrame。

示例：从日志中提取时间和错误码
python
logs = pd.DataFrame({
    'log_msg': [
        '[ERROR] 2026-03-24 Time:10:05 Code:503',
        '[INFO] 2026-03-24 Time:10:06 Code:200',
        '[ERROR] 2026-03-24 Time:10:08 Code:404'
    ]
})

定义两个捕获组：时间 (d+:d+) 和 错误码 (d+)
pattern = r'Time:(d+:d+)s+Code:(d+)'

直接赋值给多列
logs = logs['log_msg'].str.extract(pattern)

转换类型
logs['error_code'] = logs['error_code'].astype(int)

print(logs)

输出结果：
text
                               log_msg    time  error_code
0  [ERROR] 2026-03-24 Time:10:05 Code:503  10:05         503
1   [INFO] 2026-03-24 Time:10:06 Code:200  10:06         200
2  [ERROR] 2026-03-24 Time:10:08 Code:404  10:08         404

场景三：处理更复杂的数字格式（负数与科学计数法）

如果您的数据中包含负数或科学计数法（如 1.2e-3），正则表达式需要调整。

通用数值正则：
(-?d+(?:.d+)?(?:[eE][+-]?d+)?)
-?: 可选的负号。
(?:...): 非捕获组，用于逻辑分组但不提取。

python
df = pd.DataFrame({'val': ['温度 -5.5 度', '误差 1.2e-3', '正常 300']})

提取复杂数值
df['num_val'] = df['val'].str.extract(r'(-?d+(?:.d+)?(?:[eE][+-]?d+)?)')[0].astype('float')

print(df)

常见陷阱与技巧

必须使用捕获组 ()：
   如果不加括号，str.extract 会报错或返回全 NaN，因为它不知道你要提取哪一部分。
   错误：r'd+'
   正确：r'(d+)'

处理无匹配项：
   如果没有匹配到内容，结果是 NaN。这在后续计算中可能会导致问题，可以使用 fillna(0) 填充。
   python
   study['clean_num'] = study['learn_process'].str.extract(r'(d+)')[0].fillna(0).astype(int)
   
性能优化：
   对于超大数据集，正则提取可能较慢。如果规则非常简单（如只取前3个字符），使用 str.slice() 或 str.split() 可能更快。

总结

str.extract() 是 Pandas 中连接非结构化文本与结构化数值分析的桥梁。掌握基础的正则语法（如 d, ., *, +, ? 以及捕获组 ()），可以解决 80% 以上的数据清洗难题。

记得在提取完成后，务必检查数据类型并使用 astype() 将其转换为所需的 int 或 float 类型，以便进行后续的统计分析。

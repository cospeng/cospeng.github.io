---
title: '理论力学'
date: 2026-03-05
permalink: /posts/2026/03/theoretical-mechanics/
tags:
  - 理论力学
  - 笔记
  - category2
---

# 理论力学

## 质点问题概述

#d 质点

无大小、形状，但具有质量的几何点

#d 质点化条件

尺寸大小可以忽略，形状可以忽略

#d 位移

$$
\Delta \vec{r} = \vec{r}_2 - \vec{r}_1
$$

#d 路程

$$
\Delta S
$$

#d 位移与路程的关系

$$
\Delta S \geq |\Delta \vec{r}|
$$
$$
dS = |\mathrm{d}\vec{r}|
$$

#d 速度

$$
\vec{v} = \frac{\mathrm{d}\vec{r}}{\mathrm{d}t} = \lim_{\Delta t \to 0} \frac{\Delta \vec{r}}{\Delta t} = \dot{\vec{r}}
$$

#d 加速度

$$
\vec{a} = \frac{\mathrm{d}\vec{v}}{\mathrm{d}t} = \lim_{\Delta t \to 0} \frac{\Delta \vec{v}}{\Delta t} = \dot{\vec{v}} = \ddot{\vec{r}}
$$

## 参考系与坐标系

#t 引入 为什么需要参考系

微分、积分的对象都是一个数，而位移、速度、加速度都是矢量，所以需要参考系和坐标系来描述它们

#d 参考系

被选作参考的物体或物体组（坐标原点）

#d 坐标系

描述质点位置的坐标轴系统，目的在于将矢量变为分量形式(数)，以便定量计算

#d 坐标系对基本矢量(基矢)的要求

完备、线性无关。完备指的是基矢的数量必须等于空间的维数；线性无关指的是基矢之间不能通过线性组合得到另一个基矢

#d 坐标系的选择原则

使问题的描述和计算尽可能简单

#d 理论力学坐标系

标准正交基、$\vec{e}_3 = \vec{e}_1 \times \vec{e}_2$右手系

#e 坐标系

对∀$\vec{A}$，存在唯一的数$A_1$、$A_2$、$A_3$使得$\vec{A} = A_1 \vec{e}_1 + A_2 \vec{e}_2 + A_3 \vec{e}_3 = \sum_{i=1}^{3} A_i \vec{e}_i$

$A_i = \vec{A} \cdot \vec{e}_i$

得$\vec{A} = \sum_{i=1}^{3} (\vec{A} \cdot \vec{e}_i) \vec{e}_i$

#e 笛卡尔坐标系

$\vec{e}_1 = \hat{i}, \vec{e}_2 = \hat{j}, \vec{e}_3 = \hat{k}$

$\vec{r} = \sum_{i=1}^{3} \vec{r} \vec{i} \vec{i} = \vec{r} \vec{i} \vec{i} + \vec{r} \vec{j} \vec{j} + \vec{r} \vec{k} \vec{k} = x \vec{i} + y \vec{j} + z \vec{k}$

$\vec{v} = \dot{\vec{r}} = \dot{x} \vec{i} + \dot{y} \vec{j} + \dot{z} \vec{k}$

$\vec{a} = \ddot{\vec{r}} = \ddot{x} \vec{i} + \ddot{y} \vec{j} + \ddot{z} \vec{k}$

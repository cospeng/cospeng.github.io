---
title: "Project Chrono 坐标系（Coordinate Systems）学习笔记"
date: 2026-03-08
permalink: /posts/2026/03/project-chrono-coordinate-systems/
tags:
  - Project Chrono
  - 多体动力学
  - 仿真
---


# 📝 Project Chrono 坐标系（Coordinate Systems）学习笔记

---

## 🌟 总体概览

平移和旋转，以及它们之间的变换，是 Chrono 库的核心要素。平移**只用向量**表示，旋转则推荐使用**四元数**，旋转矩阵偶尔用于性能优化场合。

Chrono 提供了一套层级递进的坐标系类：
- `ChCoordsys` — 最轻量的基础坐标系
- `ChFrame` — 最常用，内置旋转矩阵加速计算
- `ChFrameMoving` — 在 ChFrame 基础上增加速度和加速度信息
- `ChMarker` — 附着在刚体上随之运动的辅助坐标系


> 💡 **初学者建议**：优先掌握 `ChCoordsys` 和 `ChFrame` 这两个类就够了。

---

## 一、向量 `ChVector3`

3D 空间中的点由 `ChVector3` 表示：$\mathbf{p} = \{p_x, p_y, p_z\}$

```cpp
ChVector3d vect1(1, 2, 3);   // 构造三维向量
ChVector3d vect2(VECT_X);    // 使用预定义单位向量
```

**预定义常量：** `VNULL`（零向量）、`VECT_X`、`VECT_Y`、`VECT_Z`

访问分量用 `.x()`、`.y()`、`.z()`；支持 `+`、`-`、`*` 等运算符；也有以大写 `V` 开头的自由函数，如 `Vcross`（叉积）。由于 `ChVector3` 不继承自 Eigen，若需要 Eigen 功能需调用 `.eigen()` 转换。

---

## 二、四元数 `ChQuaternion`

### 📐 原理

Chrono 用四元数描述旋转，基于轴-角表示法：
$$\mathbf{q} = \left\{\cos(\theta/2),\ u_x\sin(\theta/2),\ u_y\sin(\theta/2),\ u_z\sin(\theta/2)\right\}$$
注意：**标量分量在第一位**。

> ⚠️ 只有**单位四元数**才表示有效旋转！无旋转的单位四元数 `QUNIT = (1, 0, 0, 0)`

### 🔨 构造方式

```cpp
ChQuaterniond q(1, 0, 0, 0);                      // 直接给分量
ChQuaterniond q = QuatFromAngleY(20 * CH_DEG_TO_RAD); // 绕Y轴旋转20度
// 预定义：Q_ROTATE_Z_TO_X, QUNIT 等
```

### 🔄 旋转操作

```cpp
ChVector3d vA(1, 2, 3);
ChQuaterniond q = QuatFromAngleY(20 * CH_DEG_TO_RAD);
ChVector3d vB = q.Rotate(vA);   // 将点 vA 绕Y轴旋转20度
```

**旋转的连接（先 qA 再 qB）：**
```cpp
qC = qB * qA;   // 右到左：先做 qA，再做 qB
qC = qA >> qB;  // 左到右：等价写法，更直观
```

---

## 三、旋转矩阵 `ChMatrix33`

旋转矩阵 $\mathbf{R} \in SO(3)$ 是正交矩阵，因此 $\mathbf{R}^{-1} = \mathbf{R}^T$。

```cpp
ChMatrix33d rotmA(1);           // 对角线值构造（如单位矩阵）
ChMatrix33d rotmB(quat);        // 从四元数构造
ChMatrix33d rotmC(angle, axis); // 从角度+轴构造
ChMatrix33d rotmD(vecX, vecY, vecZ); // 从列向量构造

ChMatrix33d rotmC = rotmB * rotmA; // 串联旋转（先A再B）
ChVector3d vB = rotm * vA;         // 旋转一个向量
```

---

## 四、坐标系 `ChCoordsys`

`ChCoordsys` 是最轻量的坐标系表示，包含一个**平移向量** $\mathbf{d}$ 和一个**旋转四元数** $\mathbf{q}$：
$$\mathbf{c} = \{\mathbf{d}, \mathbf{q}\}$$
它是 `ChFrame` 的精简版，适合不需要高级功能时节省内存。

---

## 五、坐标帧 `ChFrame` ⭐（最常用）

### 📦 是什么？

`ChFrame` 描述坐标系 **b** 相对于另一个坐标系 **a** 的旋转和平移关系。它在 `ChCoordsys` 的基础上额外存储了一个 3×3 旋转矩阵，当需要变换大量向量时可以提升性能。

### 🔨 构造方式

```cpp
ChFramed Xa;                     // 默认：零平移，无旋转
ChFramed Xb(vec, quat);          // 从平移向量 + 四元数
ChFramed Xc(csys);               // 从 ChCoordsys 构造
ChFramed Xd(vec, theta, u);      // 从平移向量 + 绕轴旋转角
```

### 🔄 三种变换类型

| 变换类型 | 公式 | 方法名 |
|----------|------|--------|
| **方向变换**（只旋转，不平移） | $\mathbf{d}_{Pa(a)} = \mathbf{R}_{ba}\,\mathbf{d}_{Pb(b)}$ | `TransformDirectionLocalToParent()` |
| **位置变换**（旋转 + 平移） | $\mathbf{d}_{Pa(a)} = \mathbf{d}_{ba(a)} + \mathbf{R}_{ba}\,\mathbf{d}_{Pb(b)}$ | `TransformPointLocalToParent()` |
| **扳手变换**（力+力矩一起变换） | — | `TransformWrenchLocalToParent()` |

### ✍️ 简洁写法：`*` 和 `>>` 运算符

**不用 ChFrame（繁琐）：**
```cpp
d_Pa_a = d_ba_a + R_ba * d_Pb_b;
```

**用 ChFrame（简洁）：**
```cpp
d_Pa_a = X_ba * d_Pb_b;     // * 运算符（右到左）
d_Pa_a = d_Pb_b >> X_ba;    // >> 运算符（左到右）
```

### 🔗 链式变换（串联多个坐标系）

```cpp
// c→b→a 的链式变换
X_ca = X_ba * X_cb;   // 右到左
X_ca = X_cb >> X_ba;  // 左到右（更直观，下标顺序连贯）
```

> 💡 **推荐用 `>>` 运算符**：
> - 下标读起来像链条 `cb → ba`，更自然
> - C++ 从左到右执行，临时对象更少，性能更好

### 🔁 逆变换

```cpp
// 已知 X_ca 和 X_cb，求 X_ba
X_ba = X_cb.GetInverse() >> X_ca;
// 或用更高效的底层方法：
X_ba = X_cb.TransformParentToLocal(X_ca);
```

---

## 六、运动坐标帧 `ChFrameMoving`

`ChFrameMoving` 在 `ChFrame` 的基础上额外存储速度和加速度信息：
$$\mathbf{c} = \{\mathbf{p},\, \mathbf{q},\, \dot{\mathbf{p}},\, \boldsymbol{\omega},\, \ddot{\mathbf{p}},\, \boldsymbol{\alpha}\}$$
角速度 $\boldsymbol{\omega}$ 和角加速度 $\boldsymbol{\alpha}$ 既可在**局部坐标系**中设置，也可在**父坐标系**中设置。

```cpp
ChFrameMoving<> X_ba;
X_ba.SetPos(ChVector3d(2, 3, 5));
X_ba.SetRot(myquaternion);

// 设置速度
X_ba.SetPos_dt(ChVector3d(100, 20, 53));       // 线速度
X_ba.SetWvel_loc(ChVector3d(0, 40, 0));        // 角速度（局部系）
X_ba.SetWvel_par(ChVector3d(0, 40, 0));        // 角速度（父坐标系）

// 设置加速度
X_ba.SetPos_dtdt(ChVector3d(13, 16, 22));      // 线加速度
X_ba.SetWacc_loc(ChVector3d(80, 50, 0));       // 角加速度（局部系）
```

> ✅ 链式变换时，**科氏加速度、向心加速度等复杂项**会被自动计算处理，无需手动推导！

**机器人手臂典型例子（求 X_86）：**
```cpp
X_86 = X_87 >> X_70 >> (X_65 >> X_54 >> X_43 >> X_32 >> X_21 >> X_10).GetInverse();
```

---

## 七、标记点 `ChMarker`

`ChMarker` 是附着在**刚体**上的辅助坐标帧，会随刚体运动，同时也可相对于刚体运动。它最常用于 `ChLinkLock` 系列约束连接中。

---

## 📊 类层级总结

```
ChCoordsys          → 最轻量：平移向量 + 四元数
    ↑
  ChFrame            → 最常用：+旋转矩阵（加速变换）
    ↑
  ChFrameMoving      → +速度 + 加速度（含角速度/角加速度）
    ↑
  ChMarker           → 附着在刚体上的运动帧
```

| 类 | 包含内容 | 适用场景 |
|----|----------|----------|
| `ChCoordsys` | 位置 + 四元数 | 轻量存储，简单变换 |
| `ChFrame` | 位置 + 四元数 + 旋转矩阵 | 日常使用（推荐） |
| `ChFrameMoving` | 上面 + 速度 + 加速度 | 运动学分析 |
| `ChMarker` | ChFrameMoving + 绑定刚体 | 约束连接（关节） |

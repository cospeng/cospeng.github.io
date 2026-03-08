---
title: "Project Chrono 数学对象（Mathematical Objects）学习笔记"
date: 2026-03-08
permalink: /posts/2026/03/project-chrono-loads/
tags:
  - Project Chrono
  - 多体动力学
  - 仿真
---

# 📝 Project Chrono 数学对象（Mathematical Objects）学习笔记

---

## 一、线性代数（Linear Algebra）

### 🔧 底层依赖

Chrono 使用 **Eigen3** 来表示所有矩阵（稠密和稀疏）及向量。稠密矩阵按**标量类型**模板化，采用**行主序**存储。所有 Chrono 矩阵和向量类型本质上都是 Eigen 矩阵类型的别名。

> 📌 特例：`ChVector` 和 `ChVector2` 不是 Eigen 类型，是 Chrono 独立实现的 3D/2D 向量。

矩阵**从 0 开始索引**，格式为 `(行, 列)`。

---

### 📦 矩阵与向量类型速查表

| 类型 | 说明 | 示例 |
|------|------|------|
| `ChMatrixDynamic<>` | 动态大小矩阵（运行时确定） | `ChMatrixDynamic<double> A(12, 4);` |
| `ChMatrixNM<T,M,N>` | 固定大小矩阵（编译期确定） | `ChMatrixNM<double,4,4> B;` |
| `ChMatrix33<>` | 3×3 固定矩阵，专用于旋转矩阵/惯性张量 | `ChMatrix33<> R;` |
| `ChVectorDynamic<>` | 动态列向量 | `ChVectorDynamic<double> v(12);` |
| `ChVectorN<T,N>` | 固定长度列向量 | `ChVectorN<double,6> w;` |
| `ChRowVectorDynamic<>` | 动态行向量 | — |
| `ChRowVectorN<T,N>` | 固定行向量 | — |

> 💡 **固定 vs 动态怎么选？**
> - 尺寸 < 16 → 用固定大小（性能更好，避免内存分配，支持循环展开）
> - 尺寸 > 32 → 用动态大小（固定大小优势消失，栈空间可能溢出）

---

### 🧮 矩阵常用操作

```cpp
Md1.setRandom();         // 随机填充
Md1.setZero();           // 全零
Md2.setIdentity();       // 单位矩阵
Md1.transposeInPlace();  // 原地转置
Md1.resize(2, 2);        // 调整大小

// 运算符
result = A + B;    // 加法
result = A - B;    // 减法
result = A * 10;   // 标量乘
result = C * D;    // 矩阵乘法
result = A.transpose(); // 转置

// 元素访问
Md2(0, 0) = 10;

// 误差比较
A.equals(B, 0.002);  // 判断两矩阵是否在容差内相等
```

---

### 🔄 坐标变换（ChTransform）

用于在**局部坐标系**和**父坐标系**之间转换点的位置：

```cpp
ChVector<> vl(2, 3, 4);    // 局部点
ChVector<> t(5, 6, 7);     // 平移
ChQuaternion<> q(...);     // 旋转（四元数）
q.Normalize();

// 局部 → 父坐标（以下四种方式等价）
auto va1 = ChTransform<>::TransformLocalToParent(vl, t, R);
auto va2 = ChTransform<>::TransformLocalToParent(vl, t, q);
auto va3 = csys.TransformLocalToParent(vl);
auto va4 = t + R * vl;

// 父坐标 → 局部（逆变换）
auto vl1 = ChTransform<>::TransformParentToLocal(va1, t, q);
```

---

### 📐 线性方程组求解

```cpp
ChMatrixNM<double, 3, 3> A;
Eigen::Vector3d b;
A << 1, 2, 3, 4, 5, 6, 7, 8, 10;
b << 3, 3, 4;

// 用 QR 分解求解 Ax = b
Eigen::Vector3d x = A.colPivHouseholderQr().solve(b);
```

---

## 二、函数对象（ChFunction）

### 💡 是什么？

`ChFunction` 对象用于表示 `y = f(x)` 的标量函数（输入输出均为实数），在 Chrono 很多地方都有应用，例如线性驱动器中的预设位移。

Chrono 内置了很多常用函数（正弦、余弦、常数、斜坡等），如果不够用，还可以**自定义**。

---

### 📖 使用示例

**斜坡函数（Ramp）：**
```cpp
ChFunction_Ramp f_ramp;
f_ramp.Set_ang(0.1);   // 斜率
f_ramp.Set_y0(0.4);    // x=0 时的初始值

double y    = f_ramp.Get_y(10);     // 求 f(10) 的值
double ydx  = f_ramp.Get_y_dx(10); // 求 f'(10) 的导数值
```

**正弦函数（Sine）：**
```cpp
ChFunction_Sine f_sine;
f_sine.Set_amp(2);     // 振幅
f_sine.Set_freq(1.5);  // 频率
```

**自定义函数：**
```cpp
class ChFunction_MyTest : public ChFunction {
public:
    ChFunction* new_Duplicate() { return new ChFunction_MyTest; }
    double Get_y(double x) { return cos(x); }  // 自定义公式
    // 可选：覆盖 Get_y_dx() 提供解析导数（默认用数值微分）
};
```

> ✅ 至少实现 `Get_y()` 即可，导数默认会用数值微分自动计算。

---

## 三、数值积分（Quadrature）

### 💡 是什么？

积分（Quadrature）用于计算函数的面积或体积。Chrono 支持 **Gauss-Legendre 数值积分**，可对 1D、2D、3D 区间上的函数进行积分。

> ⚠️ 若被积函数是 N 次多项式，N 阶积分结果精确；否则为近似值。一般 N 取 1~10 即可满足大多数需求。N < 10 时使用预计算系数，性能最优。

---

### 📖 使用示例

**1D 积分（求 sin(x) 在 [0, π] 上的积分）：**
```cpp
class MySine1d : public ChIntegrable1D<double> {
public:
    void Evaluate(double& result, const double x) {
        result = sin(x);
    }
};

MySine1d mfx;
double qresult;
// 6阶 Gauss-Legendre 积分，区间 [0, π]
ChQuadrature::Integrate1D<double>(qresult, mfx, 0, CH_C_PI, 6);
// 解析结果为 2.0
```

**2D 积分：**
```cpp
ChQuadrature::Integrate2D<double>(qresult, mfx2d, 0, CH_C_PI, -1, 1, 6);
```

**向量值函数积分（模板化）：**
```cpp
// 积分结果是一个 2x1 向量
class MySine2dM : public ChIntegrable2D<ChMatrixNM<double,2,1>> {
    void Evaluate(ChMatrixNM<double,2,1>& result, const double x, const double y) {
        result(0) = x * y;
        result(1) = 0.5 * y * y;
    }
};
```

---

## 📊 总结一览

| 模块 | 核心类 | 用途 |
|------|--------|------|
| 线性代数 | `ChMatrixDynamic`, `ChMatrixNM`, `ChMatrix33`, `ChVectorDynamic` | 矩阵/向量运算，底层用 Eigen |
| 坐标变换 | `ChTransform`, `ChQuaternion`, `ChMatrix33` | 局部↔世界坐标系转换 |
| 函数对象 | `ChFunction` 及其子类 | 表示随时间变化的标量函数 |
| 数值积分 | `ChQuadrature` | Gauss-Legendre 1D/2D/3D 积分 |

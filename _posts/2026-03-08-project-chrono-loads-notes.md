---
title: "Project Chrono 载荷 (Loads) 学习笔记"
date: 2024-05-20
permalink: /posts/2024/05/project-chrono-loads/
tags:
  - Project Chrono
  - 多体动力学
  - 仿真
---

# 📝 Project Chrono 载荷（Loads）学习笔记

## 🌟 什么是载荷？

在 Project Chrono 中，**载荷（Load）** 就是施加在物体上的力或力矩。能承受载荷的对象必须继承自 `ChLoadable` 类，常见的有：

- `ChBody`（刚体）
- 有限元节点（FEA nodes）
- `ChShaft`（轴）

---

## 🗂️ 三大施力方式

### 方式一：`ChForce` —— 最简单，但只能用于刚体

**`ChForce` 只能用于 `ChBody`**，用来施加力或力矩，参考系可以是物体自身坐标系或世界坐标系。

**使用方法：**
```cpp
auto force = chrono_types::make_shared<ChForce>();
body->AddForce(force);      // ✅ 正确写法
force->SetMforce(10);       // 设置力的大小（必须在 AddForce 之后调用）
```

> ⚠️ **注意事项：**
> - 必须用 `body->AddForce(force)`，**不能**用 `force->SetBody(body)`，否则系统不会识别这个力！
> - `ChForce` 的方法必须在 `AddForce()` **之后**才能调用。

**可以设置什么？**
- 施力点位置
- 力的方向
- 力的大小（可以是常数，也可以用 `ChFunction` 设置时变函数）

---

### 方式二：`ChLoad` + `ChLoader` —— 更灵活，适用于所有可载荷对象

这类载荷适用于**任何** `ChLoadable` 的子类对象，与 Chrono 系统的耦合更紧密，但需要借助一个 **`ChLoadContainer`（载荷容器）** 才能加入系统。

**使用方法：**
```cpp
// 1. 创建载荷容器并加入系统
auto load_container = chrono_types::make_shared<ChLoadContainer>();
sys.Add(load_container);

// 2. 创建具体载荷并加入容器
auto load_bb = chrono_types::make_shared<ChLoadBodyBodyTorque>(
    bodyA, bodyB, ChVector3d(0, 10.0, 0), false);
load_container->Add(load_bb);
```

**使用场景：**
- 需要精确控制广义载荷时
- 需要与求解器深度集成（如提供雅可比矩阵）
- 分布式载荷（通过 Gauss 积分自动处理）

---

### 方式三：`ChLoadCustom` / `ChLoadCustomMultiple` —— 最强大，支持多物体

这类载荷可以施加在**一个或多个** `ChLoadable` 对象上，特别适合两个物体之间的交互载荷。它提供了更多预定义类，并支持引入完整的刚度矩阵块（如 `ChLoadBodyBodyBushingGeneric`）和雅可比矩阵。

---

## 🔧 其他补充工具

| 工具 | 说明 |
|------|------|
| `ChLinkTSDA` / `ChLinkRSDA` | 弹簧-阻尼连接，可提供雅可比，甚至内置 ODE |
| `AccumulateForce()` / `AccumulateTorque()` | 快速给 `ChBody` 叠加力/力矩（简化写法） |
| `GetAppliedForce()` | 读取 `ChBody` 上当前的合力 |
| `ChNodeFEAxyz::SetForce()` | 给 FEA 节点设置常值力（只能是绝对坐标系下的常向量） |

---

## 📊 方式对比总结

| | `ChForce` | `ChLoad+ChLoader` | `ChLoadCustom` |
|---|---|---|---|
| 适用对象 | 仅 ChBody | 所有 ChLoadable | 所有 ChLoadable |
| 使用难度 | ⭐ 简单 | ⭐⭐⭐ 复杂 | ⭐⭐⭐ 复杂 |
| 灵活性 | 低 | 高 | 最高 |
| 支持多物体 | ❌ | ❌ | ✅ |
| 需要 ChLoadContainer | ❌ | ✅ | ✅ |
| 支持雅可比矩阵 | ❌ | ✅ | ✅ |

---

## 💡 选择建议

- **刚体简单施力** → 用 `ChForce`，最省事
- **需要作用于 FEA/节点** → 用 `ChLoad + ChLoader`
- **两个物体之间的相互作用**（如衬套、弹簧）→ 用 `ChLoadCustomMultiple`
- **查完整类层级** → 参考 `ChLoadBase`

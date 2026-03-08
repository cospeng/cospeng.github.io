---
title: "Project Chrono 仿真系统 (Simulation System) 学习笔记"
date: 2026-03-08
permalink: /posts/2026/03/project-chrono-simulation-system/
tags:
  - Project Chrono
  - 多体动力学
  - 仿真
---

# 📝 Project Chrono 仿真系统（Simulation System）学习笔记

---

## 🌟 总体概览

`ChSystem` 是整个 Chrono 仿真的**核心容器**，所有参与仿真的对象——刚体、约束、载荷、网格等——都归属于它。没有 `ChSystem`，什么都运行不起来。

`ChSystem` 是抽象类，不能直接实例化，必须选择以下两种子类之一：

| 类名 | 接触处理方式 | 适用场景 |
|------|------------|----------|
| `ChSystemNSC` | 非光滑接触（互补约束） | 有刚性接触/碰撞；大时间步长仍高效 |
| `ChSystemSMC` | 光滑接触（罚函数弹簧） | 接触可变形；FEA 仿真必须用此类 |

> 💡 如果系统**没有接触/碰撞**，两者均可，随意选择。

---

## 一、ChSystem 基本使用

### 创建系统并添加对象

```cpp
// 创建一个 NSC（非光滑接触）系统
ChSystemNSC my_system;

// 创建刚体并加入系统
auto body = chrono_types::make_shared<ChBody>();
my_system.Add(body);

// 创建约束并加入系统
auto link = chrono_types::make_shared<ChLinkLockRevolute>();
my_system.Add(link);

// 运行一步动力学仿真（步长 0.01 秒）
my_system.DoStepDynamics(0.01);
```

> 💡 使用 `Add()` / `Remove()` 管理系统中的所有物理对象。

### 常见仿真主循环结构

```cpp
ChSystemNSC my_system;
// ... 添加刚体、约束、载荷 ...

double time_end = 5.0;    // 总仿真时长（秒）
double time_step = 0.01;  // 每步时间步长

while (my_system.GetChTime() < time_end) {
    my_system.DoStepDynamics(time_step);
    // 可在此处读取状态、写入日志等
}
```

### 三大需要调整的参数

运行一个仿真，通常需要调节以下三类参数：

1. **时间积分器（Time Stepper）**：选择积分算法和步长
2. **求解器（Solver）**：选择约束/加速度求解算法
3. **碰撞参数（Collision Parameters）**：穿透恢复速度和反弹速度阈值

---

## 二、时间积分器（Time Steppers）

### 是什么？

时间积分器负责把仿真状态从 $t$ 推进到 $t + \Delta t$，本质上是对运动方程做数值积分。

### 如何设置？

```cpp
// 方式一（推荐）：直接设置类型
my_system.SetTimestepperType(ChTimestepper::Type::EULER_IMPLICIT_LINEARIZED);

// 方式二：插入自定义积分器实例
auto my_stepper = chrono_types::make_shared<ChTimestepperHHT>(&my_system);
my_system.SetTimestepper(my_stepper);
```

### 三种常用积分器对比

#### `EULER_IMPLICIT_LINEARIZED`（默认，最常用）

**描述：** Chrono 的默认积分器，线性化隐式欧拉法。速度快，无需内部迭代，适合大多数日常仿真任务。

- ✅ 速度最快，无子迭代
- ✅ 支持 DVI 硬接触（NSC 系统）
- ✅ FEA 可用（一阶精度）
- ⚠️ 一阶精度（精度相对较低）
- 约束靠**稳定化**保持闭合

```cpp
// 设置默认积分器（其实可以不写，这是默认行为）
my_system.SetTimestepperType(ChTimestepper::Type::EULER_IMPLICIT_LINEARIZED);
```

---

#### `HHT`（高精度首选）

**描述：** 基于 Hilber-Hughes-Taylor 公式的隐式积分器，有数值阻尼调节能力，FEA 二阶精度，约束严格满足。适合需要高精度的 FEA 仿真，但**不能用于硬接触（NSC）系统**。

- ✅ 二阶精度，精度高
- ✅ FEA 仿真首选
- ✅ 约束通过内迭代**精确**满足
- ✅ 可调数值阻尼（`SetAlpha`）
- ❌ 不支持 DVI 硬接触（不能用于 NSC 系统）
- ⚠️ 速度比 EULER_IMPLICIT_LINEARIZED 慢，需要子迭代

```cpp
// 切换到 HHT 积分器
my_system.SetTimestepperType(ChTimestepper::Type::HHT);

// 进一步调节 HHT 参数（如数值阻尼）
if (auto mystepper = std::dynamic_pointer_cast<ChTimestepperHHT>(
        my_system.GetTimestepper())) {
    mystepper->SetAlpha(-0.2);   // 范围 [-1/3, 0]，越小阻尼越大
    mystepper->SetMaxiters(10);  // 最大子迭代次数
    mystepper->SetAbsTolerances(1e-4);
}
```

---

#### `NEWMARK`

**描述：** FEA 社区常用的经典积分器，性质与 HHT 类似。大多数参数选择下为一阶精度，仅在特殊参数（梯形积分规则）下才达到二阶精度。

- ✅ FEA 领域广泛使用
- ⚠️ 通常为一阶精度（特殊参数下二阶）

```cpp
my_system.SetTimestepperType(ChTimestepper::Type::NEWMARK);
```

### 积分器选型速查

| 场景 | 推荐积分器 |
|------|-----------|
| 普通多体+碰撞（NSC） | `EULER_IMPLICIT_LINEARIZED`（默认） |
| 高精度 FEA（无硬接触） | `HHT` |
| FEA 传统工程仿真 | `NEWMARK` |

---

## 三、求解器（Solvers）

### 是什么？

求解器在每个时间步内被积分器调用，用于求解**未知加速度**和**约束反力**。它通常是仿真中**最大的计算瓶颈**。

### 如何设置？

```cpp
// 方式一（推荐）：直接指定类型
my_system.SetSolverType(ChSolver::Type::BARZILAIBORWEIN);

// 方式二：插入自定义求解器实例（灵活性最高）
auto solver = chrono_types::make_shared<ChSolverMINRES>();
my_system.SetSolver(solver);
```

### 迭代次数设置（使用迭代求解器时必须注意！）

```cpp
// 对于含约束的系统，强烈建议增大迭代次数
my_system.GetSolver()->AsIterative()->SetMaxIterations(400);
```

> ⚠️ 如果约束出现"松弛/穿透"，通常是迭代次数不够导致的，应先尝试增大此值。

### 各求解器对比

#### `PSOR`（最简单，入门常用）

**描述：** 投影 SOR（逐次超松弛）迭代求解器，计算速度快但精度较低。适合对精度要求不高的小规模问题。

- ✅ 支持 DVI 硬接触（NSC）
- ✅ 速度快，实现简单
- ❌ 精度低，收敛可能停滞（尤其质量比悬殊时）

```cpp
my_system.SetSolverType(ChSolver::Type::PSOR);
my_system.GetSolver()->AsIterative()->SetMaxIterations(200);
```

---

#### `APGD`（高精度首选迭代求解器）

**描述：** 加速投影梯度下降法，收敛性很好，适合需要高精度的仿真。

- ✅ 收敛性好
- ✅ 支持 DVI 硬接触（NSC）
- ✅ 高精度场景首选

```cpp
my_system.SetSolverType(ChSolver::Type::APGD);
my_system.GetSolver()->AsIterative()->SetMaxIterations(400);
```

---

#### `BARZILAIBORWEIN`（稳健，大质量比场景推荐）

**描述：** 与 APGD 类似，但在质量比悬殊（如轻零件与重零件同时存在）时更稳健。

- ✅ 收敛性好
- ✅ 支持 DVI 硬接触（NSC）
- ✅ 大质量比场景比 APGD 更稳健

```cpp
my_system.SetSolverType(ChSolver::Type::BARZILAIBORWEIN);
```

---

#### `MINRES`（FEA 专用迭代求解器）

**描述：** 最小残差法，适合包含刚度矩阵的 FEA 问题，但**不支持 DVI 硬接触**。

- ✅ 适合 FEA 问题（含刚度矩阵）
- ❌ 不支持 DVI 硬接触（NSC）

```cpp
my_system.SetSolverType(ChSolver::Type::MINRES);

// 可以开启对角预条件，提升收敛
if (auto msolver = std::dynamic_pointer_cast<ChSolverMINRES>(
        my_system.GetSolver())) {
    msolver->SetDiagonalPreconditioning(true);
}
```

---

#### `ADMM + PardisoMKL`（最强，支持 FEA + 接触）

**描述：** 交替方向乘子法（ADMM），同时支持 FEA 和 DVI 硬接触问题，是目前功能最全面的求解器组合。需要内部线性求解器，首选 PardisoMKL（需单独安装 PARDISO_MKL 模块）。

- ✅ 同时支持 FEA 和 DVI 硬接触
- ✅ 精度高
- ⚠️ 需要安装 PARDISO_MKL 模块（否则退化为 SparseQR）

```cpp
auto solver = chrono_types::make_shared<ChSolverADMM>();
my_system.SetSolver(solver);
```

### 求解器选型速查

| 场景 | 推荐求解器 |
|------|-----------|
| 快速验证，精度要求低 | `PSOR` |
| 多体+硬接触，高精度 | `APGD` 或 `BARZILAIBORWEIN` |
| 质量比悬殊场景 | `BARZILAIBORWEIN` |
| FEA 无接触 | `MINRES` |
| FEA + 接触，最高精度 | `ADMM + PardisoMKL` |

---

## 四、其他仿真参数

### 最大穿透恢复速度（Max Recovery Speed）

**描述：** 当两物体因数值误差或不一致初始条件而发生相互穿透时，系统会尝试将其推开，但推开的速度不超过此阈值。

```cpp
my_system.SetMaxPenetrationRecoverySpeed(0.2);  // 默认值，单位 m/s
```

**调节原则：**
- 值**偏大**→ 穿透修正更激进，但可能导致物体弹出或堆叠不稳
- 值**偏小**→ 修正温和，但积分精度低时物体可能持续"下沉"互相穿透

**实际建议：** 对于堆叠稳定性要求高的场景（如沙堆、零件堆放），适当减小此值。

---

### 最小反弹速度（Min Bounce Speed）

**描述：** 碰撞时，如果接触物体的接近速度低于此阈值，则恢复系数自动视为 0（完全非弹性碰撞），有助于堆叠物体的稳定。

```cpp
my_system.SetMinBounceSpeed(0.1);  // 默认值，单位 m/s
```

**调节原则：**
- 值**偏大**→ 更多碰撞被视为非弹性，堆叠更稳定，但碰撞物理真实性下降
- 值**偏小**→ 碰撞更真实，但需要更小的时间步长，否则低速接触可能持续震荡

---

## 五、完整配置示例

以下是一个涵盖上述所有参数配置的完整示例：

```cpp
// 1. 创建仿真系统（NSC 用于含硬接触的多体问题）
ChSystemNSC my_system;

// 2. 添加物体和约束
auto bodyA = chrono_types::make_shared<ChBody>();
bodyA->SetPos(ChVector3d(0, 1, 0));
bodyA->SetMass(1.0);
my_system.Add(bodyA);

// 3. 选择时间积分器
my_system.SetTimestepperType(ChTimestepper::Type::EULER_IMPLICIT_LINEARIZED);

// 4. 选择求解器
my_system.SetSolverType(ChSolver::Type::BARZILAIBORWEIN);
my_system.GetSolver()->AsIterative()->SetMaxIterations(400);

// 5. 设置碰撞参数
my_system.SetMaxPenetrationRecoverySpeed(0.2);
my_system.SetMinBounceSpeed(0.1);

// 6. 运行仿真主循环
double time_end = 3.0;
double time_step = 0.005;

while (my_system.GetChTime() < time_end) {
    my_system.DoStepDynamics(time_step);
}
```

---

## 六、参数组合推荐方案

| 仿真类型 | 系统类 | 积分器 | 求解器 |
|----------|--------|--------|--------|
| 刚体多体+碰撞（快速） | `ChSystemNSC` | `EULER_IMPLICIT_LINEARIZED` | `PSOR` |
| 刚体多体+碰撞（高精度） | `ChSystemNSC` | `EULER_IMPLICIT_LINEARIZED` | `APGD` 或 `BARZILAIBORWEIN` |
| FEA 无接触 | `ChSystemSMC` | `HHT` | `MINRES` |
| FEA + 软接触 | `ChSystemSMC` | `HHT` | `MINRES` 或 `ADMM` |
| FEA + 硬接触（最强） | `ChSystemSMC`/`NSC` | `EULER_IMPLICIT_LINEARIZED` | `ADMM + PardisoMKL` |

---

## 七、调参 Checklist（排查仿真不稳定问题）

- [ ] 约束/铰链在仿真中出现"松弛"或"抖动"？→ 增大求解器迭代次数
- [ ] 物体之间互相穿透？→ 增大 `MaxPenetrationRecoverySpeed` 或减小时间步长
- [ ] 堆叠物体不停震荡？→ 增大 `MinBounceSpeed` 或增大 `MaxPenetrationRecoverySpeed`
- [ ] FEA 结果精度不够？→ 换用 `HHT` 积分器，减小时间步长
- [ ] 仿真太慢？→ 换用 `EULER_IMPLICIT_LINEARIZED` + `PSOR`，或增大时间步长

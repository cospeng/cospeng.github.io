---
title: "Project Chrono 约束连接 (Links) 学习笔记"
date: 2026-03-08
permalink: /posts/2026/03/project-chrono-links/
tags:
  - Project Chrono
  - 多体动力学
  - 仿真
---

# 📝 Project Chrono 约束连接（Links）学习笔记

---

## 🌟 总体概览

在 Chrono 中，**Link（连接/约束）** 用于限制两个刚体之间的相对运动。例如，转动铰让两个物体只能相对旋转、球铰只允许旋转不允许相对移动，等等。

所有约束类都继承自 `ChLink`，主要分为三大家族：

| 家族 | 代表类 | 特点 |
|------|--------|------|
| `ChLinkMate` | `ChLinkMateRevolute` 等 | **性能更高**，但不支持运动限位，仅少数支持驱动 |
| `ChLinkLock` | `ChLinkLockRevolute` 等 | **功能更全**，支持限位、力和位移读取、标记点可移动 |
| `ChLinkMotor` | `ChLinkMotorRotation` 等 | 继承自 ChLinkMate，**内置驱动功能** |

> 💡 **选型原则**：`ChLinkMate` 和 `ChLinkLock` 功能有重叠，优先选 `ChLinkMate`（效率更高）；若需要限位或详细的力/位移读取，才选 `ChLinkLock`。

---

## 一、使用约束的通用流程

约束通常指向一对 `ChMarker`，但在初始化时会自动创建并附加到刚体上，无需手动管理标记点。每个约束都有一个参考/主坐标帧，反力和轴方向均相对于该参考标记计算。

### 标准三步法

```cpp
// 第 1 步：创建约束对象
auto mylink = chrono_types::make_shared<ChLinkLockSpherical>();

// 第 2 步：初始化——指定两个被约束的刚体，以及约束坐标帧的位置和方向
mylink->Initialize(
    pendulumBody,    // 第一个刚体
    floorBody,       // 第二个刚体
    ChFramed(
        ChVector3d(1, 0, 0),                    // 约束点在世界坐标系的位置
        QuatFromAngleAxis(-CH_PI / 2, VECT_X)   // 约束坐标系的旋转
    )
);

// 第 3 步：加入仿真系统
my_system.Add(mylink);

// 第 4 步（可选）：设置约束属性，例如限位、弹簧参数等
```

> ⚠️ **注意事项：**
> - 初始化时，建议先将刚体摆放到**合理（可行）的初始位置**，否则仿真开始时系统需要额外力量来满足约束，可能导致不稳定。
> - 含有大量约束时，建议增大求解器的最大迭代次数（参见仿真系统笔记）。

---

## 二、约束连接速查表（按约束自由度分类）

> **ConDOF** = 被约束的自由度数（共 6 个 DOF：3 平移 + 3 旋转）。ConDOF 越大，约束越强，剩余运动自由度越少。

### 🔒 完全固定（ConDOF = 6）

| 类名 | 描述 |
|------|------|
| `ChLinkMateFix` | 完全固定位置和旋转（推荐） |
| `ChLinkLockLock` | 同上，ChLinkLock 版本，支持施加驱动运动 |

```cpp
// 示例：将零件固定到地面
auto fix = chrono_types::make_shared<ChLinkMateFix>();
fix->Initialize(part, ground, ChFramed(ChVector3d(0, 0, 0)));
my_system.Add(fix);
```

> 💡 若只需要固定某个刚体（不关心反力），更简单的方式是直接调用 `body->SetFixed(true)`，这会从方程组中完全移除该刚体的自由度，效率最高。

---

### 🔩 衬套连接（ConDOF = 6 或 3）

| 类名 | 描述 |
|------|------|
| `ChLinkBushing` | 线性柔性连接，可选配球铰 |

**描述：** 衬套（Bushing）是弹性连接，不像刚性约束那样完全锁死，而是允许在弹性范围内的微小相对运动。常用于模拟橡胶衬套、悬架衬套等。

```cpp
auto bushing = chrono_types::make_shared<ChLinkBushing>(
    ChLinkBushing::Spherical);   // 或 ChLinkBushing::Mount（完全约束）
bushing->Initialize(bodyA, bodyB, ChFramed(ChVector3d(0.5, 0, 0)));
// 设置弹性刚度（6x6 矩阵或简化参数）
my_system.Add(bushing);
```

---

### 🔄 转动铰（ConDOF = 5，剩余 1 个旋转自由度）

| 类名 | 说明 |
|------|------|
| `ChLinkMateRevolute` | **推荐**，绕 Z 轴旋转 |
| `ChLinkLockRevolute` | 功能版，支持限位和力读取 |
| `ChLinkRevolute` | 另一种实现 |

**描述：** 最常用的约束之一。允许两个刚体绕约束坐标系的 **Z 轴**相对旋转，其余 5 个自由度全部锁死。典型应用：门铰链、曲柄连杆、摆臂等。

```cpp
// 示例：摆锤绕轴旋转
auto rev = chrono_types::make_shared<ChLinkMateRevolute>();
rev->Initialize(
    pendulum,      // 摆锤
    support,       // 支座
    ChFramed(
        ChVector3d(0, 1, 0),          // 铰链点位置
        QuatFromAngleAxis(0, VECT_X)  // 默认 Z 轴即旋转轴
    )
);
my_system.Add(rev);
```

> 💡 约束的旋转轴是约束坐标系的 **Z 轴**。若需要绕其他轴旋转，需调整初始化时传入的旋转四元数。

---

### ➡️ 移动副（ConDOF = 5，剩余 1 个平移自由度）

| 类名 | 说明 |
|------|------|
| `ChLinkMatePrismatic` | **推荐**，沿 Z 轴平移 |
| `ChLinkLockPrismatic` | 功能版，支持限位 |

**描述：** 允许两个刚体沿约束坐标系的 **Z 轴**相对平移，旋转和其他方向的平移全部锁死。典型应用：活塞、滑块、液压缸等。

```cpp
// 示例：活塞沿轴运动
auto prismatic = chrono_types::make_shared<ChLinkMatePrismatic>();
prismatic->Initialize(
    piston, cylinder,
    ChFramed(ChVector3d(0, 0, 0))  // 滑动方向为 Z 轴
);
my_system.Add(prismatic);
```

---

### ✚ 万向节（ConDOF = 4）

| 类名 | 说明 |
|------|------|
| `ChLinkUniversal` | 沿 X 和 Y 轴的万向节 |

**描述：** 万向节（十字轴节）允许绕两个轴（X 和 Y 轴）旋转，常用于传动轴的角度补偿。

```cpp
auto universal = chrono_types::make_shared<ChLinkUniversal>();
universal->Initialize(shaftA, shaftB,
    ChFramed(ChVector3d(0, 0, 0)));
my_system.Add(universal);
```

---

### 🔁 柱面副（ConDOF = 4，剩余旋转 + 平移各 1）

| 类名 | 说明 |
|------|------|
| `ChLinkMateCylindrical` | **推荐**，Z 轴共线 |
| `ChLinkLockCylindrical` | 功能版 |

**描述：** 允许绕 Z 轴旋转**且**沿 Z 轴平移，两轴保持共线。可理解为"转动副 + 移动副"的组合。典型应用：螺旋副（配合驱动时）、导向轴。

```cpp
auto cyl = chrono_types::make_shared<ChLinkMateCylindrical>();
cyl->Initialize(bodyA, bodyB,
    ChFramed(ChVector3d(0, 0, 0)));
my_system.Add(cyl);
```

---

### ⚽ 球铰（ConDOF = 3，锁定全部平移）

| 类名 | 说明 |
|------|------|
| `ChLinkMateSpherical` | **推荐** |
| `ChLinkLockSpherical` | 功能版 |

**描述：** 锁定三个平移自由度，三个旋转自由度全部保留。两个物体可以相对绕任意方向旋转，但连接点不能相对移动。典型应用：球形关节、车辆球头等。

```cpp
// 示例：四连杆机构中的球铰
auto spherical = chrono_types::make_shared<ChLinkMateSpherical>();
spherical->Initialize(
    linkA, linkB,
    ChFramed(ChVector3d(0.5, 0, 0))
);
my_system.Add(spherical);
```

---

### 📐 平面副（ConDOF = 3）

| 类名 | 说明 |
|------|------|
| `ChLinkMatePlanar` | **推荐**，XY 平面共面 |
| `ChLinkLockPlanar` | 功能版 |

**描述：** 两个物体的 XY 平面保持共面，允许在平面内平移（X、Y）和绕 Z 轴旋转。锁定 Z 方向平移和 X、Y 轴转动。

---

### 📏 点在直线上（ConDOF = 2）

| 类名 | 说明 |
|------|------|
| `ChLinkLockPointLine` | 点约束在直线上，可自由旋转 |
| `ChLinkLockPointSpline` | 点约束在样条曲线上 |

**描述：** 刚体上的某个点被约束在一条直线（或曲线）上滑动，可以绕任意方向旋转。

---

### 📏 点在平面上（ConDOF = 2）

| 类名 | 说明 |
|------|------|
| `ChLinkMateDistanceZ` | Z 方向距离固定 |
| `ChLinkLockPointPlane` | 点在平面内自由移动 |

---

### ⚙️ 齿轮 / 皮带轮 / 齿条（ConDOF = 1）

| 类名 | 说明 |
|------|------|
| `ChLinkLockGear` | 齿轮：耦合两轴绕 Z 轴的旋转 |
| `ChLinkLockPulley` | 皮带轮：类似齿轮，含皮带特有功能 |
| `ChLinkMateRackPinion` | 齿条-齿轮：将小齿轮旋转和齿条平移耦合 |

**描述：** 这三类约束用来模拟传动机构，通过固定传动比将两个运动量耦合在一起。

```cpp
// 示例：两齿轮传动（传动比由齿数决定）
auto gear = chrono_types::make_shared<ChLinkLockGear>();
gear->Initialize(gear1_body, gear2_body, my_system);
gear->SetTransmissionRatioLocal(2.0);   // 传动比 2:1
my_system.Add(gear);
```

---

### 📏 固定距离（ConDOF = 1）

| 类名 | 说明 |
|------|------|
| `ChLinkDistance` | 两点极距离（杆长）保持固定 |

**描述：** 简单约束两个点之间的距离为常数，等效于一根无质量的刚性连杆。

```cpp
auto dist = chrono_types::make_shared<ChLinkDistance>();
dist->Initialize(bodyA, bodyB, false,
    ChVector3d(0, 0, 0),   // bodyA 上的点
    ChVector3d(1, 0, 0)    // bodyB 上的点
);
my_system.Add(dist);
```

---

## 三、驱动器（Actuators）

### 线性驱动器 `ChLinkMotorLinear`

**描述：** 在两个坐标帧之间施加线性方向的**力、速度或位置**驱动，同时可选择端部约束类型（无约束、移动副或球铰）。

```cpp
// 示例：位置驱动的线性执行器
auto motor = chrono_types::make_shared<ChLinkMotorLinearPosition>();
motor->Initialize(
    sliderBody, frameBody,
    ChFramed(ChVector3d(0, 0, 0))
);

// 设置目标位移函数（正弦运动）
auto pos_func = chrono_types::make_shared<ChFunctionSine>(
    0.1,    // 振幅 0.1m
    1.0     // 频率 1Hz
);
motor->SetMotionFunction(pos_func);
my_system.Add(motor);
```

### 旋转驱动器 `ChLinkMotorRotation`

**描述：** 在两个坐标帧之间施加绕 Z 轴的**力矩、转速或角度**驱动，可选端部约束（无约束、转动副、柱面副或 Oldham 节）。

```cpp
// 示例：恒定转速的旋转驱动器
auto rotmotor = chrono_types::make_shared<ChLinkMotorRotationSpeed>();
rotmotor->Initialize(
    rotorBody, statorBody,
    ChFramed(ChVector3d(0, 0, 0))
);

// 目标转速：2π rad/s（1 转/秒）
auto speed_func = chrono_types::make_shared<ChFunctionConst>(CH_2PI);
rotmotor->SetMotionFunction(speed_func);
my_system.Add(rotmotor);
```

---

### 线性弹簧-阻尼器 `ChLinkTSDA`

**描述：** 根据两帧之间的**距离变化**施加弹簧力和阻尼力，支持自定义力函数，也可内置完整 ODE（适合复杂阻尼模型）。

```cpp
// 示例：弹簧-阻尼连接两个刚体
auto spring = chrono_types::make_shared<ChLinkTSDA>();
spring->Initialize(
    bodyA, bodyB,
    false,                    // false = 用绝对坐标指定端点
    ChVector3d(0, 1, 0),      // bodyA 上的连接点
    ChVector3d(0, -1, 0)      // bodyB 上的连接点
);
spring->SetSpringCoefficient(500.0);   // 刚度 500 N/m
spring->SetDampingCoefficient(50.0);   // 阻尼 50 N·s/m
spring->SetRestLength(1.0);            // 自然长度 1m
my_system.Add(spring);
```

---

### 旋转弹簧-阻尼器 `ChLinkRSDA`

**描述：** 根据两帧之间绕 **Z 轴的相对转角**施加弹性力矩和阻尼力矩，支持自定义力矩函数。

```cpp
auto torsion = chrono_types::make_shared<ChLinkRSDA>();
torsion->Initialize(
    bodyA, bodyB,
    ChFramed(ChVector3d(0, 0, 0))
);
torsion->SetSpringCoefficient(100.0);   // 扭转刚度 100 N·m/rad
torsion->SetDampingCoefficient(10.0);   // 扭转阻尼 10 N·m·s/rad
my_system.Add(torsion);
```

---

## 四、所有约束完整速查表

| ConDOF | 约束类型 | 剩余自由度 | 推荐类 |
|--------|---------|-----------|--------|
| 6 | 固定（Fix） | 无 | `ChLinkMateFix` |
| 6/3 | 衬套（Bushing） | 柔性/球形 | `ChLinkBushing` |
| 5 | 转动铰（Revolute） | 绕 Z 旋转 | `ChLinkMateRevolute` |
| 5 | 移动副（Prismatic） | 沿 Z 平移 | `ChLinkMatePrismatic` |
| 4 | 万向节（Universal） | 绕 X、Y 旋转 | `ChLinkUniversal` |
| 4 | 转动+移动（Rev+Pris） | 沿 X 平移 + 绕 Z 旋转 | `ChLinkLockRevolutePrismatic` |
| 4 | Oldham 联轴节 | — | `ChLinkLockOldham` |
| 4 | 柱面副（Cylindrical） | 沿 Z 平移 + 绕 Z 旋转 | `ChLinkMateCylindrical` |
| 3 | 球铰（Spherical） | 绕任意轴旋转 | `ChLinkMateSpherical` |
| 3 | 平面副（Planar） | 在 XY 平面内移动 + 绕 Z 旋转 | `ChLinkMatePlanar` |
| 3 | 对齐（Aligned） | 三轴平移 | `ChLinkLockAlign` |
| 2 | 转动+球铰 | — | `ChLinkRevoluteSpherical` |
| 2 | 点在平面上 | — | `ChLinkLockPointPlane` |
| 2 | 点在直线上 | — | `ChLinkLockPointLine` |
| 2 | 平行（Parallel） | — | `ChLinkMateParallel` |
| 2 | 正交（Orthogonal） | — | `ChLinkMateOrthogonal` |
| 1 | 固定距离（Distance） | — | `ChLinkDistance` |
| 1 | 齿条-齿轮 | — | `ChLinkMateRackPinion` |
| 1 | 皮带轮（Pulley） | — | `ChLinkLockPulley` |
| 1 | 齿轮（Gear） | — | `ChLinkLockGear` |
| 0 | 自由（Free） | 6 个自由度 | `ChLinkLockFree` |

---

## 五、常见应用场景选型建议

| 工程场景 | 推荐约束 |
|----------|---------|
| 门铰链、摆杆 | `ChLinkMateRevolute` |
| 活塞缸、导轨滑块 | `ChLinkMatePrismatic` |
| 传动轴折角补偿 | `ChLinkUniversal` |
| 悬架球头、机械臂关节 | `ChLinkMateSpherical` |
| 齿轮传动 | `ChLinkLockGear` |
| 皮带传动 | `ChLinkLockPulley` |
| 齿条齿轮（转动→直线） | `ChLinkMateRackPinion` |
| 悬架弹簧/减震器 | `ChLinkTSDA` |
| 扭簧/扭转阻尼 | `ChLinkRSDA` |
| 电机/伺服驱动 | `ChLinkMotorRotation` / `ChLinkMotorLinear` |
| 橡胶衬套 | `ChLinkBushing` |
| 将零件固定到地面 | `body->SetFixed(true)` |

---

## 六、调试约束的 Checklist

- [ ] 约束连接后刚体位置不正确？→ 检查 `Initialize()` 中的坐标帧位置和方向是否正确
- [ ] 铰链轴方向不对？→ 约束默认绕/沿 **Z 轴**，需要通过旋转四元数调整坐标系朝向
- [ ] 约束"松弛"或刚体相互穿透？→ 增大求解器迭代次数（建议 200～400）
- [ ] 需要读取约束反力？→ 优先选 `ChLinkLock` 系列（有 `GetReactionOnBody1()` 等方法）
- [ ] 驱动器运动不平滑？→ 检查 `ChFunction` 是否连续，或换用 HHT 积分器

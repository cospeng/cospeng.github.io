---
title: "Project Chrono FEA 有限元分析学习笔记"
date: 2026-03-08
permalink: /posts/2026/03/project-chrono-FEA/
tags:
  - Project Chrono
  - 多体动力学
  - 仿真
---

## 🌟 FEA 模块是什么？能做什么？

FEA 模块用于在 Chrono 中模拟**可变形体**。支持三大类有限元：
- **实体单元**（如四面体）：适合任意几何形状的零件，如金属件、塑料模具、肌肉组织等
- **壳体单元**：适合薄壁构件，如轮胎、气囊等
- **梁单元**：适合细长结构，如缆绳、电线、传动轴、风机叶片等


大多数有限元**支持大位移**，可以模拟橡胶结构大变形、飞机复杂机动、叶片大转角等几何非线性问题。当然也支持线性小位移分析作为子集。

---

## 一、FEA 数据结构

`ChSystem` 中包含 `fea::ChMesh` 对象列表。`ChMesh` 本身是有限元（`ChElementBase` 的子类）和节点（`ChNodeFEAbase` 的子类）的容器。一个 `ChMesh` 可以包含多个单元。

层级关系如下：

```
ChSystem
  └── ChMesh（网格容器）
        ├── ChNodeFEA*（节点）
        └── ChElementBase（单元）
```

---

## 二、创建 FEA 模型的完整步骤

### 第 1 步：创建网格 `ChMesh`

**描述：** 网格是节点和单元的统一容器，必须先加入系统才能参与仿真。

```cpp
ChSystem my_system;

// 创建网格（用 make_shared 管理内存，不用手动 delete）
auto my_mesh = chrono_types::make_shared<ChMesh>();

// ⚠️ 必须把网格加入系统！
my_system.Add(my_mesh);
```

> 💡 一个系统中可以有多个 `ChMesh`，它们相互独立。

---

### 第 2 步：创建节点并添加到网格

**描述：** 节点是有限元的基础，代表空间中的自由度（xyz 位移，或带旋转自由度）。节点的初始位置在构造时设定，默认质量为 0（质量由单元贡献）。

```cpp
// 创建 4 个节点（XYZ 自由度）
auto mnode1 = chrono_types::make_shared<ChNodeFEAxyz>(ChVector3d(0, 0, 0));
auto mnode2 = chrono_types::make_shared<ChNodeFEAxyz>(ChVector3d(0, 0, 1));
auto mnode3 = chrono_types::make_shared<ChNodeFEAxyz>(ChVector3d(0, 1, 0));
auto mnode4 = chrono_types::make_shared<ChNodeFEAxyz>(ChVector3d(1, 0, 0));

// ⚠️ 节点必须加入网格！
my_mesh->AddNode(mnode1);
my_mesh->AddNode(mnode2);
my_mesh->AddNode(mnode3);
my_mesh->AddNode(mnode4);
```

**常用节点属性设置：**

```cpp
// 设置集中质量（默认为 0）
mnode1->SetMass(0.01);
mnode2->SetMass(0.01);

// 施加固定力（常值，适合简单情形；复杂情况用 ChLoad）
mnode2->SetForce(ChVector3d(0, 5, 0));  // 向 Y 方向施加 5N

// 固定某个节点（边界条件）
mnode1->SetFixed(true);
```

> ⚠️ FEA 节点的默认质量为 0，质量主要由单元的材料密度贡献。如果需要在节点处集中质量，需要手动 `SetMass()`。

---

### 第 3 步：创建材料

**描述：** 材料定义了弹性参数，可以被多个单元共享。

```cpp
// 创建连续弹性材料
auto mmaterial = chrono_types::make_shared<ChContinuumElastic>();

// 设置杨氏模量（橡胶约 0.01e9，钢约 200e9，单位 Pa）
mmaterial->SetYoungModulus(0.01e9);

// 设置泊松比
mmaterial->SetPoissonRatio(0.3);
```

> 💡 不是所有单元都需要材料，比如 `ChElementSpring`（弹簧单元）没有材料属性。

---

### 第 4 步：创建单元并装配

**描述：** 单元连接节点、绑定材料，是 FEA 计算的核心对象。

```cpp
// 创建一个 4 节点线性四面体单元
auto melement1 = chrono_types::make_shared<ChElementTetraCorot_4>();

// ⚠️ 必须加入网格！
my_mesh->AddElement(melement1);

// 指定连接哪 4 个节点
melement1->SetNodes(mnode1, mnode2, mnode3, mnode4);

// 指定材料
melement1->SetMaterial(mmaterial);
```

---

## 三、FEA 单元类型详解

### 🔵 弹簧/杆类（1D）

#### `ChElementSpring` — 最简单的弹簧单元

**描述：** 两节点之间的线性弹簧阻尼单元，适合桁架、悬架等问题。零质量，可自定义刚度和阻尼。

2 个 `ChNodeFEAxyz` 节点，支持大位移，零质量单元。参数包括原长 L、刚度 k、阻尼 r，刚度矩阵解析计算（包含材料和几何刚度）。

```cpp
auto spring = chrono_types::make_shared<ChElementSpring>();
spring->SetNodes(nodeA, nodeB);
spring->SetSpringCoefficient(1000);   // 刚度 k
spring->SetDamperCoefficient(10);     // 阻尼 r
```

#### `ChElementBar` — 有质量的杆单元

**描述：** 与 `ChElementSpring` 类似，但增加了质量效应，参数使用截面积和杨氏模量，两端为球铰（无弯矩传递）。

2 个 `ChNodeFEAxyz` 节点，支持大位移，有质量，端部无力矩（类似两个球铰）。参数为原长 L、截面积 A、杨氏模量 E、阻尼（Rayleigh beta 参数）。

---

### 🟢 实体单元（3D 体）

#### `ChElementTetraCorot_4` — 最简单的四面体（推荐入门）

**描述：** 4 节点线性四面体，是体积单元中最快的，适合快速原型验证。

4 个 `ChNodeFEAxyz` 节点，线性插值，应力恒定，1 个积分点，协旋转公式支持大位移，用极分解计算协旋转坐标系，是最快的实体单元。

```cpp
auto tet4 = chrono_types::make_shared<ChElementTetraCorot_4>();
tet4->SetNodes(n1, n2, n3, n4);
tet4->SetMaterial(material);
```

#### `ChElementTetraCorot_10` — 精度更高的 10 节点四面体

**描述：** 每条棱中点额外增加 1 个节点，二次插值，应力线性变化，精度更高但计算更慢。

10 个 `ChNodeFEAxyz` 节点（4 顶点 + 6 棱中点），二次插值，4 个积分点，注意中间节点初始化时需位于棱的中点处。

#### `ChElementHexaCorot_8` — 8 节点六面体（砖块单元）

**描述：** 规则网格上的六面体线性单元，适合结构化网格划分场景。

8 个 `ChNodeFEAxyz` 节点，线性插值，8 个积分点，协旋转支持大位移，适合结构化网格。

#### `ChElementHexaCorot_20` — 20 节点六面体（高精度）

20 个节点（8 顶点 + 12 棱中点），二次插值，27 个积分点，精度更高。

#### `ChElementHexaANCF_3813` — ANCF 砖块单元（支持超弹性）

**描述：** 基于 ANCF 方法，支持大应变，可使用 Mooney-Rivlin 超弹性材料模型（适合橡胶类材料）。

8 个 `ChNodeFEAxyz` 节点，8 个积分点，使用增强假设应变（EAS），大应变，可用 Mooney-Rivlin 超弹性材料。

#### `ChElementHexaANCF_3813_9` — 带塑性的 ANCF 砖块

**描述：** 9 节点（中心多一个），支持 J2 金属塑性和 Drucker-Prager 土体塑性模型。

应变公式支持 Green-Lagrange 和 Hencky；塑性模型包括 J2（金属）、DruckerPrager 和 DruckerPrager_Cap（土体/塑料）。

---

### 🟡 梁单元（1D 细长结构）

#### `ChElementCableANCF` — 缆索单元（电缆、绳索）

**描述：** 专为细长缆索设计，不考虑扭转和剪切，计算速度快。

2 个 `ChNodeFEAxyzD` 节点，ANCF 大位移，细梁（无剪切），不模拟扭转刚度，适合电线、缆绳。截面属性通过 `ChBeamSectionCable` 设置（A, I, E, 密度, 阻尼）。

```cpp
auto cable_section = chrono_types::make_shared<ChBeamSectionCable>();
cable_section->SetDiameter(0.01);       // 直径 1cm
cable_section->SetYoungModulus(1e9);    // 刚度

auto cable = chrono_types::make_shared<ChElementCableANCF>();
cable->SetNodes(nodeD1, nodeD2);
cable->SetSection(cable_section);
```

#### `ChElementBeamEuler` — 欧拉-伯努利梁（经典细梁）

**描述：** 经典工程梁单元，基于欧拉-伯努利理论（无剪切效应），适合弯曲为主、剪切可忽略的细长梁。

2 个 `ChNodeFEAxyzrot` 节点（带旋转自由度），线性插值，协旋转大位移，细梁假设（无剪切）。截面属性通过 `ChBeamSectionEuler` 系列类设置，包括 A, Iyy, Izz, G, J 等，还支持截面偏心、剪切中心偏移。

```cpp
// 快捷方式：圆形截面
auto beam_section = chrono_types::make_shared<ChBeamSectionEulerEasyCircular>(
    0.01,    // 半径
    1e9,     // 杨氏模量
    0.3,     // 泊松比
    1000     // 密度
);

// 或者：矩形截面
auto beam_section2 = chrono_types::make_shared<ChBeamSectionEulerEasyRectangular>(
    0.02, 0.01,  // 宽×高
    1e9, 0.3, 1000
);

auto beam = chrono_types::make_shared<ChElementBeamEuler>();
beam->SetNodes(nodeRot1, nodeRot2);
beam->SetSection(beam_section);
```

#### `ChElementBeamIGA` — 等几何梁（最高精度，推荐高端应用）

**描述：** 基于等几何分析（IGA）和 B 样条形函数的 Cosserat 杆理论梁单元，支持厚梁剪切效应（Timoshenko 理论），适合直升机旋翼、风机叶片等复杂结构。

用户可定义阶次 n（1=线性，2=二次，3=三次），每个单元是 B 样条的一段，使用 `ChNodeFEAxyzrot` 节点，支持厚梁剪切（Timoshenko），支持初始曲线形态，减缩积分防止剪切锁定。推荐使用 `ChBuilderBeamIGA` 工具类快速创建完整 B 样条梁。

截面属性通过模块化方式组合：

```
ChBeamSectionCosserat
  ├── ChElasticityCosserat（弹性）
  │     ├── ChElasticityCosseratSimple   ← 常用
  │     ├── ChElasticityCosseratAdvanced
  │     └── ChElasticityCosseratGeneric
  ├── ChInertiaCosserat（惯性）
  │     └── ChInertiaCosseratSimple
  ├── ChDampingCosserat（阻尼，可选）
  │     └── ChDampingCosseratRayleigh
  └── ChPlasticityCosserat（塑性，可选）
```

---

### 🔴 壳体单元（2D 薄壁结构）

#### `ChElementShellReissner` — 厚壳（Reissner-Mindlin）

**描述：** 4 节点四边形厚壳单元，基于 Reissner 6 自由度壳理论，无剪切锁定。

4 个 `ChNodeFEAxyzrot` 节点，双线性插值，4 个积分点，支持大位移，无剪切锁定（ANS），支持多层叠层材料（CLT 理论），节点无需对齐壳面法向（初始化时自动计算偏转）。

#### `ChElementShellANCF_3423` — ANCF 厚壳

4 个 `ChNodeFEAxyzD` 节点，ANCF 大位移，厚壳，多层材料，ANS-EAS 无剪切锁定，节点 D 方向需初始化时对齐壳体法向。

#### `ChElementShellBST` — 三角形薄壳（效率最高）

**描述：** 三角形薄壳，基于 Kirchhoff-Love 理论（无剪切），适合柔性织物、帆布、生物组织等大变形薄膜。

6 个 `ChNodeFEAxyz` 节点（3 个来自本三角形 + 3 个来自相邻三角形），常应变/常曲率，支持多层材料，基于 Kirchhoff-Love 理论（无剪切），适合织物、帆等结构。

---

## 四、单元选型速查表

| 场景 | 推荐单元 | 关键特点 |
|------|----------|----------|
| 快速验证实体变形 | `ChElementTetraCorot_4` | 最快，线性 |
| 高精度实体分析 | `ChElementTetraCorot_10` / `HexaCorot_20` | 二次插值 |
| 橡胶/超弹性材料 | `ChElementHexaANCF_3813` | Mooney-Rivlin |
| 土体/塑性材料 | `ChElementHexaANCF_3813_9` | J2/DruckerPrager |
| 细缆绳/绳索 | `ChElementCableANCF` | 无扭转，高效 |
| 通用工程梁 | `ChElementBeamEuler` | 欧拉-伯努利，简单 |
| 高精度梁/叶片 | `ChElementBeamIGA` | IGA + Cosserat |
| 薄壳（轮胎/气囊） | `ChElementShellANCF_3423` | ANCF 厚壳 |
| 织物/生物软组织 | `ChElementShellBST` | Kirchhoff-Love |

---

## 五、接触注意事项

目前 FEA 中**不支持 NSC（非光滑接触）**，如果需要接触仿真，只能使用 SMC（光滑接触，基于弹簧-阻尼惩罚法）。即必须用 `ChSystemSMC`，不能用 `ChSystemNSC`。

---

## 六、完整建模流程回顾

```
1. 创建 ChMesh → 加入 ChSystem
2. 创建节点（ChNodeFEAxyz 等）→ AddNode() 到 ChMesh
   ├── 可选：SetMass()、SetForce()、SetFixed()
3. 创建材料（ChContinuumElastic 等）
4. 创建单元 → AddElement() 到 ChMesh
   ├── SetNodes()  指定连接节点
   └── SetMaterial()  指定材料
5. 添加载荷/约束（ChLoad、ChLink）
6. 运行仿真
```

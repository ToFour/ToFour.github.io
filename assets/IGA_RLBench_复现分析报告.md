# IGA在RL-Bench复现分析报告

---

## 1. 复现方法

### 1.1 核心思路

IGA推理的是任务相关两物体点云图的对齐关系。将RL-Bench的每个任务划分为多个**对齐时刻**，每个对齐时刻标注好 `grasped_object` 和 `target_object` 分别对应sample里的 `pcd_a` 和 `pcd_b`，生成标注文件。收集专家演示后，根据标注文件获取对应时刻对应物体的点云对进行推理执行。

### 1.2 标注文件格式

以 `StackCups` 为例，该任务被划分为4个对齐时刻：

```json
{
  "task_name": "StackCups",
  "alignments": [
    {
      "grasped_object": "g",
      "target_object": "cup1_visual",
      "aligned_time": 1,
      "gripper_action": 0,
      "align_id": 0
    },
    {
      "grasped_object": "cup1_visual",
      "target_object": "cup2_visual",
      "aligned_time": 3,
      "gripper_action": 1,
      "align_id": 1
    },
    {
      "grasped_object": "g",
      "target_object": "cup3_visual",
      "aligned_time": 6,
      "gripper_action": 0,
      "align_id": 2
    },
    {
      "grasped_object": "cup3_visual",
      "target_object": "cup2_visual",
      "aligned_time": 8,
      "gripper_action": 1,
      "align_id": 3
    }
  ]
}
```

---

## 2. 实验设置

### 2.1 任务选取与分类

参照原论文涵盖的抓握、堆叠、清扫、悬挂、插入、倾倒六种操作类型，从RL-Bench中选取14个任务，按以下标准分类：

- **简单任务（1-2次对齐）**：操作步骤少，对齐精度要求较低
- **复杂任务**：需要多步连续对齐
- **困难任务**：操作精度要求高（如插入类任务）

**简单任务（8个）**：

| 任务名称               | 任务描述               |
| ------------------ | ------------------ |
| `ReachTarget`      | 将末端执行器移动到目标位置      |
| `PickAndLift`      | 抓取目标方块并提起          |
| `PickUpCup`        | 抓起杯子               |
| `LampOff`          | 将末端执行器移动到电灯开关上方并按下 |
| `BasketballInHoop` | 将篮球置于篮筐正上方后释放      |
| `PhoneOnBase`      | 将电话放在电话座机上         |
| `StackWine`        | 将红酒放在酒架子上          |
| `WaterPlants`      | 拿起水壶浇花             |

**复杂任务（3个）**：

| 任务名称               | 任务描述            |
| ------------------ | --------------- |
| `PourFromCupToCup` | 拿起一个杯子向另一个杯子倒水  |
| `StackCups`        | 将三个杯子叠放在一起      |
| `PlaceCups`        | 拿起杯子，将杯子手柄挂在杯架上 |

**困难任务（3个）**：

| 任务名称                       | 任务描述         |
| -------------------------- | ------------ |
| `PlugChargerInPowerSupply` | 将充电器插头插入电源插口 |
| `InsertUsbInComputer`      | 将USB插入电脑接口   |
| `HangFrameOnHanger`        | 将画框对准挂钩挂上    |

### 2.2 实验协议

1. **演示数据收集**：每个任务收集4条专家演示，轨迹由RL-Bench内置组件生成，演示点云对根据标注文件获取
2. **评价指标**：
   - **任务成功率**：每个任务运行5次，计算成功次数/总次数
   - **任务完成度**：单次测试中已完成对齐数/总对齐数（总对齐数由标注文件确定）；能反映任务在哪一步失败，弥补成功率在多步任务上分辨率不足的问题

---

## 3. 实验结果

### 3.1 各任务完成度与成功率

**简单任务**：

| 任务               | 完成度1 | 完成度2 | 完成度3 | 完成度4 | 完成度5 | 成功率 | 平均完成度 |
| ---------------- | ---- | ---- | ---- | ---- | ---- | --- | ----- |
| ReachTarget      | 1/1  | 1/1  | 1/1  | 1/1  | 1/1  | 5/5 | 100%  |
| PickAndLift      | 2/2  | 2/2  | 0/2  | 2/2  | 0/2  | 3/5 | 60%   |
| PickUpCup        | 1/1  | 1/1  | 1/1  | 1/1  | 1/1  | 5/5 | 100%  |
| LampOff          | 0/2  | 0/2  | 1/2  | 0/2  | 2/2  | 1/5 | 30%   |
| BasketballInHoop | 1/2  | 1/2  | 1/2  | 1/2  | 1/2  | 0/5 | 50%   |
| PhoneOnBase      | 2/2  | 1/2  | 1/2  | 1/2  | 1/2  | 1/5 | 60%   |
| StackWine        | 2/2  | 2/2  | 1/2  | 1/2  | 1/2  | 3/5 | 70%   |
| WaterPlants      | 0/2  | 1/2  | 1/2  | 0/2  | 0/2  | 0/5 | 20%   |

**复杂任务**：

| 任务               | 完成度1 | 完成度2 | 完成度3 | 完成度4 | 完成度5 | 成功率 | 平均完成度 |
| ---------------- | ---- | ---- | ---- | ---- | ---- | --- | ----- |
| PourFromCupToCup | 1/2  | 0/2  | 2/2  | 0/2  | 0/2  | 1/5 | 30%   |
| StackCups        | 4/4  | 4/4  | 3/4  | 4/4  | 4/4  | 4/5 | 95%   |
| PlaceCups        | 2/3  | 0/3  | 2/3  | 2/3  | 1/3  | 0/5 | 47%   |

**困难任务**：

| 任务                       | 完成度1 | 完成度2 | 完成度3 | 完成度4 | 完成度5 | 成功率 | 平均完成度 |
| ------------------------ | ---- | ---- | ---- | ---- | ---- | --- | ----- |
| PlugChargerInPowerSupply | 2/2  | 1/2  | 1/2  | 1/2  | 1/2  | 1/5 | 60%   |
| InsertUsbInComputer      | 0/2  | 1/2  | 1/2  | 1/2  | 0/2  | 0/5 | 30%   |
| HangFrameOnHanger        | 1/2  | 2/2  | 1/2  | 2/2  | 1/2  | 2/5 | 70%   |

> **注**：每个任务均进行5次独立测试。

### 3.2 汇总统计

| 任务类别       | 平均任务完成度 | 平均任务成功率 |
| ---------- | ------- | ------- |
| 简单任务（8个） | 61%     | 45%     |
| 复杂任务（3个） | 57%     | 33%     |
| 困难任务（3个） | 53%     | 20%     |

整体趋势符合预期：任务难度越高，成功率和完成度均下降。但注意**完成度下降幅度（61%→53%）明显小于成功率下降幅度（45%→20%）**。三类任务平均完成度均超过50%但任务成功率均小于50%，说明任务通常能完成前序步骤（一般为抓取动作），仅在最后的高精度对齐阶段失败。

---

## 4. 结果分析

### 4.1 成功案例

**ReachTarget / PickUpCup（成功率100%）**：任务仅需1次对齐，目标物体在点云中占比大，IGA能够提取到充分的任务语义，图对齐稳定，轨迹规划简单，无失败瓶颈。

**StackCups（成功率80%，完成度95%）**：虽然需要4次连续对齐，但每次对齐的目标物体（杯子）轮廓清晰，点云完整，IGA能够持续推理出合理的对齐关系。唯一失败案例来自轨迹规划阶段的碰撞，而非对齐失败。

从纯推理的对齐结果来看，抓取、倾倒、堆叠这些动作完成得较好。

### 4.2 失效模式分析

#### 失效模式1：点云中任务对齐相关部分占比低导致对齐大幅偏移

**受影响任务**：`WaterPlants`（20%）、`BasketballInHoop`（50% 完成度，0% 成功）、`LampOff`（30%）

**现象**：推理出的对齐与演示对齐偏差极大，远超可接受范围。

**原因**：任务相关对齐所涉及的物体（水壶手柄、篮筐、灯开关）在整体点云中占比过小。以 `BasketballInHoop` 为例，点云包含了整个篮球架，而任务只需推理球与篮筐的对齐关系，大量无关点云稀释了可学习的任务语义，导致IGA无法推理出合理对齐。

**论文依据**：这对应IGA论文Section 5 Limitations第1点——"we assume that **segmented point clouds are available of sufficient observability to capture task-relevant object parts**, and thus we require three depth cameras" [1]。论文在真机实验中使用3个RealSense D415深度相机，并分两步分别采集目标物体和抓取物体的点云（Section 4.3），就是为了保证点云的可观测性。在RL-Bench中，即使有6个相机，由于视角遮挡和物体几何的复杂性，这一前提仍可能被打破。失效模式1本质上是论文自身假设在仿真环境中的失效。

**案例**：
> BasketballInHoop（绿色的是篮球，黄色是篮球推理前的位置）
> ![[202b6b24e0c828d7c0dc668778add19d.jpg|371]]
> WaterPlants
> ![[8dd82e064b5b546011c2548d86b559dd.jpg]]

> **补充实验（见4.3）**验证了这一原因：点云中任务相关物体占比与对齐精度存在正相关关系。

#### 失效模式2：对齐精度不足导致高精度操作失败

**受影响任务**：`InsertUsbInComputer`（0%）、`PlaceCups`（0%）、`PhoneOnBase`（20%）、`PlugChargerInPowerSupply`（20%）、`HangFrameOnHanger`（40%）

**现象**：推理出的对齐大致方向正确，但偏差量级在毫米甚至厘米级，无法满足插入类操作的容差要求。

**原因**：根据4.3的结论，需要精密对齐的点云部分（如插孔周围）占比过少。比如 `PlugChargerInPowerSupply` 任务，如果想要让插头对准插孔，插孔周围的点云就需要有足够占比，但实际上这部分点云很难获取。

**论文依据**：两个依据叠加：
1. **Limitation #6** [1]："Training of our model relies on data generated using a **heuristic method**, which may introduce inaccuracies in the training samples and **limit the model's precision**." 训练数据由ShapeNet物体随机SE(3)对齐+DIF-Net形变生成，这种启发式生成方式本身不够精细，限制了模型精度上限。
2. **真机实验佐证**：论文Table 2中Insert任务（bottle into shoe）成功率为11/15（73%），是所有任务中最低的（除Sweep外）。但论文选用的"bottle into shoe"是大口径插入，容差大；换成RL-Bench中的USB/充电器插口等小口径、高精度任务后，这一精度瓶颈被显著放大。

**案例**：
> PlugChargerInPowerSupply
> ![[9b170786c7bfc6e1a60b2c030fe9a205.jpg|301]]
> HangFrameOnHanger
> ![[截图 2026-05-07 08-29-27.png|280]]

#### 失效模式3：RL-Bench路径规划失败（与IGA无关）

**受影响任务**：`PickAndLift`（部分失败）、`LampOff`（部分失败）、`InsertUsbInComputer`（部分失败）、`StackCups`（1次失败）

**现象**：IGA推理出的对齐目标是合理的，但RL-Bench内置的运动规划器在生成执行轨迹时出现逆解失败或碰撞。

**论文依据**：IGA的pipeline设计中，运动规划本身不属于IGA的职责范围。论文Section 3.1指出"we treat individual time steps in a trajectory independently... Assuming rigid grasps, we can **directly apply** this sequence of transformations to the robot's end-effector" [1]；Section 4.3进一步说明"executed with a **scripted controller**" [1]。即IGA只负责推理waypoint位姿，运动规划交由外部控制器完成。在真机实验中论文使用了精心设计的脚本控制器，而RL-Bench使用的是RRTConnect等通用运动规划器，两者的规划能力有本质差距。因此，此类失败源于运动规划层而非IGA的感知推理层。

**说明**：这说明当前的**任务成功率同时受对齐质量和运动规划两个因素影响**，成功率会低估IGA的实际对齐能力，也说明如果要把IGA用在操作任务，需要在运动规划方面多下功夫。

#### 失效模式4：对称物体存在多种对齐可能

**受影响任务**：`HangFrameOnHanger`

**现象**：对称物体存在多种符合任务语义的对齐方式，但推理出的对齐不具备可执行性。

**论文依据**：论文Section 3.4 [1]指出"The energy-based approach allows us to learn complex and **multi-modal distributions**... allowing for **different ways of aligning objects**. This flexibility is crucial when **multiple valid alignments exist, e.g. when dealing with symmetrical objects**." 即多模态对齐是IGA的设计特性。但论文Section 4.1的Multi-Modal Demonstrations评估（Table 1）也表明，多模态条件下误差最大（Trans 3.9±2.3cm, Rot 11.2±6.7°），说明对称性带来的歧义是IGA的已知弱点。在实际操作中，某些模态的对齐虽然在几何上合法，但在运动学上不可执行。

**案例**：在测试 `OpenDoor` 任务时，选取夹爪与门作为点云对，由于门板的对称性，夹爪可能对齐到转轴那端导致动作不可执行。`HangFrameOnHanger` 出现画框倒挂的情况也属于同一类问题。
> HangFrameOnHanger倒挂
> ![[inversehang.png|288]]

### 4.3 补充实验：任务对齐相关点云占比对对齐效果的影响

针对失效模式1的假设，设计了补充实验：通过改变收集演示对齐的点云对象，观察对齐效果。

以 `LampOff` 为例。任务相关物体为夹爪和电灯本体、电灯开关按钮。采集了两组演示点云对进行推理：
- **第一组**：pcd_a为夹爪，pcd_b为电灯本体和电灯开关按钮（含无关点云）
- **第二组**：pcd_a为夹爪，pcd_b仅含电灯开关按钮（纯任务相关点云）

#### 第一组（含无关点云）

**演示点云**：
> ![[lampoff_demo2.png]]

**推理效果**：
> ![[lampoff_live2.png|474]]

#### 第二组（纯任务相关点云）

**演示点云**：
> ![[lampoff_demo1.png]]

**推理效果**：
> ![[lampoff_live1.png|523]]

**结论**：蕴含任务对齐相关的点云占本身点云的比例过少，会导致点云对所携带的可学习任务语义不足，进而使IGA推理出的对齐大幅偏移。这是当前最值得改进的瓶颈点。

---

## 5. 思考

对于把IGA运用在机器人操作任务，有以下几点思考：

### 5.1 针对失效模式1：任务感兴趣区域（RoI）提取

在送入IGA之前，增加一个**任务相关区域提取**步骤，根据演示对齐提取与任务最相关的点云区域，去除无关背景点云，提升任务对齐相关点云的占比。

### 5.2 完整的点云是IGA的一切前提

由于只采用了点云这一种数据，点云的完整性（特别是与任务对齐相关的点云）十分重要，几乎决定了操作的成败。在初期采取的是根据当前帧获取点云，由于RL-Bench的相机设置问题，物体经常会被机械臂遮挡，获取到的点云少得可怜，导致成功率惨淡。后面将从初始帧到当前帧的点云合并才改善了这个问题。

### 5.3 对物体多样性的泛化缺乏探讨

RL-Bench给的任务物体都是一个外观，其变种也只是变换一下物体的摆放位置，因此IGA的核心卖点——对物体多样性的泛化，没有在RL-Bench的任务中体现出来。

### 5.4 IGA难以适配长时序操作任务

在本次复现中，都是人工规划了完成任务的每个步骤。当然规划这个问题好说，LLM用于机器人规划的应用很常见。更重要的是IGA的对齐无法表达出"接近但未接触"的意图——对于操作距离并不关心，当把抓取物体这个动作分解为两个对齐（先移动到物体附近，再执行抓取，以防干涉/碰撞），第一个对齐往往直接就贴到物体了。这个特点可能会导致IGA难以执行一些需要安全间距的调整动作。

### 5.5 IGA是一个很solid的工作

从方法的呈现到仿真实验和真机实验，甚至有长达8页的附录讲述具体的细节，真机demo也拍得很漂亮。也是从这篇工作学到了一个高质量的工作该如何完成。

---

## 参考文献

[1] Vosylius, I., & Johns, E. (2023). Few-Shot In-Context Imitation Learning via Implicit Graph Alignment. *Conference on Robot Learning (CoRL)*.

[2] James, S., et al. (2020). RLBench: The Robot Learning Benchmark & Learning Environment. *IEEE Robotics and Automation Letters (RA-L)*.

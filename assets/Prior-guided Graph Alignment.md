  ## 华工林泽锴
  
  ## &#x20;1 引言

$\qquad$在IGA的复现与初步实验中，我发现：当任务相关点云在整体点云中占比过低时，模型的对齐误差会显著增大，且无法有效推理出演示对齐过程中蕴含的任务语义特征。深入分析表明，导致该现象的原因主要有以下两点：

1.  **演示多样性瓶颈**：按理说，物体外观与演示的高度一致性本应降低学习难度。但在实验中，当提供多组物体外观完全一致的演示时，IGA的对齐效果反而劣于提供多样化演示的情况，见Figure1。这表明IGA难以在外观同质化的演示中学习到统一的任务语义。其本质是因为IGA是基于上下文的模仿学习,多次同质化的演示相当于单次演示，导致上下文不足。充足的上下文需要获取涵盖丰富外观变化的演示数据，但这往往成本高昂且难以实现。
2.  **任务无关特征的干扰**：在对齐的两个物体中，任务相关部分的点云占低会导致对齐效果变差。这导致IGA的图注意力机制被大量任务无关的背景或冗余点云稀释，无法聚焦于关键的几何特征。
![[Pasted image 20260603110902.png]]
**Figure1:**  不同的演示多样性对平移误差的影响。（引自IGA图5）

$\qquad$为了解决上述问题，我提出了一种先验引导的图对齐方法（Prior-guided Graph Alignment, 简称 **Prior-GA**）。本方法引入了一个基于视觉语言模型的任务ROI提取模块，该模块通过分析演示视频序列，识别出对任务成功至关重要的局部几何特征，从而在演示点云中标注出任务感兴趣区域。在获取标注后的演示点云后，本文采用点云配准的方法，将Task-ROI标签迁移至实时点云。在此机制下，Prior-GA成为一个专注于Task-ROI的高效图对齐器，仅接受任务相关区域的点云输入，从而更精准地聚焦于任务关键部分的对齐。相较于IGA，Prior-GA的优势可概括如下：

**（1）One-Shot 学习能力：**  本文引入了基于视觉语言模型的任务ROI提取模块，能够直接获取演示对齐中的任务相关局部。该方法仅需单次演示即可完成对齐，缓解了IGA对多样化演示依赖。
**（2）更高的任务信息密度：** 基于Task-ROI提取的点云对过滤了大量冗余信息，蕴含更高的任务信息密度，从而提升了复杂场景下图对齐的精准度与鲁棒性。
**（3）高效精准的局部映射方案：** 本方法采用了一种基于 FPFH 粗配准与 ICP 精配准的局部映射方案，将演示点云与实时点云在空间上精确对齐。该方案能将任务相关的局部区域从演示点云传递至实时点云，且平均配准耗时仅为 **0.002 秒**，证明了其在真实机器人系统上实时运行的潜力。

---
## 2 方法

### &#x20;2.1 基于VLM的任务ROI提取模块

$\qquad$在获取到演示视频序列以及对齐时刻的点云对后，我设计了一个基于视觉语言模型的任务ROI提取模块。首先，将演示视频的关键帧与任务的动作描述输入至VLM中，其中对齐帧被划分为5\*5的网格供VLM选择。通过提示词工程，引导VLM识别物体上与任务执行高度相关的局部区域，并输出相应的网格ID。随后，利用Segment Anything Model (SAM) 在输出的局部区域生成候选分割掩码。接着，再次利用VLM对生成的掩码图像进行视觉推理，从中筛选出最符合任务语义的局部掩码。最后，结合为物体本身的掩码，从而在演示点云中提取出Task-ROI点云。
![[result1.jpeg]]
**Figure2:**  该图展示了HangFrameOnHanger任务中VLM提取的掩码和Task-ROI点云。红色为物体$A$的掩码，蓝色为物体$B$的掩码。最右侧为过滤好的点云。
### &#x20;2.2 基于点云配准的局部映射 

$\qquad$为了解决Task-ROI点云的实时识别问题，我提出了基于点云配准的局部映射方法。点云配准能够实现同一物体在不同位姿下的空间对应。为了将演示阶段提取的Task-ROI应用于实时推理阶段，本方法通过计算演示点云与实时点云之间的刚体变换矩阵，将演示点云中的Task-ROI标签映射至实时点云。在配准策略上选择了基于FPFH的粗配准和迭代最近点（ICP）的精配准的组合方案。
$\qquad$这种基于配准的局部映射策略，不仅避免了在实时推理阶段重复调用VLM带来的高昂计算开销，保证了系统的实时性，还能有效克服实时场景中视角变化或轻微遮挡带来的ROI识别误差，确保任务关键区域在动态执行过程中的稳定追踪。
![[e7849e7cc075229fdb60f04a1903055c.png|340]]
**Figure3:**  该图展示了点云配准的效果。红色为演示点云，蓝色为实时点云。红蓝交叉重叠意味着经过配准后演示点云与实时点云已高度重叠。

---

## 3.实验

### 3.1 对齐实验

#### 3.1.1 实验设置

$\qquad$这个实验主要回答一个问题：在单一演示（One-Shot）的设置下，Prior-GA能否获得更好的对齐性能。为了评估模型的对齐精度，我设计了如下实验： 从给定的演示点云对序列 $$\{A_1, A_2, \dots, A_n\},n=2$$ 中，对点云对 $A_1$中的物体$A$施加一个随机的噪声 $SE(3)$变换 $T_{gt}$，将其作为推理的**实时点云**。序列中剩余的点云对则作为**演示输入**。推理的结果$T_{pred}$则会被作用在实时点云的物体$B$上。我采用平移误差和旋转误差作为核心评价指标：

**平移误差 ($E_{trans}$)**：衡量预测平移向量 $\mathbf{t}_{pred}$ 与真值 $\mathbf{t}_{gt}$ 之间的欧几里得距离（单位：cm）：
$$E_{trans} = \|\mathbf{t}_{pred} - \mathbf{t}_{gt}\|_2 \times 100$$

**旋转误差 ($E_{rot}$)**：衡量预测旋转矩阵 $R_{pred}$ 与真值 $R_{gt}$ 之间的测地距离，并转换为角度（单位：$^\circ$）：
$$E_{rot} = \arccos\left( \frac{\text{tr}(R_{pred} R_{gt}^\top) - 1}{2} \right) \cdot \frac{180}{\pi}$$
#### 3.1.2 实验结果与分析

<table>
  <thead>
    <tr>
      <th rowspan="2">Task</th>
      <th colspan="2">Prior‑GA</th>
      <th colspan="2">IGA</th>
    </tr>
    <tr>
      <th>Trans (cm) </th>
      <th>Rot (°) </th>
      <th>Trans (cm) </th>
      <th>Rot (°) </th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>PickUpCup</strong></td>
      <td class="better">1.17</td>
      <td class="better">5.97</td>
      <td>4.91</td>
      <td>58.01</td>
    </tr>
    <tr>
      <td><strong>LampOff</strong></td>
      <td class="better">1.25</td>
      <td class="better">27.90</td>
      <td>30.08</td>
      <td>60.71</td>
    </tr>
    <tr>
      <td><strong>BasketballInHoop</strong></td>
      <td class="better">9.44</td>
      <td class="better">59.18</td>
      <td>22.00</td>
      <td>71.13</td>
    </tr>
    <tr>
      <td><strong>PhoneOnBase</strong></td>
      <td class="better">2.27</td>
      <td class="better">4.40</td>
      <td>8.76</td>
      <td>47.10</td>
    </tr>
    <tr>
      <td><strong>WaterPlants</strong></td>
      <td>5.48</td>
      <td>17.20</td>
      <td class="better">3.14</td>
      <td class="better">11.70</td>
    </tr>
    <tr>
      <td><strong>HangFrameOnHanger</strong></td>
      <td class="better">4.13</td>
      <td class="better">19.85</td>
      <td>28.41</td>
      <td>123.59</td>
    </tr>
  </tbody>
</table>

**Table1:**   各个任务的对齐结果的平移与旋转误差（均值）。

| **指标**          | **IGA** | **Prior-GA** | **提升幅度**     |
| --------------- | ------- | ------------ | ------------ |
| **Trans (cm)**​ | 16.22   | **3.96**​    | **75.6%**​ ↓ |
| **Rot (°)**​    | 62.04   | **22.42**​   | **63.8%**​ ↓ |
**Table2:**  对齐结果的平移与旋转误差（均值）。表现最优的方法数值以加粗显示。

$\qquad$实验结果表明,Prior-GA通过引入视觉语言模型提取任务ROI，改善了 IGA 中任务无关特征稀释注意力及演示同质化导致任务语义缺失的问题。在平移误差和旋转误差两个指标分别获得了75.6%和63.8%的提升幅度，初步验证了任务ROI对IGA的提升。
![[ba6303b07b14d953391373b9e06fb92d.png|697]]
**Figure4:**  该图展示了 LampOff、BasketballInHoop 和 HangFrameOnHanger这三个任务的点云对，其中红色为任务相关点云。

$\qquad$针对 LampOff、BasketballInHoop 和 HangFrameOnHanger 这三个任务相关点云占比极低的代表性任务，实验结果有力地验证了 Prior-GA 的核心优势：IGA 由于依赖全局上下文建模，在开关按钮、篮筐、挂钩卡槽等微小关键区域被海量背景点云淹没时，图注意力机制发生严重稀释，导致平移误差分别高达 30.08cm、22.00cm和28.41cm。而 Prior-GA 通过 VLM 显式提取 Task-ROI，成功过滤了大量的冗余几何信息，迫使模型仅聚焦于任务相关局部进行对齐，从而将平移误差大幅降低至 1.25cm、9.44cm 和 4.13cm，验证了在单次演示的设置和任务点云占比过低的场景下，语义先验引导的局部对齐的有效性。
$\qquad$ 
### 3.2 RLbench实验
### 3.2.1 实验设置

$\qquad$实验基于RLbench环境，图像尺寸为$512X512$,更大的图像尺寸能提供更加丰富的点云。（之前的复现实验，图像尺寸为$128X128$）。提供物体外观一致的演示数量$n=2$。在任务开始时，机械臂已经抓取了其中一个物体，这点与IGA的真机实验保持一致，（Pick 任务除外）。每个任务进行三次测试。

| 任务                | IGA    | Prior-GA   |
| :---------------- | :----- | :--------- |
| PickUpCup         | 3/3    | 3/3        |
| LampOff           | 0/3    | 3/3        |
| BasketballInHoop  | 0/3    | 3/3        |
| PhoneOnBase       | 2/3    | 2/3        |
| WaterPlants       | 0/3    | 1/3        |
| HangFrameOnHanger | 0/3    | 1/3        |
| **Success Rate**  | 27.78% | **72.22%** |
**Table3:**  各任务成功率。表现最优的方法数值以加粗显示

$\qquad$实验结果表明，Prior-GA 将任务成功率从 27.78% 提升至 72.22%。其中大部分提升来源于LampOff和BasketballInHoop这一类任务相关点云占比极低的任务，通过Task-ROI提取，Prior-GA获得了显著的成功率，而IGA因淹没在了大量的任务无关点云而完全失败。在HangFrame-OnHanger这个需要对齐精度的任务中，Prior-GA也体现出了优势，如**Figure5**所示。

![[result3.jpeg]]
**Figure5:**  该图展示了 HangFrameOnHanger这个任务的对齐效果。左边为IGA，右边为Prior-GA.

---
## 4. 总结与讨论

$\qquad$本文针对IGA在点云对齐中因任务相关点云占比过低导致的注意力稀释与同质化演示中任务语义缺失问题，提出了Prior-GA（先验引导的几何对齐）方法。通过引入视觉语言模型从演示视频中提取任务关键区域，并结合FPFH粗配准与ICP精配准实现高效局部映射，Prior-GA仅需单次演示即可完成对齐，缓解了IGA对多样化数据的依赖。在RLBench环境下的实验表明，Prior-GA将任务成功率从IGA的27.78%显著提升至72.22%，尤其在任务相关点云占比极低的场景中，有效改善了IGA因大量任务无关点云干扰导致的对齐失败问题。
$\qquad$关于本方法的局限，基于VLM的任务ROI提取模块严重依赖视觉图像，如遇到遮挡或分割掩码不能覆盖关键局部，常常会失败。这也是我选择测试的任务相比于上次复现少的原因。我认为主要的原因是我尝试用视觉去对齐点云，模态的差距较大；如果能直接搭建起视觉与3D点云的桥梁或者直接进行点云的语义分割，该模块会更加高效鲁棒。
$\qquad$IGA用来机器人任务还是太勉强了，主要原因是他预测的仅仅是一个位姿，而机器人操作需要的是轨迹。我认为条件动作生成的范式会更加适合机器人操作任务。我也在RLbench上初步尝试了Instant Policy,基于图扩散的Instant Policy在机器人操作任务上表现更好，事实上当前主流的视觉语言模型和世界模型的动作专家大多采用扩散模型或者流匹配。
$\qquad$与 Instant Policy 不同，现有模型在适配全新任务时通常依赖参数微调，这一过程往往需要大量的训练数据。即便是当前顶尖的模型 GEN-1，学习一项新任务仍需耗费约 1 小时的机器人数据。相比之下，Instant Policy 采用的上下文模仿学习（In-Context Imitation Learning）范式无需更新模型参数，不仅在理论上具备更高的样本效率，也更贴近人类通过观察即可习得新技能的学习模式。因此我认为在机器人对全新任务的适配上In-Context Imitation Learning的范式是值得探索的。
## 5 参考文献

1.     Vosylius V, Johns E. Few-Shot In-Context Imitation Learning via Implicit Graph Alignment. _arXiv_. Preprint posted online October 18, 2023:arXiv:2310.12238. doi:10.48550/arXiv.2310.12238
2.     Fang X, Huang BR, Mao J, et al. Keypoint Abstraction using Large Models for Object-Relative Imitation Learning. arXiv.org. October 30, 2024. Accessed May 22, 2026. https://arxiv.org/abs/2410.23254v1
3.     Wang S, You J, Hu Y, Li J, Gao Y. SKIL: Semantic Keypoint Imitation Learning for Generalizable Data-efficient Manipulation. _arXiv_. Preprint posted online 2025. doi:10.48550/ARXIV.2501.14400
4.  Kirillov A, Mintun E, Ravi N, et al. Segment Anything. In: 2023 IEEE/CVF International Conference on Computer Vision (ICCV). IEEE; 2023:3992-4003. doi:[10.1109/ICCV51070.2023.00371](https://doi.org/10.1109/ICCV51070.2023.00371)
5. Wang S, You J, Hu Y, Li J, Gao Y. SKIL: Semantic Keypoint Imitation Learning for Generalizable Data-efficient Manipulation. _arXiv_. Preprint posted online 2025. doi:[10.48550/ARXIV.2501.14400](https://doi.org/10.48550/ARXIV.2501.14400)



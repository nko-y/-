- 物理动画 

- 骨骼动画：skeleton(基础骨架) $\rightarrow$ skeletonmesh(设置了骨架的模型资源) $\rightarrow$ characterskeletoncomponent(带skeleton的pawn) $\rightarrow$ 动画蓝图/动画序列(动画控制模式)

  动画蓝图由一系列节点构成：

  - 基础动画资源：蒙太奇、曲线驱动动画、变形目标
  - 动画混合节点：混合空间、惯性混合、指定关节混合
  - 动画处理节点：IK、同步组、rootmotion、动画通知、动画重定向
  - 逻辑选择节点：动画状态机



AnimMode

- 动画序列：记录了某个特定时间点处的一个骨骼的位置、旋转和缩放比例
- 动画蓝图：父类是animinstance 把动画逻辑从character中解耦
  - EventGraph：包含update等生命周期函数，可以用GetOwner得到Character实例
  - AnimGraph：动画蓝图的核心控制实现动画如何播放最终输出到OutputPose



状态机

- 将不同的动画作为一系列状态，按照转换规则管理这些状态



动画混合

（1）直接混合：

- 全身混合
- 部分混合，LayerBlendPerBone按层混合过渡

（2）混合空间：

- 在动画序列中采样动画资源
- 瞄准偏移AnimOffset，采样动画后使用叠加的方式添加到BasePose上

（3）惯性混合：

- 传统混合会计算源姿态和目标姿态然后做插值混合，惯性混合通过计算前一个Pose最后的状态(速度,位置等)通过多项式拟合来过渡到下一个Pose



动画资源：

- 动画蒙太奇：引用动画资源用于提供逻辑交互
  - 编辑动画序列，可以将动画序列中的任意时间段序列组合在一起并对其进行分段分组
  - slots槽，当蒙太奇播放时会使用蒙太奇中对应槽的Pose来覆盖之前节点的Pose

- 曲线驱动动画
- 变形目标：类似blendshape附带动画曲线



动画处理节点

- 动画通知：notify(只触发一次) + notifyState(持续触发)
- 同步组：用于解决不同长度周期的步行和跑步动画混合(左脚混左脚，右脚混右脚)
  - 最简单的就是拉长动画(但是不一定能解决)
  - 打上同步组记录对应的左脚右脚

- RootMotion

- 动画重定向
- IK逆向动力学：
  - TwoBoneIK + 极向量约束
  - 启发式：CCDIK和FABRIK
  - FullBodyIK：用于处理全身的IK（UE426中使用的是雅可比计算，UE5使用的是改进的PBD）
  - Jacobian法：以各个关节的旋转 $q$ 为变量可以表示出端点的运动 $f(q)$，这样可以直接将其当做一个优化问题进行梯度下降求解，或者可以求出 $\frac{\part f(q_i)}{\part q_i}$ 组成雅可比行列式 $q$ 然后用各类雅可比求逆的方法直接求解，或者用Gauss-sedial迭代求解



物理动画

- RigidBody布娃娃
  - 可以给每个关节设置包围盒进行物理模拟及运动约束
  - AnimDynamic：为骨骼添加约束(线性、角度、平面)



动画调试

- 动画蓝图选择attach实例然后观察对应的Pose和运行流向
- 控制指令 ShowDebugAnimation、AnimationInsights



AnimationBlueprintLinking动画蓝图链接

希望在动画蓝图A中重用动画蓝图B的部分

- 链接特定动画蓝图
- 链接动画层
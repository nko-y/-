#### 原理

（1）目的：Sequence由Pose构成，动画状态机是以Sequence为基础单位进行播放的这样就存在滑步的现象(实际输入和动画播放的动作不对应)，MotionMatching就是解决了输入和动画不匹配的问题(因为他是以pose为单位的)

（2）问题：效率低、占用数据内存大、有延迟不适合动作游戏

（3）算法：每个Pose的特征数据(位置、旋转等信息)与所有pose做差平方求出最小Cost

基础Cost：Heading、Pose、Postion、Trajectory

- Trajecotry参数：

  - Domain：Pose的上下文信息

    - 有两种类型
      - Time：以时间为偏移单位记录这个Pose周围的Pose数据
      - Distance：以当前Pose为原点选取相对于这个位置偏移Offset距离的Pose，根运动

    - 越界了怎么办：
      - 动画为Loop时直接循环采用
      - 不为Loop则尝试采样Lead in Sequence和Follow Up Sequence
      - 如果都为空则会通过Extrapolation Parameters进行预测(使用时间段内平均角速度和线速度)

  - Weight：这一项所占权重

- Pose参数：
  - 左右脚骨骼数据(速度、位置、旋转、相位)
    - 相位：读取Pose骨骼相对于Root的位置后将最大最小值记录下来中间用sin cos模拟
  - SampleTimes，采样偏移值可以通过这个值采样其他数据默认为0

- Heading参数：
  - 与Pose一致只不过可以选定某个轴
  - Input Query Pose：
    - Use Continuing Pose选择使用预处理读取动画序列中的数据，比较节省性能，充分利用数据不需要History节点
    - Use Character Pose实际移动用到的Pose数据，需要用History节点进行保存

- Position参数：
  - 和Pose参数一致但是只能读取Position

（4）Motion Matching节点

- 传入 Trajectory(CharacterMovementTrajectory数据)，传出保存到History
- Search Throttle Time，不会每帧搜索因为性能会受到严重影响，只有距离上次搜索超过这个Time阈值才会搜索
- Blend Time，Max Active Blends，Blend Profile，Blend Option用于混合

（5）调节权重

- Data Preprocessor是否对特征数据进行归一化
- Continuing Pose Cost Bias保持当前sequence的程度防止出现频繁跳跃
- Base Cost Bias每个Pose加上这个权重值用于Animal Notify State时重写
- Mirror Mismatch Cost Bias给所有镜像动画额外添加的Cost

（6）单独调整某些动画的权重 $\rightarrow$ Rewind Debugger调试

- Block Transition :防止Search的时候进行跳转，即标有Block Transition标记的Pose，会被忽略掉
- Exclude From Database：上面说过全局删减首尾部分或者单个动画修剪首尾部分，带有Exclude From Database的片段会直接从数据库中剔除掉，不希望被使用的Pose
- Override base cost bias:上面有控制全局的，这个是对全局的值进行覆盖，比如希望某个片段优先被使用或者少使用，需要用到这个
- Override Continuing Pose Cost bias:重写上面Continue Cost的值

（7）性能优化：

- Search Throttle Time减少Search次数
- PoseSearchMode：暴力算法 or KDTree
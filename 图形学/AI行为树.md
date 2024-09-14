行为树基础

（1）行为树可以看做AI的大脑，他是一个自上而下的树结构，包含四种类型的节点

- Tasks节点，一般位于叶节点的位置控制AI执行一系列的行为，例如MoveTo
- Composites节点，决定Tasks节点的执行顺序
  - Selectors节点，基于一些条件决定执行哪些子节点，用作分支选择
  - Sequences节点，从做到右依旧执行其子节点
  - Paralles节点，一次执行多个子节点
- Decorators，修饰一个composites节点来决定子节点是否可以执行，或者基于存储数据来决定是否终止运行
- Services节点，将附着在Composites节点上的附加类型节点，用来检测和更新数据进而做出决定

（2）运行：从根节点开始向下执行一个或多个Tasks节点，在执行过程中通常会在一个Composites节点中一直执行，当其他分支的触发条件满足时可以直接跳转到对应分支而不必等待Sequence执行完成



AI组件

（1）创建组件并关联

- 行为树AICharacter_BehaviourTree
- 黑板AICharacter_Blackboard包含所有用来给AI做决定的信息
- AICharacter_Controller控制AI的actor



赋予AI能力

（1）赋予AI视力，加入视觉回调函数当收到信息后将黑板值设置为True之后触发对应选择节点到MoveTo行为

（2）加入巡逻状态，没有看到玩家时处于自由移动状态



EQS场景信息查询系统(Environment Query System)

（1）EQS基本原理：在场景中生成一堆点然后遍历查看点是否满足条件，最后返回一个最适合的点

- Distance距离测试节点
- Dot点乘
- Gameplay Tags标签进行查询
- Pathfinding采样点域内容进行导航寻路器测试
- Project投射测试
- Trace射线测试
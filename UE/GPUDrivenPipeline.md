#### 静态合并

（1）在编辑器里将不同的instance合并成一个大的mesh

- 优点：合并过程不占用运行时，可以堆不同mesh不同instance进行合并
- 缺点：
  - 合并的mesh拥有一个比较大的包围盒，部分instance即使不在相机视线内也不会被剔除
  - 索引buffer中有大量重复索引(因为是按照instance合并的)

（2）动态合并：有些时候我们也会在游戏运行的时候进行instance合并



#### Instancing技术

（1）对于具有相同mesh的instance，我们可以通过一次drawcall把他们渲染出来，需要一个单独的instance buffer记录每个instance特有的数据并在VS中根据instance_ID来获取这些数据

（2）Merge-Instancing：解决Instance不同mesh间Instance合并的问题

- 过程：
  - 将不同mesh的顶点和索引合并到一个大的Vertex Buffer和Index Buffer中
  - 核心思想：每个mesh具有相同的索引数量，如果索引数量相同那么instance index就可以通过vertex index求得了

（3）Mesh Cluster：

- 不同mesh顶点数相差很大，需要将mesh拆分成多个cluster，cluster具有固定数量的索引，如果cluster不够64就需要加入退化三角形进行补齐，这样就可以通过vertexindex求出clusterindex再从cluster信息中读取instanceindex了



#### GPU Driven Render Pipeline

核心：将原本CPU做的事情全交给GPU来做

（1）COARSE FRUSTRUM CULLING

- 视椎体剔除：场景组织为八叉树/BVH树，利用这些加速结构快速剔除掉不在视椎体内的instance，现在可以使用CS在GPU中处理
- GPU端Hierarchical Z-Buffering做遮挡剔除

（2）拆分Instance：通过测试的instance会被拆分为cluster（可能也会被拆分为chunk再拆分为cluster）

（3）Cluster剔除

（4）三角面片剔除

- 背面剔除 和 零区域剔除，透视除法转换到NDC空间
- Depth Culling Hi-z
- Small Primitive Culling
- Frustum Culling
- Index buffer compaction



#### Nanite

（1）Instance Culling && Persistant Culling

- 以Mesh为单位先进行简单的剔除
- 将Mesh的三角形分为cluster并组织成BVH，使用HZB层次剔除
  - 设置一个全局FIFO队列，computeshader固定个线程去取任务，每个任务即为BVH的节点判断(生产者消费者)
- HiZ整个阶段分为两个Pass：MainPass + PostPass
  - MainPass：使用上一帧的HZB
  - PostPass：使用当前帧再去构建一个HZB再去处理一遍被遮挡了的对象

（2）Rasterization

- 每个cluster根据屏幕空间大小送至不同的光栅器
  - 大三角形和非Nanite基于硬件光栅
  - 小三角形基于compute shader写成的软光栅（因为小三角形用硬光栅会多计算不必要的像素点）
- Visibility Buffer：不同于GBuffer里面存储的内容需要存储的数据更少
  - R：0-6bit三角形ID
  - R：7-31bit ClusterID
  - G：0~31bit深度信息
- 软光栅基于扫描线算法：每个cluster启动一个computeshader
  - 计算并缓存所有ClipSpaceVertexPosition到Shared memory
  - CS每个线程读取对应三角形的Index Buffer和变换后的Vertex Position，计算出三角形边执行背面剔除和小三角形剔除
  - 利用原子操作完成Z-Test并写入VisibilityBuffer

（3）Emit Targets

- Emit Scene Depth/Stencil/Nanite Mask/Velocity Buffer，将Nanite纹理合并
  - 如果Nanite Mask表示这是Nanite Mesh，就将深度写入scenedepthbuffer，是否贴花写入scenestencilbuffer，上一帧位置计算的motionvector写入velocitybuffer

- Emit Material Depth：写入Material ID Buffer
- Deffered material：
  - nanite的shader是在screenspace执行的(将可见性和材质解耦)
  - 不是每种材质一个全屏pass
    - 例如屏幕大小800×600分成8×8的块共100×75个，Emit Targets之后用CS统计每个块内MaterialID种类
  - Material Depth Buffer设置为Stencil Target比较函数相等时才会执行

（4）整体先非Nanite的GBuffer再Nanite的Visibility Buffer




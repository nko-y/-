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



#### Compute Shader

不需要通过图形流水线也可以直接利用GPU的硬件单元的并行计算

（1）与传统渲染管线的比较：

- 传统渲染管线
  - CommandProcessor接收CPU发送的渲染绘制命令
  - 将命令转发给Graphics Processor单元，这些单元将顶点着色器需要运算的任务提交给GPU中的Compute Unit(在英伟达中被称未SM, Streaming MultiProcessor)
  - 光栅化阶段，由Rasterizer光栅化器完成，完成后继续回到Compute Unit中进行着色
  - 最后渲染完成的Render Target提交给FrameBuffer

- 计算着色器
  - 同样接受CommandBuffer发送的执行命令
  - 直接将指令提交到ComputeUnit开始计算，不需要经过GraphicsProcessor和其他图形管线单元处理

（2）Compute Shader运行方式

- 硬件基础
  - SIMDUnit + LocalDataShared + ScalarRegister/VectorRegister(寄存器)  $\rightarrow$ Compute Unit
- 简单CS例子
  - 每个ComputeUnit都对应执行一个ThreadGroup工作
  - ThreadGroup中每个Thread都会打包成WaveFront的形式发送给SIMD
    - AMD 64线程打包为一个WaveFront
    - 英伟达32线程打包为一个Wrap
  - numthreads线程组包含线程个数时应设置为wavefront的整数倍

```c++
#pragma kernel CSMain // CSMain是执行的compute kernel函数名

RWTexture2D<float4> Result; // CS中可读写纹理

[numthreads(8,2,4)]  // 线程组中的线程数,4张横向长度8竖向长度未2的表格
void CSMain(uint3 id : SV_DispatchThreadID){
    Result[id.xy] = float4(id.x & id.y, (id.x&15)/15.0, (id.y&15)/15.0, 0.0)
}
```

```c#
// C#端
var mainKernel = _filterComputeShader.FindKernel(_kernelName);
ComputeShader.GetKernelThreadGroupSizes(mainKernel, out uint xGroupSize, out uint yGroupSize, out _);
// 使用多少个ThreadGroup, (4,3,2)表示4*3*2个线程组，2张横向长度为4竖向长度未3的表格，表格的每一格都是一个numthread(8,2,4)的threadgroup
cmd.DispatchCompute(ComputeShader, mainKernel, 4, 3, 2);
```

- 线程间数据交换
  - 两个ThreadGroup之间交换数据会通过L2缓存
  - ThreadGroup内两个线程交换数据时会通过Local Data Share(LDS)速度非常快

- Vector Register & Scalar Register
  - non-uniform：有些变量在不同线程间有不同的数值，变量在线程中独立。每个变量需要 $64\times$ size_of_variable空间
  - uniform：有些变量在不同线程间完全相同，变量在线程间是共享的，我们称之为uniform。每个变量只需要 $1\times$ size_of_variable空间

- 注意事项
  - 如果一个线程组只分配了4个线程，一个WaveFront依旧会打包64个线程，SIMD一次执行16个线程所以为了这个WaveFront执行了4此，绝大部分都是空算
  - 避免在Compute Shader中出现分支

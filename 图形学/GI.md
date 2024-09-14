GI



采样方法

（1）Monte Carlo Ray Tracing(Offline)

- 如何更好地sampling？按照PDF采用且PDF和积分更可能接近

（2）Photon Mapping：从相机看转换为光源出发，光打出很多光子停留在物体表面然后人眼去收集



RSM方法

（3）RSM(Reflect Shadow Map)：从光的角度看shadowmap就是第一次照亮的地方，那是不是把这些点irradiance收集起来就是最终的贡献。对ReflectShadowMap每个点都算一遍(连线平方衰减) <span style="color:red">给出了一种注入光线的方法</span>

- 优化1：Cone Tracing，随机去打光线打到远的地方cone大些打到近的地方cone小些。
- 优化2：对屏幕的像素每隔几个采一次然后中间插值共用 $\rightarrow$ 根据法线等信息需要抛弃掉插值重新进行采样

（4）(弃)Light Propagation Volumes(LPV)

- 将场景体素化空间中每一点的光照都可以用SH表达，由表面的voxel向空间中去扩散。<span style="color:red">给出了一种体素中更新光的方法</span>

（5）(弃)SVOGI(Sparse Voxel Octree for Real-time Global Illumination)<span style="color:red">一种复杂的体素划分方法</span>

- 对空间体素划分，保守光栅化，再薄再小的三角形可以voxel起来，然后收集surface voxel，可以用八叉树
- 空间的体素是不均匀的
- Shading With Cone Tracing：采样采的是cone

（6）Voxelization Based Global Illumination(VXGI) <span style="color:red">一种简单的体素划分方法</span>

- clipmap：离眼睛近的地方GI重要(很密的voxel)远的地方可以粗略，但是在ScreenSpace差别不太大
- Voxel Update：用世界空间来映射clipmap的相同位置
- 每个面Voxel有一个透光率，沿着ConeTracing的时候是一层一层叠加过去的
- RSM注入光线，对于每个屏幕像素进行ConeTracing,，然后不断alphablend累乘

- 问题：
  - opacity计算是估计的会存在lightleaking，比如某个voxel遮住上半另一个voxel遮住了下半



（7）ScreenSpaceGlobalillumination

- 知道屏幕上一个点(normal)，知道相机方向，直接射ray
- screenspace raymarching：找到某一点depth比这一点小就是挡住了，可以用Hi-Z加速

- 邻近点可以共用插值
- 问题：screenspace如果屏幕空间没有就拿不到



Lumen

（1）Phase1：Fast Ray Trace in Any Hardware，SDF(Signed Distance Field)

- SDF是空间的物体的对偶表达，原始的三角形表达是离散的且有很多冗余数据表达三角形的连接，SDF是连续可微的
- 如何表达？
  - 每个mesh做一个sdf，各个sdf可以合成为一个整个的sdf
  - 特别细怎么办？把物体撑开来一些
- RayTracing with SDF：这一步跨多远直接从SDF里面读取就可以了，而且负数也可以trace回来。ConeTraing时可以预估遮挡率
- SDF可以做LOD，而且梯度可以表达其法向，从而可以做模型简化
- 远处的物体：GlobalSDF加速
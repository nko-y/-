#### Cascaded Shadow Map

（1）基本步骤：

- 摄像机视锥分割
  - 目标每一级的CSM阴影分辨率在投影到相机屏幕空间时有着相似的分辨率，设dp为摄像机近平面的一小段距离，ds是shadowmap平面的一小段距离，$\frac{dp}{ds}=n · \frac{dz}{z·ds}\frac{cos(\phi)}{cos\theta}=const$ 所以可以计算出 $z=n*(\frac{f}{n})^s$ 
    - 对 $s$ 进行等比分割 $z_i = n*(f/n)^{i/N}$
    - 与线性分割进行一个结合 $z_i = lerp(n*(f/n)^{i/N},n+\frac{i*(f-n)}{N}, \lambda), i=1...N$ 
- 子视锥包围盒计算
  - 最紧致的包围盒(包围住四个顶点)：这时候光源旋转时包围盒也会发生旋转尺寸发生较大的改变，从而引起分辨率变化产生阴影边缘闪烁的瑕疵
  - 最大方形包围盒
  - 最大球形包围盒，紧致球形包围盒
- 光源投影矩阵计算
- ShadowMap贴图渲染
- ShadowReceive计算
  - 层级选择
    - View Distance Selection：camera forward向量上的投影和Cascade分割距离做对比来判定落入哪个子视锥
    - Bounding Sphere Selection：只适用于球包围盒，计算像素世界坐标到包围球球心的距离
    - Map Selection：直接像素投影到shadowmap上如果UV在0-1内则命中

- 分帧更新：
  - 每帧绘制所有层级的 shadow map 可能会性能开销大，因此我们可以对远处阴影的 shadow map 进行更低频率的更新：对于高层级的 shadow map，我们不必要每帧都更新，而是可以每 n 帧重新绘制一次。

（2）
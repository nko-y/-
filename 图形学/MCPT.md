基本原理：使用蒙特卡洛积分近似渲染方程，通过多重重要性采样加速收敛



#### 蒙特卡洛积分

（1）对于复杂积分可以用任意概率密度函数的采样进行近似
$$
\int_a^b f(x)dx \approx \frac{1}{N}\sum^{N}_{i=1}\frac{f(X_i)}{pdf(X_i)} 
$$
（2）重要性采样：如果概率密度函数与 $f(x)$ 越接近收敛的速度也越快

（3）如何采样PDF？先求出概率分布函数CDF，对CDF均匀采样再带入CDF的反函数即可得到按照PDF采样的结果

- 设随机变量 $X\sim pdf(x)$ ，记累积分布函数 $F_X(x)=P(X\leq x)$ 现希望从该概率分布中采样
- 假设随机变量 $Y\sim U(0,1)$ 不难得出该满足均匀分布的随机变量的累积分布函数为 $P(Y\leq y)=y$
- 现构造另外一个函数 $Z=F^{-1}_X(Y)$ 则 $P(F_X^{-1}(Y)\leq x)=P(Y\leq F_X(x))=F_X(x)$ 



#### 渲染方程

$$
L_o(p,w_o)=L_e(p,w_o)+\int_{\Omega^+}L_i(p,w_i)f_r(p,w_i,w_o)(n·w_i)dw_i
$$

（1）辐射度量学

- 能量(单位焦耳J）(radiant energy)：每个光子都有特定的波长，并携带了一定的能量，能量和波长的关系是 $Q=\frac{hc}{\lambda}$ 
- 光通量(功率，单位瓦特W)(radiant flux/power)：一段时间做的功，光的功率就是光通量，也就是单位时间内通过某个区域的总能量 $\Phi=lim\frac{\Delta Q}{\Delta t}=\frac{dQ}{dt}$ 
  - 光源的总辐射一般用光通量来表达
  - 对于一个光源和包含它的两个不同半径的球面，光通量是一样的但是二者在光源下表达出来的亮度是不同的(因为他们的面积不同)
- 辐射照度(irradiance)：给定有限的区域 $A$，辐射照度 $E=\frac{\Phi}{A}=\frac{d\Phi}{dA}cos\theta$  
  - 实际上有两种辐射照度，辐射入射度(E)和辐射出射度(M)
  - 那么对于围绕光源的一个球面，辐射度即为 $E=\frac{\Phi}{4\pi r^2}$ 
  - 朗伯定律：到达某个表面的光的能量正比于光的方向和表面法向量之间的余弦值。考虑一个面光源正着打到表面和斜着打到表面，他们打到表面的光通量是一样的，但是他们打到的面积分别是 $A*cos\theta$ ，此时单位点的亮度是和这个面积成反比的
- 光强/辐射强度(I)(radiant intensity)，光源某个角度的发射功率 $I = \frac{\Phi}{4\pi}$ 更一般的 $I=lim \frac{\Delta \Phi}{\Delta w}$ 
  - 光强和辐射照度没有关系，只是从不同的角度对光通量进行了分析
- 辐射亮度/辐射率(radiance)：辐射照度在立体角上的微分 $L(p,\omega)=lim\frac{\Delta E_{\omega}(p)}{\Delta \omega}=\frac{dE}{d\omega cos\theta}$  
  - **辐射照度是入射光线在单位面积上的光通量，辐射亮度是出射光线在单位立体角上的光通量** 
  - **辐射照度是不具有方向性的代表所有方向在dA上的光通量之和，辐射率是带方向的代表dA单位角度收到的光通量**
  - $E_w$ 表示的是垂直于方向 $\omega$ 的表面的辐射照度，这种表示正好抵消了郎伯定律里面的cos
  - 综上辐射亮度可以定义为光通量在垂直面积和立体角上的微分 $L=\frac{d\Phi}{dw dA^{垂直}}$

（2）BRDF：双向反射分布函数，给定一条入射光的时候，某一条特定的出射光线的性质是什么样的

- 计算公式 $f(l,v)=\frac{dL_o(v)}{dE(l)}$ 出射光辐射率的微分和入射光辐照度的微分，因为一条入射光会分解为各个立体角上的反射光
- $E=\frac{\Phi}{A}=\frac{d\Phi}{dA}cos\theta$  和 $L(p,\omega)=lim\frac{\Delta E_{\omega}(p)}{\Delta \omega}=\frac{dE}{d\omega cos\theta}$  这个除可以从打入的角度考虑，打入就要进行除法
- 那其实本质上就是 $f_r(w_i\rightarrow w_r)=\frac{dL(w_r)}{dE(w_i)}=\frac{dL(w_r)}{L(w_i)cos\theta_i dw_i}$ 

（3）PBR(基于物理的着色)

三个条件：基于微平面的表面模型、能量守恒、基于物理的BRDF

- 微平面模型：
  - 平面越粗糙，这个平面上的微平面的排列就越混乱，入射光就趋向于向完全不同的方向发散开来。
  - 使用粗糙度，光线方向和视线方向的半程向量 $h=\frac{l+v}{||l+v||}$ 表示

- 能量守恒：
  - 折射和反射：基于物理的渲染(平面上的每一点所有的折射光都会被完全吸收而不会散开)，次表面散射(光线会继续沿着随机方向发散然后再和其他的粒子碰撞直至能量完全耗尽或者再次离开这个表面)。折射和反射和应该正好占比为1.0
- 基于物理的BRDF：
  - 接受入射光方向、出射光方向、平面法线和一个用来表示微平面粗糙程度的参数【如果一个平面拥有完全光滑的表面，那么对于所有的入射光线 $w_i$ 只有一束出射光线 $w_o$ 拥有1.0的返回值其他都会返回0.0】
  - Cook-Torrance BRDF： $f_r=k_d f_{lambert}+k_sf_{cook-torrance}$ ，$k_d$ 被折射部分能量所占比率，$k_s$ 被反射部分的比率，$f_{lambert}=\frac{c}{\pi}$ ，$f_{cook-torrance}=\frac{DFG}{4(w_o·n)(w_i·n)}$ 
    - D法线分布函数(normal distribution function)，估算在收到表面粗糙度的影响下，朝向方向与半程向量一致的微平面数量，用于估算微平面的主要函数。
    - F菲涅尔方程(Fresnel Rquation)，不同表面角下所反射光线所占的比率**【菲涅尔项直接给出了KS的值】** 此时 KD = (1.0-KS)*metallic。$F_{Schlick}(h,v,F_0)=F_0+(1-F_0)(1-(h·v))^5$ 
    - G几何函数(Geometry Function)，描述微平面自阴影的属性，当一个平面相对比较粗糙的时候，平面上的微平面可能遮挡住其他的微平面。需要考虑观察方向(几何遮蔽Geometry Obstruction)和光线方向向量(几何阴影Geometry Shadowing) smith's method

（4）HDR+Gamma校正

- 由于PBR计算都在线性空间，最后需要进行gamma校正。同时我们希望所有光照的输入都尽可能的接近他们在物理上的取值，这样他们的反射率或者说颜色值就会在色谱上有比较大的变化空间，`Lo`作为结果可能会变大得很快(超过1)，但是因为默认的低动态范围（LDR）而取值被截断。所以在伽马矫正之前我们采用色调映射使`Lo`从LDR的值映射为HDR的值

$$
color = color / (color + vec3(1.0))\\
color = pow(color, vec3(1.0/2.2))
$$

- 反射率(albedo)在美术人员创建的时候就已经在sRGB空间了，因此我们需要在光照计算之前先把他们转换到线性空间，环境光遮蔽贴图(ambient occlusion maps)也需要我们转换到线性空间。不过金属性(Metallic)和粗糙度(Roughness)贴图大多数时间都会保证在线性空间中。

（5）BSDF = BRDF + BTDF



#### 重要性采样

（1）非重要性采样：半球均匀采样。

- 首先计算归一化系数 $\int_{\Omega^+} c *sin\theta d\theta d\phi=1$ 可以得到 $c=\frac{1}{2\pi}$ 
- 于是球面 $\theta$ 和 $\phi$ 的联合概率密度函数为 $\frac{1}{2\pi}sin\theta$ 
- 由于 $pdf(\theta)=\int_0^{2\pi}p(\theta,\phi)d\phi=sin\theta$ 且 $pdf(\phi|\theta)=\frac{pdf(\theta,\phi)}{pdf(\theta)}=\frac{1}{2\pi}$ ，由此可以计算 $F(\theta)=1-cos\theta$ 和 $F(\phi|\theta)=\frac{\phi}{2\pi}$ 
- 设 $\epsilon_1,\epsilon_2\in[0,1]$ 利用逆分布得到 $\theta=cos^{-1}(1-\epsilon_1),\phi=2\pi\epsilon_2$ 最终再将其转换为三维坐标  

（2）重要性采样

$L_o(p,w_o)\approx \frac{1}{N}\sum\frac{L_i(p,w_i)f_r(p,w_i,w_o)(n·w_i)}{p(w_i)}$，正比于被积函数的一部分

- 正比于 $n·w_i=cos\theta$ 

  - 首先计算归一化系数  $\int_{\Omega^+} c *sin\theta cos\theta d\theta d\phi=1$ 可以得到 $c=\frac{1}{\pi}$ 
  - 于是球面 $\theta$ 和 $\phi$ 的联合概率密度函数为 $\frac{1}{\pi}sin\theta cos\theta$ 
  - 由于 $pdf(\theta)=\int_0^{2\pi}p(\theta,\phi)d\phi=sin2\theta$ 且 $pdf(\phi|\theta)=\frac{pdf(\theta,\phi)}{pdf(\theta)}=\frac{1}{2\pi}$ ，由此可以计算 $F(\theta)=1-cos^2\theta$ 和 $F(\phi|\theta)=\frac{\phi}{2\pi}$ 
  - 设 $\epsilon_1,\epsilon_2\in[0,1]$ 利用逆分布得到 $\theta=cos^{-1}\sqrt{(1-\epsilon_1)},\phi=2\pi\epsilon_2$ 最终再将其转换为三维坐标  

- 正比于BRDF

  - 正比于漫反射项 $pdf(w_i)=\frac{cos\theta}{\pi}$ 

  - 正比于法线分布函数
    - 由于法线分布函数满足 $\int_{H^2}cos\theta_h D(w_h)dw_h=1$ 即每单位立体角所有法向为 $w_h$ 的微平面的面积
    - 利用Cook-Torrance BRDF推导可得 $\frac{dw_h}{dw_i}=\frac{1}{4(w_h·w_i)}$ 带入上式得到 $pdf(w_i)=\frac{cos\theta_h}{4(w_h·w_i)}D(w_h)$ ，但其实这里可以只对 $w_h$ 进行采样计算
  - 两者结合 $k_sD(w_h)\frac{cos\theta_h}{4(w_h·w_i)}+(1-k_s)\frac{cos\theta_i}{\pi}$ ，首先生成一个随机数 $r_1$ 如果其小于 $k_s$ 则用漫反射来采样否则对镜面反射采样



#### 多重重要性采样

（1）原理：将多种采样分布结合起来的无偏估计的方法，例如综合考虑对出射方向采样和对光源的采样

（2）采样的组合权重：倘若在一个点你采样的pdf越大，其实也就说明了这个重要性采样分布更加擅长这个区域，理应给它更高的权重，而如果pdf小，意味着该重要性采样分布对这个区域没有什么自信，当然应该减小权重，来降低误差，从而进行加权求和



#### 其他

（1）什么时候停下来，俄罗斯轮盘赌：设定一个概率 $P$，有 $P$ 概率光线会继续递归并设置返回值为 $L_o/P$，有 $1-P$ 的概率光线停止递归并返回0，这样计算得到的结果是无偏的 $E=P*(L_o/P)+(1-P)*0=L_o$  

（2）低差异化序列：高效地生成在高维空间均匀的随机数

- Discrepancy定义：$D_N(P)=sup_{B\in J}|\frac{A(B)}{N}-\lambda_s(B))|$ ，对于一个在$[0,1]^n$空间中的点集，任意选取一个空间中的区域B，此区域内点的数量A和点集个数的总量N的比值和此区域的体积λs的差的绝对值的最大值，就是这个点集的Discrepancy。分布越均匀的点集，任意区域内的点集数量占点总数量的比例也会越接近于这个区域的体积。











参考文献：

1. [重要性采样和多重重要性采样在路径追踪中的应用](https://blog.csdn.net/qq_38065509/article/details/115260040) 
2. [learnopenglcn-光照-理论](https://learnopengl-cn.github.io/07%20PBR/01%20Theory/)
3. [怎么理解分布函数的分布函数服从均匀分布](https://www.zhihu.com/question/430491621/answer/1842937272)
4. [低差异序列（一）- 常见序列的定义及性质](https://zhuanlan.zhihu.com/p/20197323?columnSlug=graphics)


# 实时渲染一：软阴影实现

本篇文章深入探讨了两趟渲染（Two-Pass Algorithm）、Percentage Closer Filtering（PCF）阴影软化技术，以及Percentage Closer Soft Shadows (PCSS)软阴影实现算法的核心理念和运行原理。同时，我们也将讲解各种算法的优化策略及其背后的数学逻辑。

![image-20230731102509105](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230731102509105.png)

### Shadow Mapping

本章内容：

* What is Shadow Mapping
* 2-Pass Algorithm
* Issues in Shadow Mapping - Shadow acne
* Bias
* Second-Depth Shadow Mapping (SDSM)
* Issues in Shadow Mapping - Jagged Shadows

#### Shadow Mapping 介绍

Shadow Mapping 是用于生成物体阴影的一种方法，通过对灯光源视角的场景进行渲染并创建深度图（也称为阴影图）来实现阴影效果。

#### 两趟渲染（2-pass algorithm）

Shadow Mapping 的过程可以被划分为两个阶段，通常这种策略被称作 "2-pass algorithm"，或被直译为“两趟算法”。这两个阶段分别被称作“光照处理阶段（Light Pass）”和“相机处理阶段（Camera Pass）”。

![image-20230731094629963](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230731094629963.png)

**Light Pass**：我们从**光源**的视角渲染场景，并记录每个像素到光源的深度信息，也就是距离光源最近的物体的信息。这个过程产生了一个深度图，也就是 Shadow Map (SM)。基本上，Shadow Map 是场景中每个可见点相对于光源的深度值的纹理。

**Camera Pass**：我们从**摄像机**的视角渲染场景。对于摄像机视角下的每个像素，我们将其转换到光源视角下，并查找其在 Shadow Map 中的深度值。如果该像素的深度值大于 Shadow Map 中存储的深度值，那么说明该像素被其它物体遮挡，处于阴影之中。否则，该像素被光源照亮。

#### 阴影痤疮（Shadow acne）

这个问题在表面与光源平行时最严重，因为这时候从光源看过去的深度和从相机看过去的深度几乎是一样的，很可能会造成**错误的自遮挡判断**。当阴影和物体表面接近平行时，阴影会出现类似条纹的模式。这是由于深度比较（深度缓冲）和深度图（阴影图）的分辨率有限导致的。

Shadow Map 的每个像素都对应于3D场景中的一块区域。在阴影图生成阶段，该区域中所有点的深度值会被合并并记录到一个像素中。这导致了一种深度不精确的现象，即阴影图像素的深度值可能并不完全等于原始3D空间中对应区域的实际深度值。

简单举一个例子，对于那些深度值略大于阴影图中记录深度的点，我们会错误地认为它们在阴影中，而对于那些深度值略小于阴影图中记录深度的点，我们会错误地认为它们在光照下。

我们可以在Unity中轻松复现这种 Shadow Acne。在Light组件下可以将Bias属性、Normal Bias属性调节为**0**并且Type属性设置为**Directional**，即可复现。如下图所示：

![image-20230803105828953](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230803105828953.png)

#### 阴影偏移（Shadow Bias）

![image-20230731102509105](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230731102509105.png)

为了减少自遮挡的问题，通常会添加一个**偏移值 (bias)** 到 Shadow Map 的深度中。这样就可以让物体的阴影稍微偏离物体表面，使得实际像素的深度值比深度图的深度值稍微近一点。这样，即使由于采样误差造成的深度不匹配，实际像素也能正确地被认为是被照亮的，减少自遮挡的现象。

但是，如果偏移值设置得过大，就会产生阴影分离 (Shadow Detachment) 的问题，即阴影看起来与产生它的物体分离了。另一方面，如果偏移量过小，可能无法完全解决自阴影问题。

#### Second-depth shadow mapping

Second-Depth Shadow Mapping (SDSM) 是 Shadow Mapping 技术的一种变体，它试图通过记录每个像素的第一和第二深度（从光源视角看）来改进基础的 Shadow Mapping 方法。

> 具体翻译我也不知道，下文直接使用缩写SDSM。

在传统的 Shadow Mapping 中，我们只关心每个像素的第一深度。而在 SDSM 中，我们会同时关心第一和第二深度，然后，我们使用这两个深度之间的中点作为阴影判断的深度。

![image-20230731112254244](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230731112254244.png)

然而，这种方法也有一些限制：

1. 首先，为了计算第二深度，**物体必须是完全封闭的（watertight）**。否则，第二深度的计算可能会出错。
2. 其次，这种方法的开销比基础的 Shadow Mapping 大。因为我们需要记录并处理两个深度值，而且还需要对每个像素进行两次深度测试。
3. 最后，虽然这种方法可以提高阴影的精度，但是它性价比不高。在许多情况下，基础的 Shadow Mapping 加上一些后处理技术，如阴影模糊或者阴影偏移，就已经能得到很好的结果，而且开销更小。

因此，工业界基本不会在实际项目中应用这个方法。

***

这也从侧面说明了一点：**实时渲染强调简单有效的方法**。

在实时渲染中，性能和效率至关重要。一个算法可能在理论上看起来很完美，但如果它过于复杂，以至于无法在实时环境中高效运行，那么它就没有太大的实用价值。

相反，一个相对简单甚至简陋但能有效解决问题的算法，通常会被优先考虑。

> RTR does not trust in COMPLEXITY

***

#### 锯齿状阴影（Jagged Shadows）

![image-20230731113221101](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230731113221101.png)

当阴影映射到物体上时，由于深度图（Shadow Map）的像素化，可能会在阴影边缘产生锯齿状的效果。这个问题尤其在深度图的分辨率比较低时会比较明显。

为了解决这个问题，可以尝试增加深度图的分辨率，或者使用某种形式的阴影平滑技术，如 Percentage-Closer Filtering（PCF）或 Variance Shadow Maps（VSM）。这些方法可以使阴影边缘看起来更加平滑，减少锯齿状阴影的出现。

### Percentage-Closer Filtering (PCF)

PCF用于消除阴影映射中的锯齿状边缘效果（即阴影边缘的抗锯齿）。

基本思想是在进行阴影测试时，不仅仅对当前像素进行测试，还对其周围的像素进行测试，然后对这些测试结果进行平均。这样可以得到一个更加平滑的阴影边缘。

实施步骤如下：

1. 对于渲染图像中的每一个像素（或者称为片段，fragment），执行多次（例如7x7次）深度比较。在这里，比较的对象是当前像素与阴影图中对应像素的深度值。
2. 然后，对这些比较结果进行平均。这里的平均操作是指对比较结果的二元值（阴影/非阴影）进行平均，而非对深度值进行平均。

![image-20230731132751055](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230731132751055.png)

例如，对于在地板上的点P，我们需要对其进行以下操作：

1. 将点P的深度值与阴影图中黑框内所有像素的深度值进行比较，例如，黑框是一个3x3的像素框。记录这些深度比较的结果，每个结果是一个二元值，表示点P相对于对应像素是在阴影中（0）还是非阴影中（1）。例如，得到如下矩阵：

$$
\left[\begin{array}{} 1 & 0 & 1 \\ 1 & 0 & 1 \\ 1 & 1 & 0 \end{array}\right]
$$

2. 对这些比较结果进行平均，得到的平均值代表了点P的可见性（visibility）。如果平均值接近1，那么点P主要处于光照之下；如果平均值接近0，那么点P主要处于阴影之中。上述矩阵的平均结果为 0.6667 。

也就是说，这个 Filtering 做的是将所有着色点的深度值设置为，其与很多次不同的阴影的遮挡关系比较的结果的平均。

除此之外， 我们还一些问题没有讨论。是否能够使用 PCF 来实现软阴影效果？Filtering的大小对阴影有什么影响呢？我们究竟要设置多大的像素框呢？

* 答案是肯定的。
* 小的过滤器尺寸会导致更锐利的阴影边缘，而大的过滤器尺寸会产生更软的阴影效果。
* 一般而言，偏向自然得场景一般都是用较大的 Filtering 来产生软阴影的效果。

但是请注意，这个过程并非基于物理的。虽然PCF能够实现一定的软阴影效果，但是其产生的仍然是假的软阴影，因为它并**没有考虑光源的大小和形状**，而真实的软阴影是由光源的大小和形状决定的。

### Percentage Closer Soft Shadows (PCSS)

生成软阴影，我们用PCSS。需要注意的是，我们上面学习的PCF原本的目的是为了消除阴影的锯齿。但是在实践的过程中我们发现在调节 Filtering 抗锯齿的时候可以让阴影效果近似于软阴影。于是，PCSS产生了。

PCSS 是一种高级阴影映射技术，它扩展了基本的PCF方法，通过考虑光源的大小来模拟阴影的模糊程度。

在真实世界中，一个大的光源（比如一片云朵遮挡太阳）会产生模糊的阴影，而一个小的光源（比如一个点光源）会产生锐利的阴影。PCSS模拟了这个现象，通过对光源大小的考虑，来改变阴影边缘的模糊程度。简单地说，就是一片阴影的不同地方应用不同大小的 Filtering ，而这个大小（阴影软or不软），我们定义为$\omega\_\text{Penumbra}$：

$$
w_{\text {Penumbra }}=\frac{\left(d_{\text {Receiver }}-d_{\text {Blocker }}\right) \cdot w_{\text {Light }}} { d_{\text {Blocker }}}
$$

![image-20230731142003749](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230731142003749.png)

也就是说，$\omega$越大，阴影越软。至此，我们就通过接受平面与遮挡物、遮挡物与光源与光源大小确定了软阴影过渡区域的大小。即可以将PCF的思想完美应用到PCSS上了。

在PCSS中，每个像素的阴影软度是独立计算的，取决于像素到光源的距离。而且距离光源越远的阴影边缘会更模糊，而距离光源越近的阴影边缘会更尖锐。那么，如何计算具体某个像素点的阴影呢？

#### 具体步骤

计算方法分为三个步骤：阻挡物搜索，Penumbra（半影）估计，以及百分比接近过滤。

> 顺带一提，一般而言面光源都是通过一个点光源来生成一张Shadow Map的。

* **第一步，阻挡物搜索**

确定哪些物体阻挡了光源，从而在其他物体上投下了阴影。从每一个像素点（Shading Point）出发向点光源搜索，看看有没有Blocker的遮挡，也就是说，在其对应的深度图（Shadow Map）中找到所有的阻挡物。**一般而言，遍历深度图中的一片区域是一个半径为R的圆**，并且对于这个区域内的每一个像素，我们会比较其深度值与需要渲染阴影的像素点的深度值。如果像素点（Shading Point）与点光源之间有遮挡，也就是说深度图中的像素深度小于需要渲染阴影的像素点深度，那么就说明深度图的那个像素点就是一个Blocker遮挡点。另外，在这一步中，我们还会计算出所有阻挡物深度的平均值，后面会用到。

但是，这个步骤的遍历区域应该取多大呢？

* 第一种方法：自己随便设置。简单省时。
* 第二种方法：一种启发式的方法。

我们看下面这一张图。假设Shadow Map在Near平面上的视锥体区域内，原因是：

1. 在光源视角下，只有落在光源的视锥体内的几何体才能产生阴影。
2. 创建一个Shadow Map时，我们是通过从光源的视角渲染场景来捕捉深度信息的，**因此会像处理近裁剪面一样处理Shadow Map**。

注意一下，把Shadow Map放在ZNear近裁切面这只是一种比喻，实际上Shadow Map和近裁剪面是两个不同的概念，它们存在于不同的空间（一个在光源空间，一个在相机空间）。

![image-20230801212625280](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230801212625280.png)

在Percentage-Closer Soft Shadows (PCSS)算法中，采样区域大小是由光源的大小和物体距离光源的距离共同决定的。具体的计算公式为：

$$
searchWidth = lightSize \cdot (receiverDepth - lightDepth) / receiverDepth
$$

其中，`lightSize`是光源的大小，`receiverDepth`是接收阴影的物体的深度，`lightDepth`是光源的深度。

通过上面的公式我们确定了不同Shading Point上的采样区域大小，进而在这个区域中搜索Blocker。具体的搜索方法是，如果采样点的深度小于当前像素的深度（表示采样点在当前像素的“前面”），那么我们就认为这个采样点是一个阻挡物（在光源与Shading Point之间的一个深度值就是我们计算的结果）。

取得所有的Block阻挡物的深度值之后，计算Blocker阻挡物的平均深度值。

既然我们已经找到了Blocker的平均深度值，接下来就已经完全把PCSS问题转化为了PCF问题。

* **第二步：Penumbra 估计**

上一步我们计算出了所有的Blocker的平均深度信息，这一步用这个信息来估计Penumbra（半影）的大小。光源越大，或者遮挡物越近，Penumbra就越大。

* **第三步：PCF（Percentage Closer Filtering）**

在估计的Penumbra范围内进行多次采样，每次采样都要对比深度值，然后将结果进行混合。

***

我们不难发现，每一个Shading Point都需要综合场景的所有的Blocker信息（或者周围的区域），这会导致严重的性能问题。接下来我会介绍一些优化方法。

但是PCSS本身还是非常值得学习的。一句话概括就是，**在深度图中使用可变大小的滤波器来模拟软阴影效果的技术。**

下图就是PCSS在实际游戏中的表现（消逝的光芒截图）：

![image-20230802151510959](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230802151510959.png)

### 复杂函数乘积积分的近似

当你有一个复杂的乘积形式的积分，如：

$$
\int_{\Omega} f(x) g(x) \mathrm{d} x
$$

，其中 f(x) 和 g(x) 是两个函数，Ω 是积分区间，这个积分可能很难直接计算。一种常见的解决方法是将它转化为一个更简单的形式：

$$
\int_{\Omega} f(x) g(x) \mathrm{d} x \approx \frac{\int_{\Omega} f(x) \mathrm{d} x}{\int_{\Omega} \mathrm{d} x} \cdot \int_{\Omega} g(x) \mathrm{d} x
$$

其中，这个近似公式在某些情况下是合理的，特别是在以下情况下：

* f(x) 和 g(x) 是相对独立的
* 积分区间 Ω 足够小
* f(x) 和 g(x) 都足够平滑，即它们在 Ω 上的变化是平缓的，没有突然的跳跃或尖峰。

#### numpy代码实践

下面现场写一段 python 程序体会一下这个近似公式：

```python
import numpy as np
from scipy import integrate

# 定义两个函数
def f(x): return np.sin(x)
def g(x): return np.cos(x)

# 定义积分的区间
a = 0 ; b = np.pi / 4

# 计算真实的积分值
real_val, _ = integrate.quad(lambda x: f(x)*g(x), a, b)
print("真实的积分值:", real_val)

# 计算 f 和 g 的积分
int_f, _ = integrate.quad(f, a, b)
int_g, _ = integrate.quad(g, a, b)

# 计算近似的积分值
approx_val = (int_f / (b - a)) * int_g
print("近似的积分值:", approx_val)
```

> 输出结果：
>
> 真实的积分值: 0.24999999999999997 近似的积分值: 0.2636965437895247

#### 应用到渲染方程中

渲染方程通常形式为：

$$
L_o(p, ω_o) = L_e(p, ω_o) + ∫_Ω f_r(p, ω_i, ω_o)L_i(p, ω_i)(n·ω_i) dω_i
$$

其中，$L\_o$是出射光线，$L\_e$是自发光线，$L\_i$是入射光线，$f\_r$是BRDF，$n$是表面法线，$ω\_i$和$ω\_o$分别是入射和出射方向，$Ω$是半球。

我们使用近似积分公式将渲染方程中的：

$$
∫_Ω f_r(p, ω_i, ω_o)L_i(p, ω_i)(n·ω_i) dω_i
$$

近似为：

$$
\frac{∫_Ω f_r(p, ω_i, ω_o) dω_i}{|Ω|} \cdot ∫_Ω L_i(p, ω_i)(n·ω_i) dω_i
$$

这样可以分别计算 $f\_r$ 和 $L\_i(n·ω\_i)$ 的积分，然后将它们相乘，大大简化了计算过程。

最后再说下，这种近似方法假设场景中的光照和反射率（即 BRDF）在积分区间上的变化不大。因此，如果场景中的光照和反射率分布相对均匀，这种近似方法可能会得到较好的结果。如场景中的光照和反射率变化非常大，这样近似结果的误差就会非常大。

### Deep look - PCF

卷积公式：

$$
[w * f](p)=\sum_{q \in \mathcal{N}(p)} w(p, q) f(q)
$$

PCF：

$$
V(x)=\sum_{q \in \mathcal{N}(p)} w(p, q) \cdot \chi^{+}\left[D_{\mathrm{SM}}(q)-D_{\text {scene }}(x)\right]
$$

### Reference

1. Percentage-Closer Soft Shadows - Randima Fernando NVIDIA Corporation
2. GAMES202
3. Real-Time Rendering 4th Edition

Unity3D的引擎熟练度不敢说精通，但是理解整个开发流程，以前写过完整的音乐游戏。

代码方面，学习过 事件处理系统，以前用.net开发过一些桌面小工具箱，结合API识别图片文字等等。

另外渲染方面系统学习过101，熟悉GPU渲染管线流程，光照、PBR、阴影这些都亲自写过代码。

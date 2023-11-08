# 实时渲染二：环境光照

## 202 Lecture5-7：环境光照与PRT

![image-20231016172835230](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231016172835230.png)

> 本文的手绘图片可以随意使用，我自己画的～

### Signed Distance Field（SDF）

SDF是一个函数，描述了空间中的每个点到某个表面的距离。这个“距离”带有一个符号，通常表示点是在表面的内部还是外部。

* 如果点在表面上，SDF的值为0。
* 如果点在表面外部，SDF的值为正，表示该点到最近表面的距离。
* 如果点在表面内部，SDF的值为负，表示该点到最近表面的距离。

在程序上，SDF通常表示为一个函数，它接受一个点（一般是三维向量）作为输入，并返回一个标量值，代表该点到最近表面的有符号距离。

例如，一个圆的SDF函数（在二维空间中）：

```c++
float circleSDF(vec2 point, vec2 center, float radius) {
    return length(point - center) - radius;
}
```

1. `point` 是你想要查询的位置。
2. `center` 是圆的中心。
3. `radius` 是圆的半径。
4. `length` 函数计算两点之间的距离。

![image-20231008145206080](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231008145206080.png)

具体来说，给一个点 `Point_1` ，通过SDF函数，就可以求出 `length` 。

#### Ray Marching

Ray Marching（尤其是Sphere Tracing）是SDF（Signed Distance Field）的最知名的应用之一。Ray Marching 使用SDF来逐步沿光线方向推进，直到找到或者接近场景中的表面。

概述Ray Marching如何使用SDF的：

1. **初始化**：从观察者的位置开始，为每个像素发射一条光线。
2. **查询距离**：对于光线上的当前点，使用SDF查询该点到最近物体表面的距离。
3. **前进**：沿光线方向移动距离，即你从SDF中获得的值。因为SDF给出的是到最近表面的距离，你可以安全地沿光线移动这个距离而不会错过任何表面。
4. **迭代**：重复查询距离和前进的步骤，直到你足够接近一个表面（SDF值接近0）或超出了最大步数/最大距离。
5. **着色**：一旦找到表面，可以计算光照、纹理等来着色该像素。

![image-20231008150850298](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231008150850298.png)

#### 生成软阴影

通常，在Ray Marching中，「安全距离」被用来快速找到场景中的表面。但是，这些距离也可以用来估算光源的遮挡程度，从而生成软阴影。

怎么做呢？

首先，使用常规的Ray Marching技术找到场景中的表面。这意味着你会沿光线迭代，使用SDF来确定每一步的距离，直到你接近或达到一个表面。

一旦找到表面，你可以从该点发射一条射线到光源，这条射线被称为阴影射线。

![image-20231008152147938](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231008152147938.png)

与Ray Marching相似，你会沿阴影射线迭代前进，再次使用SDF来确定每一步的距离。

![image-20231008153635637](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231008153635637.png)

一旦阴影射线的前进距离超过了从视点到光源的距离，或者超过了预定义的最大步数，步进过程就会停止。

上图是没有遮挡物的情况，如果有遮挡物，那么就需要进行「估计遮挡」步骤。也就是说，如果在前进到光源之前，SDF的值变得非常小（接近于0），那么这意味着我们碰到了另一个物体，因此原始的视点是在阴影中的。

![image-20231008155216422](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231008155216422.png)

但为了得到软阴影，我们不仅检查是否有遮挡，还考虑遮挡的程度。这通常通过考虑当前点到SDF表面的距离与到光源的距离之比来估计。较大的表面距离意味着较淡的阴影，而较小的表面距离意味着较深的阴影。

![image-20231008161522750](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231008161522750.png)

在实际光照中，当一个光源被一个物体部分遮挡时，它会产生一个阴影区域，这个区域分为两部分：**全影区**和**半影区**。

![image-20231008162411080](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231008162411080.png)

接下来引入一个概念，"Safe Angle"（安全角）。这个的安全角的概念与半影的产生直接相关。首先从一个Shading Point出发射向光源，在某一点处（Current Point）可以得到对应的SDF值SDF(Current Point)。如果这个SDF值比较大，那么Safe Angle越大，说明阴影并不硬（黑）。

![image-20231008165851400](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231008165851400.png)

上图中我只画出了其中某个Current Point，实际上是有好几趟SDF查询的，就像上面Ray Marching那样。实际来说，每一步进都会有一个Safe Angle，这里只需要取最小的即可。

![image-20231008171315840](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231008171315840.png)

那么问题来了，我们要怎么计算Safe Angle？直接计算一个反三角函数就可以了。

$$
\text{Safe Angle} = 2 \arcsin{\frac{SDF(p)}{|p-o|}}
$$

![image-20231008173205313](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231008173205313.png)

但是，在实际计算中arcsin的计算是非常缓慢的，因此我们不会在shader中使用反三角函数。我们使用一个更简单的公式：

$$
\text{Safe Angle} = 2 \text{ min}\{ \frac{k \cdot SDF(p)}{|p-o|},1.0 \}
$$

其中，`k` 是一个常数，用于控制阴影的软硬程度。增大 `k` 会得到更软的阴影，而减小 `k` 会得到更硬的阴影。

当然，计算真正的“Safe Angle”需要使用反三角函数（如arcsin）。为了性能考虑，在shader计算中通常会避免这样做。所以，代替直接计算“Safe Angle”，人们通常会使用一个与安全距离相关的比值来估计它，然后使用这个估计值来决定阴影的软硬程度。

#### 总结SDF

优点就是，一旦SDF完成了计算，那么渲染效率是非常高的，计算出的阴影边缘也非常平滑。

但是缺点也很显著，SDF的存储空间占用很大，复杂的场景预计算时间也很长，动态场景就需要重新计算SDF。

为什么我们要先讲SDF呢？因为SDF提供了一种高效、精确的方式来描述场景中的物体。这在计算环境光照时尤其有用，特别是当考虑到遮挡、反射或间接光照等因素时。

接下来我们进入正题，讲讲环境光照。

### 从环境光照做Shading

当使用Signed Distance Fields进行渲染时，Ray Marching可以高效地模拟环境光照，特别是环境遮挡和间接光照。通过在SDF上进行多次Ray Marching，可以模拟光在场景中的反弹，从而得到全局光照的效果。

#### 分析渲染方程

从渲染方程（没有写visibility项）的角度来看，环境光照可以被视为全局光照的一个简化。

$$
L_o\left(p, \omega_o\right)=L_e\left(p, \omega_o\right)+\int_{\Omega} L_i\left(p, \omega_i\right)\ f_r\left(p, \omega_o, \omega_i\right) \left(\omega_i \cdot n\right) d \omega_i
$$

当我们只关心环境光照时，这个方程可以大大简化。

![image-20231008203334654](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231008203334654.png)

目前的渲染方程就只有积分里面的两项了，分别是BRDF项和Lighting项。

其中BRDF项包含了两个部分，分别是diffuse和specular。

$$
f_r=k_d f_{\text {Lambert }}+f_{\text {CookTorrance }}
$$

其中，假设这里的BRDF的漫反射项使用的是 $f\_\text{Lambert}$ （即Lambertian模型），specular项使用的是Cook-Torrance模型。

* 漫反射项：

$$
f_{\text {Lambert }} = \frac{c}{\pi}
$$

* 高光项：

$$
f_{\text {CookTorrance }}=\frac{D F G}{4\left(\omega_o \cdot n\right)\left(\omega_i \cdot n\right)}
$$

其中，DGF是非常经典的三项。

首先是D项，描述的是微表面法线的分布，主流使用的是GGX模型。

$$
D(h)=\frac{\alpha^2}{\pi\left((h \cdot n)^2\left(\alpha^2-1\right)+1\right)^2}
$$

GGX模型相比于Phong模型和Beckmann模型而言，高光反射过渡更加柔和。Roughness是由 $\alpha$ 控制的。

![image-20231010150036707](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231010150036707.png)

接下来是F项，大多数都是Schlick近似模型。

$$
F(\theta)=F_0+\left(1-F_0\right)(1-\cos (\theta))^5
$$

最后是G项。可以使用Schlick的几何遮挡近似。

$$
G_{\text {Schlick }}(v, l)=\frac{\mathbf{n} \cdot \mathbf{v}}{\mathbf{n} \cdot \mathbf{v}+(1-k)(1-\mathbf{n} \cdot \mathbf{v})} \times \frac{\mathbf{n} \cdot \mathbf{l}}{\mathbf{n} \cdot \mathbf{l}+(1-k)(1-\mathbf{n} \cdot \mathbf{l})}
$$

BRDF项示意图。

![image-20231010201020547](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231010201020547.png)

目前为止，渲染方程的计算已经明确，我们可以用蒙特卡洛方法计算。

但是，蒙特卡洛方法的准确性通常与使用的样本数量成正比。一个简单的像素可能需要数百到数千次样本来减少噪声，而在实时渲染中，但凡涉及到采样的方法都会非常慢，因此我们需要避免。并且，每次光线与物体交互时，都需要计算该物体的BRDF，非常复杂。

接下来，我们需要化简这个积分，也就是对环境光照和BRDF项都做化简。

#### 化简第一步 - 环境光照项

下面这个公式想必大家都非常熟悉，用于估算两个函数的乘积在一个域上的积分。

$$
\int_{\Omega} f(x) g(x) \mathrm{d} x \approx \frac{\int_{\Omega_G} f(x) \mathrm{d} x}{\int_{\Omega_G} \mathrm{~d} x} \cdot \int_{\Omega} g(x) \mathrm{d} x
$$

当 $g(x)$ 在 $\Omega\_G$ 内近似为常数或者变化不大的时候，结果会比较准确。而此处的 $g(x)$ 正好可看作BRDF。因此，我们将渲染方程化简为下式。

$$
L_o\left(p, \omega_o\right) \approx \frac{\int_{\Omega_{f_r}} L_i\left(p, \omega_i\right) \mathrm{d} \omega_i}{\int_{\Omega_{f_r}} \mathrm{~d} \omega_i} \cdot \int_{\Omega^{+}} f_r\left(p, \omega_i, \omega_o\right) \cos \theta_i \mathrm{~d} \omega_i
$$

现在这个式子相当牛逼啊，把环境光项和BRDF分开了。注意看，拆出来的这一部分（下式），$\Omega\_G$ 表示表面粗糙度对应的微小的角度范围或方向范围。

$$
\frac{\int_{\Omega_{f_r}} L_i\left(p, \omega_i\right) \mathrm{d} \omega_i}{\int_{\Omega_{f_r}} \mathrm{~d} \omega_i}
$$

就目前而言，我们可以通过预计算得到一张完整的环境贴图（也可以实时渲染）。为了计算此处的积分，我们需要在 $\Omega\_{f\_r}$ 的区域内做积分。此处我们还可以进一步化简，即「预先对纹理进行滤波操作（模糊），我们只需要对滤波后的环境贴图采样一次光线方向就能得到积分」。

通过预计算的方式，将上面的公式化简为了一个更为简单的函数，假设滤波后的环境贴图是$L\_{i}'$：

$$
\frac{\int_{\Omega_{f_r}} L_i\left(p, \omega_i\right) \mathrm{d} \omega_i}{\int_{\Omega_{f_r}} \mathrm{~d} \omega_i} \approx L_{i}'(\omega_o)
$$

不再需要计算积分，因为这已经在预滤波过程中完成了。您只需要从已滤波的环境贴图中进行简单的采样。

![image-20231011111442116](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231011111442116.png)

对于不同的BRDF，会有不同的Lobe。因此，决定Lobe的因素有很多，包括：粗糙度、金属度、不同的BRDF模型等等。

其中，粗糙度是决定Lobe宽度的主要因素。平滑的表面（如镜子）具有非常尖锐的Lobe。粗糙的表面会使光线散射到更广的角度范围，导致更宽的Lobe。在实际开发中，我们可以直接根据粗糙度来使用不同的Lobe，像下面代码所示。而不同的Lobe就需要采用不同层级的Mipmap进行采样。

```glsl
vec3 GetEnvironmentLighting(vec3 n, vec3 v, float roughness) {
    // 根据表面的反射方向和粗糙度计算采样方向
    vec3 r = reflect(-v, n);
    // 使用粗糙度选择合适的mipmap级别
    float lodLevel = roughness * MAX_MIPMAP_LEVEL;
    // 从预滤波的环境贴图中采样
    vec3 irradiance = textureLod(envCubeMap, r, lodLevel).rgb;
    return irradiance;
}
```

当考虑环境光和BRDF的交互时，我们实际上是在对两者进行卷积。对于一个平滑的表面，其BRDF的反射Lobe非常尖锐，相对地，一个粗糙的表面有一个宽广的BRDF Lobe。对于每种粗糙度，环境光照都被预先模糊，并存储在特定的mipmap级别中。为了得到更平滑的结果，可以用三线性插值。

![image-20231011121117830](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231011121117830.png)

#### 化简第二步 - BRDF项

目前的渲染方程，已经被我们整成下面这样了，我们已经高效解决了左边这个部分，接下来就专心处理BRDF的部分即可（蓝色框框的内容）。

![image-20231011121837897](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231011121837897.png)

在分析渲染方程的时候我们详细探讨了基于微表面材质的BRDF项。

$$
f_{\text {Microfacet}}=\frac{D F G}{4\left(\omega_o \cdot n\right)\left(\omega_i \cdot n\right)}
$$

其实最重要的，

* 菲涅尔项（F）：垂直看向物体有多少能量被反射，不垂直又会反射多少能量
* 微表面的法线分布（D）：如果法线大多数都垂直于平面，那么就比较接近镜面

![image-20231011143330263](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231011143330263.png)

渲染方程中，我们要对上面这一个方程在半球面上积分，相当的麻烦。并且我们发现，这个方程接收的参数非常多，是一个五维的函数（Roughness、入射角度的极角和方位角、出射角度的极角和方位角）。因此，我们需要降维。

这里使用Schlick近似，代入到渲染方程中。

$$
\int_{\Omega^{+}} f_r\left(p, \omega_i, \omega_o\right) \cos \theta_i \mathrm{~d} \omega_i \approx \\ F_0 \int_{\Omega^{+}} \frac{f_r}{F}\left(1-\left(1-\cos \theta_i\right)^5\right) \cos \theta_i \mathrm{~d} \omega_i+ \int_{\Omega^{+}} \frac{f_r}{F}\left(1-\cos \theta_i\right)^5 \cos \theta_i \mathrm{~d} \omega_i
$$

> Schlick近似：
>
> $$
> F(\theta)=F_0+\left(1-F_0\right)(1-\cos (\theta))^5
> $$

那么至此，BRDF就变成了一个二维的函数了。输入参数是 **粗糙度（Roughness）** 和 **法线与半向量之间的夹角** $cos(\theta )$ 。这样就可以愉快的预计算了，最终得到specular的IBL结果。这也就是我们平时所说的二维LUT（Look-Up Table）。

![image-20231011170035738](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231011170035738.png)

最终，我们避免了采样，完成了渲染方程的计算。而这个方法叫做 「Split Sum方法」。

#### 球谐函数

很多复杂的函数可以通过一组更简单的基函数来逼近或表示。傅立叶变换就是其中典型的例子。下图是一系列基函数，上图是表示不同数量基函数逼近一个复杂的方波函数。

![image-20231012094238947](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231012094238947.png)

这种思想还运用在各种领域，其中一个就是现在要讲的球谐函数**Spherical Harmonics**。这个球谐函数是定义在球面空间上的一系列的二维的基函数的集合。

![image-20231012101145306](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231012101145306.png)

每一个球谐函数的基函数都是由「勒让德多项式」构成的。这里买弄悬虚一下，贴出公式。以下内容参考[Wiki百科](https://en.wikipedia.org/wiki/Spherical\_harmonics)。

***

球谐函数可以表示为：

$$
Y_l^m(\theta, \phi)=\sqrt{\frac{(2 l+1)(l-m) !}{4 \pi(l+m) !}} P_l^m(\cos \theta) e^{i m \phi}
$$

其中：

* $\theta$ 是从正 $z$ 轴到点的方向的夹角（即天顶角或与极轴的夹角），它的范围是 $0, \pi$ 。
* $\phi$ 是从正 $x$ 轴到点在 $x y$ 平面上的投影的方向的夹角（即方位角），它的范围是 $\[0, 2 / \pi)$。
* $P\_l^m$ 是关联的勒让德多项式。
* $e^{i m \phi}$ 是复指数函数, 为球谐函数引入方位依赖性。

球谐函数之间具有正交性质。这意味着当对单位球面上的任意两个不同的球谐函数进行积分时, 它们的积分为零：

$$
\int_0^{2 \pi} \int_0^\pi Y_l^m(\theta, \phi) Y_{l^{\prime}}^{m^{\prime}}(\theta, \phi) \sin \theta d \theta d \phi=\delta_{l l^{\prime}} \delta_{m m^{\prime}}
$$

其中 $\delta$ 是Kronecker delta函数, 只有当 $l=l^{\prime}$ 和 $m=m^{\prime}$ 时它才等于 1 , 否则 等于0。

***

接下来再介绍一个非常有用的性质，投影性质。

假设我们有一个定义在单位球面上的函数 $f(\theta, \phi)$ ，我们可以将其展开为球谐函数的线性组合：

$$
f(\theta, \phi)=\sum_{l=0}^{\infty} \sum_{m=-l}^l a_l^m Y_l^m(\theta, \phi)
$$

其中系数 $a\_l^m$ 可以通过与相应的球谐函数进行内积来得到:

$$
a_l^m=\int_0^{2 \pi} \int_0^\pi f(\theta, \phi) Y_l^m(\theta, \phi)^* \sin \theta d \theta d \phi
$$

这里的 $\*$ 表示复共轭。因为球谐函数是正交的，所以这种投影会得到与 $Y\_l^m(\theta, \phi)$ 关联的 $f(\theta, \phi)$ 的“成分”或“权重”。

上面这一部分看不懂也没关系，但是我们需要知道的是，对于一个环境光照，我们用一个球面函数来表示。而这个球面函数，可以用一系列的球谐函数逼近/近似。更高的阶数对应更高的频率，更高的频率意味着对环境光照更精确的模拟程度。

而刚刚说的投影性质，正好可以帮助我们求得给定一个球面函数（环境光照）对应阶数的所有对应球谐函数的基函数的系数。这里还有一个理解「投影」的方式，「**得到系数的过程就叫做投影**」。就像我们投影一个向量，得到对应系数那样，是一样的道理。上面提到的Kronecker delta的性质也恰恰体现了正交的性质。举个例子，当只有x轴投影到同一个x轴的时候，他才不可能是0，否则结果都是0。

![image-20231012105252162](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231012105252162.png)

接下来，当环境光照照亮一个diffuse的BRDF物体时，可以将散射BRDF视为低通滤波器。因为漫反射BRDF没有对特定方向的偏好，它不会增强或衰减任何特定方向的光照变化，这正是低通滤波器的特性。

使用低阶的球谐函数（例如只使用L=2或L=3）来近似环境光照时，我们主要捕捉到的是光照的低频部分。因此，在渲染漫反射BRDF的材质时，可以使用球谐函数。

![image-20231012170347249](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231012170347249.png)

#### 化简第三步 - 利用球谐函数化简

我们将环境光照分别乘上各个基函数然后在球面上做积分，就可以得到各个基函数对应的系数了。

计算球谐函数的系数，可以利用球谐函数的正交性，对两边进行积分。至于背后的数学原理在此处不展开，下面直接给出结论。

$$
L_{l m}=\int_{\Omega} L(\omega) Y_{l m}(\omega) d \omega
$$

这里， $\Omega$ 表示整个球面。这个积分实际上是环境光照函数和球谐基函数之间的点积。

如果想恢复原始的环境光照图，就将所有的系数乘上对应基函数然后累加即可。

$$
L(\omega) \approx \sum_{l=0}^N \sum_{m=-l}^l L_{l m} Y_{l m}(\omega)
$$

其中， $L\_{l m}$ 是球谐函数的系数， $Y\_{l m}(\omega)$ 是球谐基函数。

换句话说，在做Shading的时候，环境光照在整个半球上都有一定的分布，给定Shading Point的法线之后，将环境光照的球谐系数与Shading Point的法线方向上的球谐函数值相乘，最终就可以得到需要的Shading了。下图回顾整个思路。

![image-20231015154037061](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231015154037061.png)

#### 总结

我们已经学习了基函数的使用：

* 使用足够多的基函数可以表示任何函数。
* 通过使用较少的基函数，我们可以保持特定的频率内容（低频）。
* 通过某种方式（如何实现仍待明确），我们将积分简化为点积。

目前为止，我们仍然只是从环境光照中进行着色，而没有考虑阴影的效果。

**下一步：预计算辐射度传输（PRT）**

这种方法可以处理阴影和全局照明问题！但是这种方法的代价是什么呢？

### 有遮挡物的环境光照

这里就要考虑visibility项了。对于没有遮挡的方向，该值为1，否则为0。这样，当我们计算环境光影响时，遮挡项可以通过逐元素乘法与光照项相乘，从而考虑到阴影效果。

$$
L_o\left(p, \omega_o\right)=\int_{\Omega} L_i\left(p, \omega_i\right) \times B R D F\left(p, \omega_i, \omega_o\right) \times V\left(p, \omega_i\right) \times \cos (\theta) d \omega_i
$$

![image-20231012184954893](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231012184954893.png)

因此我们利用基函数的基本原理把一些东西先预计算出来，从而节省开销。否则，计算量是非常大的。

PRT的基础思想是把渲染方程分为两个部分，会动的和不会动的。且两个部分都是球面函数，可以用球谐函数化简。

![image-20231013154044525](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231013154044525.png)

#### Diffuse的BRDF

当BRDF是diffuse时，BRDF就是一个常数，因此我们把BRDF提到外面。

然后 $L\_i$ 写成基函数的形式，并且把 $l\_i$ 常数系数放在积分号外面，求和符号也可以近似放出来。

$$
L(\mathbf{o}) \approx \rho \sum l_i \int_{\Omega} \mathrm{B}_i(\mathbf{i}) V(\mathbf{i}) \max (0, \boldsymbol{n} \cdot \mathbf{i}) \mathrm{d} \mathbf{i}
$$

这样，就可以把积分部分预计算了，最终用的时候就是几个点积的操作。

$$
L(\mathbf{o}) \approx \rho \sum l_i \boldsymbol{T}_i
$$

但是，这是有代价的，即「**场景不能动了**」。

#### Glossy的BRDF

Glossy物体的BRDF不仅与入射角度有关而且还和观察角度有关，不同视角观察物体表面同一点会有不同的光照。

因此，右边那个积分就不能简单的化简为一个常数 $T\_i$ 了，而是一个包含观察方向 $o$ 的函数 $T\_i(o)$ 。

但是不慌，我们延续之前的思路继续SH近似它！

基本思路是将 $T\_i(o)$ 展开为球谐函数的线性组合, 即 :

$$
T_i(o) \approx \sum_{l=0}^L \sum_{m=-l}^l a_l^m Y_l^m\left(\omega_o\right)
$$

其中, 系数 $a\_l^m$ 可以通过类似之前的方法来计算 :

$$
a_l^m=\int_{\Omega} T_i\left(\omega_o\right) Y_l^m\left(\omega_o\right) d \omega_o
$$

![image-20231016142641895](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231016142641895.png)

也就是说，我们采样其中一些观察角度做预计算。

如果选取16个基函数，在Glossy中，每一个Shading Point的计算：长度16的向量 乘上 16\*16的光传输矩阵。

如果我们要求更多的基函数才能达到满意的效果，那么PRT恐怕就没办法解决了，也就是不能用SH来做基函数。

另外值得一说的是，我们可以在控制好基函数数量的情况下，将预计算的质量做得相对较高。计算多次反射、甚至于Path Tracing这种脏活累活也自然可以交给预计算处理。反正最终到实时渲染的时候也只是做基函数数量的点乘而已。

![image-20231016150153999](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231016150153999.png)

#### 另一个角度理解SH和渲染方程

上一节中，我们将渲染方程看作 Lighting 和 Light Transport 。如下图，Light Transport 中横线部分是BRDF，则绿色圈圈我们可以理解为某种类似光照的基函数。

![image-20231016145458505](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20231016145458505.png)

#### 总结

预计算辐射度传输（PRT）很牛逼，在预计算阶段，时间通常不是限制因素。因此，可以使用高质量的算法，例如路径追踪、考虑多次反射，来捕获场景的细节和复杂的光-物体相互作用。

球谐函数和基函数也是很神奇的工具。即使使用了较多的基函数，实时阶段的操作也是线性的，并且与基函数数量成正比，因此速度非常快。

局限也非常明显。PRT尤其适用于那些几何和材料属性不频繁变化的场景。而动态物体（如玩家和敌人）可能需要其他实时光照技术。

## 基于屏幕空间的全局光照(SSAO)

### 一些主流的全局光照技术

首先全局光照技术可以分为以下三类：

* 基于画面空间的（Image Space）：**Reflective Shadow Maps**
* 基于3D空间的：**Light Propagation Volumes**、**Voxel Global Illumination**
* 基于屏幕空间的（Screen Space）：**Screen Space Ambient Occlusion**、**Screen Space Directional Occlusion**

为了得到全局光照，我们就需要看渲染方程中的间接光照。

#### Reflective Shadow Maps（RSM）

RSM一般会直接用于的场景是：手电筒。

直观地说，为了得到间接光照，我们将直接光照能够照射到的各个地方（一般会选取关键的位置）视作一个新的光源，这个新的光源仅为了计算间接光照。

RSM其实是Shadow Mapping的拓展。了解RSM之前先回顾一下什么是SM。

Shadow Mapping 是用于生成物体阴影的一种方法，通过对灯光源视角的场景进行渲染并创建深度图（也称为阴影图）来实现阴影效果。

而RSM就是在SM生成的这张深度图中添加更多的信息，每个像素的法线和辐射。

也就是说，RSM由三项内容构成：深度值（**depth**）、法线（**world space normals**）和辐射（**flux**）。

每一个Shadow Map的像素，都对应到场景里面的一个小的纹素（Texel）。那么我们会注意到一个问题，就是一张Shadow Map一般是 $512_512$ 的分辨率。也就是有 $512_512$ 个次级光源，照亮一个shading point，考虑当前摄像机的观察方向，这个计算量是非常大的。

我们如何从一个特定的观察方向获取的信息中计算出从光源出发的所有其他方向的辐射度(radiances)，这是极其困难的。因此，为了化简，这里做了一个大胆的假设，将所有的的次级光源照射的Shading Point的材质当作Diffuse。

#### LPV

### References

1. [图形学渲染基础(7) 实时环境光照(Real-time Environment Mapping)](https://juejin.cn/post/7026291302547324964)
2. Games202-Lecture5-7

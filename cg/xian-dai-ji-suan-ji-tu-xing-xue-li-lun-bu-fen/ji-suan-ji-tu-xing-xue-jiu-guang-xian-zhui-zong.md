# 计算机图形学九：光线追踪

本章內容：

1. Whitted-Style光线追踪原理
2. 光线-表面交点流程
3. Möller Trumbore 算法解析
4. AABB轴对齐的边界框
5. DDA线性插值算法
6. 空间划分：八叉树、BSP树和KD树
7. 分离轴定理 Separating Axis Theorem
8. HBV加速模型

![image-20230613125912709](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/Cf79npNRJLhv5ow.png)

个人学习小记，欢迎讨论。

## 计算机图形学九：光线追踪

详细介绍了计算机图形学中光线追踪的基本原理。

### Whitted-style光线追踪

递归式（Recursive）光线追踪，也被称为Whitted-style光线追踪，由 Turner Whitted 在 1980 年提出的。Whitted是第一个将光线追踪算法应用于计算机生成的图像的人。

在Whitted-style光线追踪中，我们追踪一条从相机出发，经过像素，进入场景的光线。

![image-20230613124500534](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/YPf5X9382KqnTDC.png)

当这条光线遇到一个物体时，我们计算反射和折射光线，并将这些光线追踪到下一个交点。然后，我们继续递归追踪反射和折射的光线，直到达到设定的最大递归深度，或者直到光线与没有反射或折射属性的物体相交。

![image-20230613125312230](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/y1tA67U92SlbdeR.png) ![image-20230613124802354](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/SHloypLbMRXx3qA.png)

然后我们可以计算每个交点处的颜色，并将这些颜色从递归的最深层次反向累加回来，以获取最初射入相机的光线的颜色。这种方式可以生成非常逼真的图像，包括镜面反射、折射（例如透明物体）和硬阴影等效果。

![image-20230613125717794](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/KS8uqRVDdWsyv7h.png)

获得良好画面的同时付出的代价则是大量的计算资源，每个像素都可能需要追踪数百或数千次光线。

![image-20230613125912709](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/Cf79npNRJLhv5ow-20230627155606511.png)

此外，它无法处理某些复杂的光照情况，如散射、全局光照和柔和阴影，这需要使用更复杂的算法，如路径追踪或光线传输等。

至此，Whitted-style光线追踪概要就讲述完毕了。接下来讲解其中的技术细节，以便能够将技术落地。

技术细节主要有以下几点：

1. 光线-表面交点（Ray-Surface Intersection）
   1. 光线与三角形求交的基本步骤
   2. Möller Trumbore 算法
   3. 包围体积加速模型（Bounding Volumes）

### 光线-表面交点（Ray-Surface Intersection）

光线-表面交点（Ray-Surface Intersection）是光线追踪最基本/核心的问题之一。

**计算光线与三维对象表面的交点位置。**

不同类型的几何形状会有不同的交点计算方法。最基础的几何形状是球体和平面，它们的交点计算相对直接。然而，对于更复杂的形状，例如多边形网格、曲线和曲面等，交点计算可能更为复杂，甚至需要借助迭代或数值方法。

我们首先需要得到光线的定义。

#### 光线方程（Ray Equation）

![image-20230613142043312](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/hpCsqT8cabIUnMQ.png)

光线（Ray）被描述为一条从原点（o）开始，并沿着一个方向（d）扩展的射线。表达式如下：

$$
\mathbf{r}(t)=\mathbf{o}+t \mathbf{d} \quad 0 \leq t<\infty
$$

其中，

* $\mathbf{r}(t)$ ：是光线在时刻t的位置。它是一个三维向量，表示在3D空间中的一个点。
* $o$ ：是光线的原点，也是一个三维向量，表示光线的起始点。
* $t$：是光线方向上的距离参数。当t=0时，r(t)就是光线的原点；当t增大时，r(t)将沿着光线的方向移动。
* $d$ ：是光线的方向，通常被规范化为单位向量，表示光线的前进方向。

光线的任何一个点可以通过从原点开始，沿着光线的方向走特定的距离来获得。光线的方向向量乘以t，得到的是从光线原点出发到达该点的向量。把这个向量加到原点上，就得到了那个点的位置。

#### 光线与球体求交（Ray Intersection With Sphere）

对于光线与球体的交点求解，我们需要满足光线方程和球体方程。

![image-20230613142027371](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/esOPNLo8nauhmSv.png)

假如目前有一个光线的参数方程：

$$
\mathbf{r}(t)=\mathbf{o}+t \mathbf{d}
$$

和一个球体方程：

$$
(p-c)^2 = r^2
$$

其中， `c` 是球心，`r` 是球半径，`p` 是球面上的任一点。

当光线与球相交时，光线上的点满足球体方程，因此，有

$$
(\mathbf{r}(t) - c)^2 = r^2
$$

简单化简得到：

$$
a*t^2 + b*t + c = 0
$$

其中：

* $a = d \cdot d$（即方向向量的点积），
* $b = 2d \cdot (o-c)$（两倍的方向向量与从球心到原点向量的点积），
* $c = (o-c) \cdot (o-c) - r^2$（原点到球心向量的点积减去球半径的平方）。

对上面的二次方程求解得到t：

$$
t=\frac{-b \pm \sqrt{b^2-4 a c}}{2 a}
$$

整理最后得到：

$$
t=\frac{-\mathbf{d} \cdot(\mathbf{e}-\mathbf{c}) \pm \sqrt{(\mathbf{d} \cdot(\mathbf{e}-\mathbf{c}))^2-(\mathbf{d} \cdot \mathbf{d})\left((\mathbf{e}-\mathbf{c}) \cdot(\mathbf{e}-\mathbf{c})-r^2\right)}}{(\mathbf{d} \cdot \mathbf{d})}
$$

如果方程有实根，则光线与球体有交点；如果方程无实根，则光线与球体不相交。对于两个实根，通常选择较小的那个（如果它是正数），因为这对应于沿着光线方向的第一个交点。

![image-20230613142959855](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/yGZdzu3xNPkU92E.png)

#### 光线与隐式表面求交（Ray Intersection With Implicit Surface）

![image-20230613143409798](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/SWFCdMGXwPipKZ3.png)

如上图所示，许多形状可以通过隐式方程来描述。这样的方程形式为 `f(P) = 0`，其中 `P` 是一个在空间中的点，`f` 是一个函数，它将一个点映射到一个实数。

当 `f(P) = 0` 时，点 `P` 就在这个隐式形状上。

**光线和隐式表面的交点求解**就是找到一个满足下面两个条件的参数 `t`：

1. 点 `P = o + td` 在光线上；
2. 点 `P` 同时也在隐式表面 `f(P) = 0` 上。

所以，我们需要解决的问题就转化为了找到一个 `t`，使得 `f(o + td) = 0`。

这样的问题通常需要使用数值方法（如牛顿法）进行求解，因为并非所有隐式方程都有解析解。

对于一些简单的隐式表面（例如球体和平面），我们可以直接求解，但对于更复杂的隐式表面，可能需要使用迭代方法。

另外需要注意的是：方程可能有**多个解**，这说明有**多个点**被光线从隐式表面穿过。

#### 光线与三角形网格求交（Ray Intersection With Triangle Mesh）

三角形网格是在计算机图形学中最常用的几何形状之一。

它们可以用来近似任何3D表面，无论是简单的立方体还是下面这一只复杂的小奶牛。

![image-20230613144419595](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/UnyI2zLcwbixSDv.png)

**光线与三角形求交的基本步骤**

1. Firstly，将三角形表示为三个顶点 `v0, v1, v2` 。然后，构建一个平面方程，该方程表示三角形所在的平面。这可以通过计算三角形的法向量来完成，法向量可以通过两个边的向量交叉乘积得到，然后将一个顶点和法向量用于构建平面方程。
2. 接下来，求解光线与上一步求得的平面的交点。
3. 然后，我们需要检查这个交点是否在三角形内部。这可以通过使用**重心坐标**或者半空间测试来实现。本文使用的是基于重心坐标的Möller Trumbore算法。

在实际开发中，一个像素对所有三角形都进行求交显然是极度困难的。（计算量可能超过 $2^{14}$ 的量级）

因此，各种加速数据结构被开发（如BVH、八叉树或k-d树），即只测试光线可能与之相交的三角形，而不是所有的三角形。

对于一个光线和三角形网格，可能有0个、1个或多个交点。在大多数情况下，我们只关心最近的交点，也就是对应最小正`t`值的交点。

**1. 构建平面**

假设三角形位于平面上，该平面的方程是：

$$
n \cdot (p - p_0) = 0
$$

其中 `n` 是平面法向量， `p` 是平面上任意一点， `p0` 是平面上已知的一点（例如，可以是三角形的一个顶点）。

![image-20230613150218492](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/YWEU9rI2bqlnKvz.png)

**2. 求光线与平面的交点**

假设一个光线，表示为：

$$
r(t) = o + td
$$

其中 `o` 是原点，`d` 是方向向量，`t` 是参数。

现在让 $p=r(t)$ ，然后将光线代入平面方程得到：

$$
(p-p')\cdot N= (o + t \cdot d - p') \cdot N= 0
$$

整理得到：

$$
t=\frac{\left(\mathbf{p}^{\prime}-\mathbf{o}\right) \cdot \mathbf{N}}{\mathbf{d} \cdot \mathbf{N}}
$$

如果 $d \cdot N = 0$ ，则说明光线与平面平行，无交点。

否则我们就可以求解出 `t`，进而得到交点 `p = o + td`。

![image-20230613145910594](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/59jDhQdIsyKfVk8.png)

**Möller Trumbore 算法**

Möller-Trumbore 算法是用于求解**光线与三角形交点**的快速算法，**无需预先计算**包含三角形的平面方程。

该算法的优势在于其高效的性能以及直接提供了**重心坐标**，使得求解过程更为简洁。

也就是说，可以在一次计算中**同时得到**是否存在交点以及交点的参数和重心坐标。

**算法主要思路：**

将光线使用重心坐标表达：

$$
\overrightarrow{\mathbf{O}}+t \overrightarrow{\mathbf{D}}=\left(1-b_1-b_2\right) \overrightarrow{\mathbf{P}}_0+b_1 \overrightarrow{\mathbf{P}}_1+b_2 \overrightarrow{\mathbf{P}}_2 \tag{1}
$$

其中， $P\_0 , P\_1 , P\_2$ 表示三角形的三个顶点，他的三个系数是重心坐标的权重。但是根据重心坐标的性质（三个系数总和为1），因此等号右边只有两个未知量 $b\_1 , b\_2$ ，等号左边还有一个未知量 $t$​ 。将上面的\*\*公式(1)\*\*简单化简一下：

$$
o=\left(P_1-P_0\right) b_1+\left(P_2-P_0\right) b_2-t \vec{D}+P_0 \tag{2}
$$

设向量 $S$ 从三角形一点指向光源点，以及计算两个边向量 $E\_1,E\_2$ ：

$$
\begin{aligned} & \overrightarrow{\mathbf{S}}=\overrightarrow{\mathbf{O}}-\overrightarrow{\mathbf{P}}_0 \\ & \overrightarrow{\mathbf{E}}_1=\overrightarrow{\mathbf{P}}_1-\overrightarrow{\mathbf{P}}_0 \\ & \overrightarrow{\mathbf{E}}_2=\overrightarrow{\mathbf{P}}_2-\overrightarrow{\mathbf{P}}_0 \\ \end{aligned} \tag{3}
$$

将**公式(3)带入公式(2)**：

$$
\overrightarrow{\mathbf{S}}= \overrightarrow{E_1} b_1+\overrightarrow{E}_2 b_2-t \overrightarrow{D} \tag{4}
$$

写成矩阵形式：

$$
\left[\begin{array}{lll} -D & E_1 & E_2 \end{array}\right]\left[\begin{array}{c} t \\ b 1 \\ b 2 \end{array}\right]=S \tag{5}
$$

根据克莱姆法则，可以算出 $t$ ：

$$
t=\frac{\operatorname{det}\left(\left[\begin{array}{lll} S & E_1 & E_2 \end{array}\right]\right)}{\operatorname{det}\left(\left[\begin{array}{lll} -D & E_1 & E_2 \end{array}\right]\right)} \tag{6}
$$

根据**三重积（scalar triple product）**，解得：

$$
t=\frac{\left(S \times E_1\right) \cdot E_2}{E_1 \cdot\left(D \times E_2\right)} \tag{7}
$$

此时，我们计算光线方向向量 $D$ 与其中某个边向量 $E\_2$ 的叉乘，也就是\*\*公式(7)\*\*中分母的一个因子：

$$
\begin{aligned} & \overrightarrow{\mathbf{S}}_1=\overrightarrow{\mathbf{D}} \times \overrightarrow{\mathbf{E}}_2 \\ \end{aligned} \tag{8}
$$

再计算 $E\_1$ 与 $S\_1$ 的点乘，如果结果接近于0，那么**说明光线与三角形平行，没有交点。** 正好这一项是\*\*公式(7)\*\*中分子的因子：

$$
\begin{aligned} & \overrightarrow{\mathbf{S}}_2=\overrightarrow{\mathbf{S}} \times \overrightarrow{\mathbf{E}}_1 \end{aligned} \tag{9}
$$

将\*\*公式(8)(9)**带入**公式(7)\*\*化简：

$$
t=\frac{S_2 \cdot E_2}{S_1 \cdot E_1} \tag{10}
$$

再使用克莱姆法则、三重积求出另外两个未知量 $b\_1 , b\_2 $ ：

$$
b_1=\frac{\operatorname{det}\left(\left[\begin{array}{lll} -D & S & E_{2} \end{array}\right]\right)}{\operatorname{det}\left(\left[\begin{array}{lll} -D & E_{1} & E_{2} \end{array}\right]\right)}=\frac{S \cdot\left(D \times E_{2}\right)}{S_1 \cdot E_{1}}=\frac{S_{1} \cdot S}{S_1 \cdot E_{1}} \tag{11}
$$

$$
b_2=\frac{\operatorname{det}\left(\left[\begin{array}{lll} -D & E_{1} & S \end{array}\right]\right)}{\operatorname{det}\left(\left[\begin{array}{lll} -D & E_{1} & E_{2} \end{array}\right]\right)}=\frac{D \cdot\left(S \times E_{1}\right)}{S_1 \cdot E_{1}}=\frac{S_{2} \cdot D}{S_1 \cdot E_{1}} \tag{12}
$$

由\*\*公式(9)(10)(11)\*\*整理可得：

$$
\left[\begin{array}{c}t \\b_1 \\b_2\end{array}\right] = \frac{1}{\overrightarrow{\mathbf{S}}_1 \cdot \overrightarrow{\mathbf{E_1}}}\left[\begin{array}{c} \overrightarrow{\mathbf{S_2}} \cdot \overrightarrow{\mathbf{E_2}} \\ \overrightarrow{\mathbf{S_1}} \cdot \overrightarrow{\mathbf{S}} \\ \overrightarrow{\mathbf{S_2}} \cdot \overrightarrow{\mathbf{D}} \end{array}\right] \tag{13}
$$

> The [scalar triple product](http://en.wikipedia.org/wiki/Triple\_product#Scalar\_or\_pseudoscalar) is defined as the dot product of one of the vectors with the cross product of the other two:
>
> $$
> a \cdot (b×c)
> $$
>
> The scalar triple product can also be understood as the determinant of the |3|x|3| matrix having the three vectors either as its rows or its columns:
>
> $$
> \mathbf{a} \cdot(\mathbf{b} \times \mathbf{c})=\operatorname{det}\left[\begin{array}{lll} a_1 & a_2 & a_3 \\ b_1 & b_2 & b_3 \\ c_1 & c_2 & c_3 \end{array}\right]
> $$

### 包围盒体积（Bounding Volumes）

在讲BVH加速模型之前，需要知道包围盒的概念。BVH (Bounding Volume Hierarchy) 是一种基于Bounding Volumes的加速模型。

包围盒类似于之前的Bounding Box，只不过从二维变到了三维。

Bounding Volumes的基本思想是，对于一组复杂的物体，先用一个简单的体积（例如，球或者盒子）将其包围起来。

![image-20230613183154881](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/kp4BJzGW8lPbO97.png)

这样，在进行物体碰撞或者光线追踪等运算时，可以先判断这个简单的体积，从而大大减少所需要的运算量。

那么怎么判断呢？

#### 光线求交（Ray-Intersection With Box）

一个盒子是由3组平行的面求交而成的，因此，可以先判断光线与每个面的交点，然后看交点是否在盒子的边界内。我们称这个边界盒为：AABB。

![image-20230613183454230](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/Yp1xHVPa2sbZUgi.png)

**AABB (Axis-Aligned Bounding Box)** 轴对齐边界盒（AABB）是一种特殊的边界盒，它的边与坐标轴平行。这样的特性使得AABB的光线求交运算更为简单。

![image-20230613183852584](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/pblg7BD6eH5kxWf.png)

也就是说，当这三段求一个交集时，得到的结果就是在盒子内的点集。

#### AABB碰撞点（Ray-AABB intersection）

设AABB的最小顶点（即各个坐标值都最小的顶点）为（xmin, ymin, zmin），最大顶点为（xmax, ymax, zmax）。光线的方程为`r(t) = o + td`，其中o为原点，d为方向向量。那么，光线与AABB的交点t可以通过下列公式求解：

```c++
tmin = max((xmin - ox) / dx, (ymin - oy) / dy, (zmin - oz) / dz)
tmax = min((xmax - ox) / dx, (ymax - oy) / dy, (zmax - oz) / dz)
```

其中，ox, oy, oz是光线原点o的坐标，dx, dy, dz是光线方向d的坐标。如果`tmin > tmax`，说明没有交点；否则，光线与AABB的交点就在`tmin`和`tmax`之间。

AABB是一种非常常用的Bounding Volume，因为它既可以有效地加速光线追踪等运算，又比边界球等其他类型的Bounding Volume更加紧凑（即更贴近包围的物体）。

总的来说，就是t\_enter < t\_exit && t\_exit >= 0。

一般而言，光线对面求交的公式在上面已经给出：

$$
t=\frac{\left(\mathbf{p}^{\prime}-\mathbf{o}\right) \cdot \mathbf{N}}{\mathbf{d} \cdot \mathbf{N}}
$$

![image-20230613145910594](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/59jDhQdIsyKfVk8-20230627155656334.png)

特别的，如果光线沿着某一个坐标轴方向平行（比如x），那么 $t$ 的求解则变得简单：

$$
t=\frac{\mathbf{p}^{\prime}{ }_x-\mathbf{o}_x}{\mathbf{d}_x}
$$

![image-20230613191117933](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/7pUKwzZrCNgMiYA-20230627155658146.png)

### AABBs加速光线追踪

#### 均匀网格空间划分 Uniform Spatial Partitions (Grids)

一种用于加速光线追踪图形算法的技术。

该方法主要思想是将三维空间划分为一系列的均匀的网格单元，然后将每个几何对象存储在其所在的一个或多个网格单元中。当需要判断一个光线与哪些对象可能相交时，只需检查该光线穿过的网格单元中的对象，而无需检查所有的对象。

以下是具体步骤：

1.  **找到边界盒（Bounding Box）**

    首先，需要找到包含场景中所有几何对象的边界盒。这个边界盒将被划分为网格。

    ![image-20230613232604680](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/YE5T4CVk7LjPHx2.png)
2.  **创建网格（Create Grid）**

    根据需要的精度，将边界盒划分为一系列的均匀的网格单元。每个网格单元都是一个小的立方体（或者在二维情况下是一个正方形）。

    ![image-20230614083814869](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/hR4tCkAyYzKI17v.png)
3.  **存储每个对象在重叠的单元格中（Store Each Object in Overlapping Cells）**

    对于每个几何对象，找出与其重叠的所有网格单元，并将这个对象存储在这些单元中。注意，一个对象可能在多个单元格中，如果它的大小超过了单元格的大小或者位置跨越了多个单元格。

![image-20230614084331878](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/jwbBfl5FPhQrZiN.png)

**射线-场景交集（Ray-Scene Intersection）**

**按照光线遍历的顺序遍历网格（Step Through Grid in Ray Traversal Order）**

首先，需要找出光线与边界盒的交点。

![image-20230614085111941](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/PvgX7p6W8GM3VtO.png)

然后，可以使用一种叫做3D DDA（Digital Differential Analyzer）或者"Amanatides-Woo"算法的技术，按照光线遍历的顺序遍历网格。

> **Digital differential analyzer (graphics algorithm)**
>
> 3D DDA算法是在射线追踪中用于遍历均匀划分的网格单元的算法。
>
> 利用轴对齐的边界盒（AABB）与射线的特性的高效算法。
>
> 首先，我们需要理解3D DDA是如何在2D空间中工作的。在2D空间中，射线是一个格子挨着移动到下一个格子的。沿射线方向，x和y坐标以**不同的速度**增加，所以我们需要找出哪个坐标首先达到下一个格子的边界。通过比较移动到下一个x或y边界所需的t值（射线参数）来实现。
>
> 将此方法推广到3D空间，即3D DDA算法，需要考虑x、y、z三个方向。此算法的大致步骤如下：
>
> 1. **初始化**：计算射线起点到三个方向（x、y、z）的下一个边界的t值。设这三个值分别为txNext, tyNext, tzNext。同时，计算到各方向下一个边界所需的t值的增量，设这三个增量分别为txDelta, tyDelta, tzDelta。
> 2. **遍历**：在每一步，选择txNext, tyNext, tzNext中最小的值，假设为tMin。然后，向tMin所在的方向步进一个格子，并将tMin增加相应的增量（如果步进的是x方向，那么txNext = txNext + txDelta）。重复这个过程，直到到达边界盒的另一侧，或者找到一个交点。
>
> 3D DDA算法的优点是它不需要在每一步都检查所有三个方向，而只需要更新一个方向，这使得算法更高效。此外，3D DDA算法还可以很好地处理空格子，这在均匀网格中是很常见的情况。
>
> 参考文章：
>
> 1. https://zhuanlan.zhihu.com/p/415869768
> 2. https://en.wikipedia.org/wiki/Digital\_differential\_analyzer\_(graphics\_algorithm

我们刚刚通过DDA算法获得了所有与光线相交的网格单元。

![image-20230614085257494](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/Lp5XjNJaGZd16KA.png)

**对于每个网格单元，测试与所有存储在该单元中的对象的交点（For Each Grid Cell, Test Intersection with All Objects Stored at That Cell）**

当光线进入一个网格单元时，测试光线与该单元中所有存储的对象的交点。如果找到一个交点，那么就可以停止搜索，因为这将是光线的第一个交点（假设对象是不透明的）。如果没有找到交点，那么就继续向后搜索。

**不同的网格分辨率（Grid Resolution）**

网格的分辨率（即每个维度上的单元格数）是影响均匀空间分割（Uniform Spatial Partitioning）性能的关键因素。

因此，选择合适的网格分辨率是非常重要的，因为这直接影响了算法的效率。

**只有一个单元格（One cell）**：如果整个空间只有一个单元格，那么这就相当于没有进行空间分割，所有的物体都在同一个单元格中。这样，每次进行光线追踪或者碰撞检测时，都需要测试所有的物体，所以无法提高性能。

![image-20230614094628650](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/EGxHuv59K3RyUqV.png)

**太多的单元格（To Many Cells）**

如果单元格过多，则光线一次就会穿过太多单元格子，导致不必要的计算。此外，过多的单元格也会导致存储压力的提高。

![image-20230614130005664](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/KAVIhLqSUX3bgQB.png)

**启发式算法（Heuristic）**

只有当物体分布均匀的时候，使用USP均匀空间分割才能得到最好的效果。

而均匀分割的的数量也有讲究，经过大量的试验，统计出的结论：

简单地说，启发式算法是一种经验式的算法。

* **网格的分辨率为物体数量的三次根（cube root）**

原因是考虑到了三维空间的特性。在三维空间中，体积（Volume）与边长的立方（Length^3）成正比。

> 假设有N个物体，我们需要将整个空间划分为N个立方体，每个立方体内大致有一个物体。这样，每个立方体的体积应该是整个空间体积的1/N。如果空间是一个立方体，每条边长为L，那么整个空间的体积就是L^3。为了使得每个网格单元的体积是整个空间体积的1/N，我们需要让每个网格单元的边长是L的N^(1/3)，即L的三次根。这样，每个维度上的单元格数（即网格的分辨率）就应该是物体数量的三次根。

![image-20230614131405859](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/ZXYpzrGblF78eRP.png)

**USP总结**

均匀空间分割是一种简单而有效的加速技术，但是也有一些局限性，比如在物体的分布非常不均匀时，可能会有一些单元格非常密集，而一些单元格非常稀疏，这会影响算法的性能。也就是常说的 **“ Teapot in a stadium ” 问题**。

![image-20230614094155129](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/1CPacXnJEgDzRd7.png)

### 空间划分（Spatial Partitions）

空间划分大致有三种：Oct-Tree，BSP-Tree和**KD-Tree**。在本文中，重点介绍的是KD-Tree。

![image-20230614131849615](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/wsUcbAg36OzLJVH.png)

#### 八叉数（Oct-Tree）

**构建**八叉树的过程：每次把空间划分为八个区域，简单地说就是沿着 $x, y, z$ 轴用一个平面切开。

然后每一个区域递归继续划分。

也就是说，一个节点都有八个孩子节点。其中，每个节点对应一个立方体的空间区域，每个节点都有八个子节点，分别对应其区域的八个八分之一的子区域。

八叉树的根节点对应整个空间区域，叶节点对应的区域内部只有一个或者很少的物体。

![image-20230614132622816](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/2K9rY35EVLwu6kN.png)

在八叉树中进行**查询**的时候，可以先判断查询的对象（例如一个点或者一条光线）是否在节点的区域内，如果不在，则无需查询此节点及其子节点；如果在，且此节点是叶节点，则直接处理此节点内的物体；如果在且此节点不是叶节点，则需要进一步查询其子节点。这样，八叉树可以大大减少需要处理的物体数量，提高查询的效率。

值得注意的是，八叉树并不适用于所有场景。如果物体的分布非常不均匀，那么八叉树的构造和查询会变得复杂和低效。

#### BSP树（Binary Space Partitioning tree）

BSP树第一次被应用是在1993年的商业游戏《DOOM》上，可是随时渲染硬件的进步，基于BSP树的渲染慢慢淘汰。但即便在如今也有很高的学习价值。BSP树是一种更通用的空间分割技术，可以在**任意维度**的空间中使用。可以视作是KD-Tree的推广应用。

每个节点对应一个空间区域，和KD树一样每个节点都有**两个子节点**，分别对应其区域的两半。也就是说，BSP-Tree在3D空间下其每个节点表示一个平面，其代表的平面将当前空间划分为前向和背向两个子空间，分别对应左儿子和右儿子。

![image-20230614135506659](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/W4m21XytDorfauA.png)

想要**构建**一颗比较平衡的BSP树，需要尽可能每次划分出一个节点时，让其左子树节点数和右子树节点数相差不多。说白了，就是切一刀，左右两边的物体数量要相当。对象完全在超平面的前面，完全在超平面的后面，就可以直接将对象放到对应的子节点中。但是很多时候，切一刀会将某些物体切成两半，此时需要将对象划分为两部分，并分别放到前后子节点中。构建BSP树的关键在于如何选择超平面和如何划分对象。这两个步骤决定了BSP树的结构和性能。

详细的构建方法请查阅：

> https://en.wikipedia.org/wiki/Binary\_space\_partitioning#Generation

我们可以利用Z-Buffer技术，通过很少的代价对空间的片元进行排序，将靠近摄像机的片元分配在靠近树根的节点，将远离摄像机的片元分配在深度更深的节点中。也就是说，Z值最小的物体会被优先渲染，这也是完全符合大脑模式的。当然，著名的「画家演算法」也是类似的渲染思路，但是BSP-Tree算法的速度要快得多得多。

对于此处的快速排序，可以参考以下论文：

> Fuchs, Henry; Kedem, Zvi. M; Naylor, Bruce F. (1980). ["On Visible Surface Generation by A Priori Tree Structures"](http://www.cs.unc.edu/\~fuchs/publications/VisSurfaceGeneration80.pdf) (PDF). _SIGGRAPH '80 Proceedings of the 7th annual conference on Computer graphics and interactive techniques_. ACM. pp. 124–133. [doi](https://en.wikipedia.org/wiki/Doi\_\(identifier\)):[10.1145/965105.807481](https://doi.org/10.1145%2F965105.807481).

另外，BSP树的一个重要用途是解决隐藏表面消除（Hidden Surface Removal）问题，即确定哪些表面在给定的观察点应该被看到，哪些应该被隐藏。

#### k-d树（KD-Tree）

k-d树是一种在k维空间中使用的空间分割技术。每个节点都有两个子节点，分别对应其区域的两半。

**KD树构造**

k-d树的划分方向在每一层上都会轮换，例如，在三维空间中，第一层可能沿x轴划分，第二层沿y轴划分，第三层沿z轴划分，然后再回到x轴。

选择切分点的常见策略包括：选择中位数，或者选择一个能使左右子树大小尽可能平衡的点。

![image-20230614144348970](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/HJvE5epAMcfOu23.png)

如上图所示，非叶子节点存储划分的方式，叶子节点存储该区域的物体。

**KD树查询**

首先从父节点 `A` 出发判断，也就是最外层的AABB判断。

![image-20230614145809276](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/imMuD9QdsXfUW1V.png)

如果发现有碰撞点，那么就进一步查询。先从左节点开始查询，也就是图中的节点 `1` 。此时就需要将节点 `1` 中的所有三角形求交。求完之后，再查询节点 `B` ，循环往复递归查询。

![image-20230614145941676](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/1HWiVjGKmXQFEnv.png)

在遍历过程中，我们可以使用光线与边界框的交点来剪枝，即忽略那些光线无法到达的节点，以此达到加速的效果。

#### 分离轴定理（Separating Axis Theorem，简称SAT）

但是有一个比较困难的问题，判断AABB与三角形的位置关系。

![image-20230614152409850](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/3aUJtdRNWIM4SH1.png)

我们先把三维的问题简化为二维来研究，也就是先判断二维中两个凸多边形是否有重叠。

分离轴定理的基本思想是：如果两个凸形状没有重叠，那么我们应该能够找到一个轴，使得这两个形状在这个轴上的投影是分离的。反之，如果对于所有可能的轴，这两个形状在任何一个轴上的投影都有重叠，那么这两个形状就是重叠的。

但是方向是连续的，我们不可能把所有的轴都试一遍。我个人的理解是，只需要试每个多边形的每条边的法线方向就行了。此处参考的资料：

> https://gamedevelopment.tutsplus.com/collision-detection-using-the-separating-axis-theorem--gamedev-169t

![image-20230614152431971](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/GVhvNRsXup4WUql.png)

以下是判断两个凸多边形（在二维空间中）或凸多面体（在三维空间中）是否重叠的基本步骤：

1. **选择一组候选的分离轴。在二维空间中，这些轴通常是每个多边形的每条边的法线**（即垂直于边的线）。在三维空间中，这些轴可以是每个**多面体的每个面的法线**，以及每对边（一个来自每个多面体）的向量叉积。
2. 对于每个候选的分离轴，计算每个形状在这个轴上的投影，并检查这些投影是否重叠。
3. 如果我们**找到了一个**轴，使得两个形状在这个轴上的投影是分离的，那么我们可以断定这两个形状是分离的，即它们没有重叠。否则，如果对于所有候选的轴，这两个形状在每个轴上的投影都有重叠，那么我们可以断定这两个形状是重叠的。

注意：对于非凸形状，该定理可能会给出错误的结果。

关于分离轴定理的数学证明可以参考以下文章：

> https://zhuanlan.zhihu.com/p/395510228

虽然分离轴定理提供了一个理论上的解决方案来解决AABB与三角形的位置关系问题，但是这种方法计算性能较低，仍然需要大量的优化。在实践中我们还是将\*\*「减少一个物体（三角形）出现在多个AABB」\*\*作为目标，例如通过使用包围盒或其他方法来减少需要考虑的形状和轴的数量。

所以随着研究，人们提出了另外的解决方式，BVH加速模型。

### BVH加速模型（Bounding Volume Hierarchy）

BVH与前几种方法最显著的区别就是，**不再以空间作为划分依据，而是从对象的角度考虑，即三角形面**。

「空间划分」不会出现AABB重叠的情况，而「BVH」会出现AABB相互重叠的情况。

但是值得庆祝的是，BVH不会出现一个物体（三角形）出现在多个包围盒的情况。例如这样：

![image-20230614154340849](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/keWTmOH8YUQ6j7E.png)

又或者这样：

![image-20230614154411468](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/RUXSv1tAydMkDiN-20230627155819247.png)

#### 构建BVH

构建Bounding Volume Hierarchy（BVH）的过程通常是**递归**的，主要步骤如下：

1. \*\*找到边界框：\*\*计算场景中所有对象的边界框。这个边界框应该足够大，可以包含场景中的所有对象。
2. \*\*递归地分割对象集合：\*\*找到一个可以将边界框有效分割为两部分的平面。这个平面通常是通过一些启发式方法（Heuristic）找到的。通常有两种启发式方式分割：
   1. 把较长的轴“剪短”，使得射线更少地与不必要的边界体相交。
   2. 在中位数对象的位置分割节点，这样可以尽可能平均地将对象分配给两个子节点，从而保持树的平衡，使得遍历树的平均深度尽可能小。
3. \*\*重新计算子集的边界框：\*\*一旦把对象分割成了两个子集，你需要为每个子集计算新的边界框。
4. \*\*停止条件：\*\*如果子集中的对象数量已经足够小，或者已经达到了预定的递归深度，那么就停止递归。这个阈值通常是事先设定的。常见的启发式方法是：
   1. 当节点包含的对象数量已经足够少（例如小于5）时
   2. 递归深度大于20时
5. \*\*在叶节点存储对象：\*\*最后，每个叶节点都会存储一组对象和一个边界框，该边界框包含这些对象。

构建BVH的目标是尽可能地使得每个边界框内部的对象接近，以便在射线追踪过程中能够快速排除与远离射线的对象的交集。虽然构建最优的BVH是一个困难的问题，但是通过使用一些启发式方法，我们通常可以得到一个在实践中效果良好的BVH。

#### BVH数据结构

* **非终端节点**（Internal Nodes）存储的信息：
  * \*\*边界框（Bounding Box）：\*\*为内部节点存储一个边界框，该框包含该节点子树中所有对象的边界框。这个边界框在进行光线追踪时非常重要，因为它可以让我们迅速排除那些与当前光线无关的部分。
  * \*\*子节点（Children）：\*\*指向两个子节点的指针。这些子节点可能是其他内部节点，也可能是叶节点。
* **叶子节点**（Leaf Nodes）存储的信息：
  * \*\*边界框（Bounding Box）：\*\*叶节点的边界框包含了该节点所有的对象。同样，这个边界框在光线追踪时也是非常有用的。
  * \*\*对象列表（List of Objects）：\*\*叶节点存储一个对象列表，包含了该节点所有的几何对象。这些对象可能是三角形、球体或其他任何类型的几何形状。

#### BVH遍历

![image-20230614161506601](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/ylSP5uW6kqvI4YJ.png)

1. \*\*从根节点开始遍历：\*\*首先，我们从BVH的根节点开始，测试射线是否与根节点的边界框相交。
2. \*\*检查子节点：\*\*如果射线与根节点的边界框相交，我们就需要继续检查它的子节点。先检查离射线起点更近的子节点。
3. \*\*递归遍历：\*\*如果射线与子节点的边界框相交，我们就继续递归地遍历这个子节点。如果射线与子节点的边界框不相交，或者我们已经在其他地方找到了更近的交点，我们就可以跳过这个子节点，不需要检查它的子节点。
4. \*\*检查叶节点：\*\*如果我们遍历到一个叶节点，并且射线与叶节点的边界框相交，我们就需要检查射线是否与叶节点包含的几何对象相交。

伪代码如下：

```python
Intersect(Ray ray, BVH node) {
    if (ray misses node.bbox) return;
    if (node is a leaf node)
         test intersection with all objs;
         return closest intersection;
    hit1 = Intersect(ray, node.child1); 
    hit2 = Intersect(ray, node.child2);
    return the closer of hit1, hit2;
}
```

### 空间划分与物体划分总结

![image-20230614163510971](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/ASytOzf239eP5CF.png)

空间划分（Spatial Partitioning）和物体划分（Object Partitioning）都是用于高效地组织和查询三维数据的两种常见技术。

* \*\*空间划分（Spatial Partitioning）：\*\*将三维空间划分为不同区域，每个区域可能包含一个或多个物体，且一个物体也可能被分在两个或多个区域中。例如，KD-tree（k维树）就是一种常见的空间划分技术。KD-tree 是一种二叉树，用于将 k 维度的空间进行划分，每次划分都沿着某一个维度的中间线划分，将空间划分为两个子空间。这样做的优点是可以高效地查询特定范围内的物体。但是，如果场景中的物体分布不均匀或者动态变化，KD-tree 的性能可能会下降。
* \*\*物体划分（Object Partitioning）：\*\*是直接对物体进行划分的技术。在物体划分中，我们首先根据物体在空间中的分布建立一个层次结构，然后根据这个层次结构进行查询。例如，BVH（Bounding Volume Hierarchy）就是一种常见的物体划分技术。BVH 是一种树形结构，每个节点都包含了一些物体的边界体（Bounding Volume）。当我们查询一个物体时，可以通过检查这些边界体来快速排除那些与查询无关的物体，从而提高查询效率。BVH特别适合处理静态场景，但处理动态场景时可能需要经常重新构建。

### Reference

1. Fundamentals of Computer Graphics 4th
2. GAMES101 Lingqi Yan
3. https://sites.cs.ucsb.edu/\~lingqi/teaching/resources/GAMES101\_Lecture\_14.pdf
4. https://www.scratchapixel.com/lessons/3d-basic-rendering/ray-tracing-rendering-a-triangle/moller-trumbore-ray-triangle-intersection.html
5. https://zhuanlan.zhihu.com/p/451582864
6. https://en.wikipedia.org/wiki/Octree#Common\_uses
7. https://web.cs.wpi.edu/\~matt/courses/cs563/talks/bsp/document.html
8. https://www.cnblogs.com/KillerAery/p/10878367.html
9. https://cg2010studio.com/2011/10/20/binary-space-partitioning-tree/
10. https://en.wikipedia.org/wiki/Binary\_space\_partitioning
11. https://gamedevelopment.tutsplus.com/collision-detection-using-the-separating-axis-theorem--gamedev-169t
12. https://zhuanlan.zhihu.com/p/395510228

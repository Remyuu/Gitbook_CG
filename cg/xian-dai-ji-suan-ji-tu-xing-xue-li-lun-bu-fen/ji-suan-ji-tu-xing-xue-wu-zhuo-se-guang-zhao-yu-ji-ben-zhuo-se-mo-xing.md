# 计算机图形学五：着色（光照与基本着色模型）

本章內容：

1. Blinn-Phong 反射模型
2. 视觉观察（Perceptual Observations）
3. 着色点（shading point）
4. 漫反射（Diffuse Reflection）

### 1. Blinn-Phong 反射模型概述

Blinn-Phong 是一种经典的局部照明模型，被广泛用于计算机图形中以模拟光的反射。

Blinn-Phong反射模型将反射分为三个部分：环境光反射、漫反射和镜面反射。

* **环境光反射**：考虑到现实世界中，光是在各种物体之间反射的，我们引入环境光以模拟这种效果，使物体即使在阴影中也不会完全黑暗。
* **漫反射**：是物体表面粗糙，光照在表面产生多方向反射的效果。其强度与光源方向和表面法线方向的夹角有关。
* **镜面反射**：是指光源在物体表面产生的强烈反射，通常表现为亮点，这也是我们通常说的高光效应。其强度取决于观察者的视线方向和反射光方向的夹角。

![image-20230524211011221](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/AWSdGBTeKqpwHmU.png)

### 2. 视觉观察（Perceptual Observations）

* 高光反射（Specular highlights）：当光源直射在物体表面时，会在表面产生一个明亮的点，这就是高光反射，是镜面反射的一种体现。
* 漫反射（Diffuse reflection）：当光线照射在物体表面时，会沿着各个方向均匀地反射，造成物体颜色均匀、无亮点的现象，这就是漫反射。
* 环境光照（Ambient lighting）：环境光照是指来自物体周围所有方向的光，不直接来自光源，它提供了场景的基本光照，使场景中的物体即使在阴影区域也能看到。

### 3. 着色点（shading point）

在进一步讨论上面的三种视觉观察之前，我们先定义一些基本的量。

由于着色是逐像素计算，所以定义\*\*某个像素（也就是着色点）\*\*的相关概念就行了。

* **观察者方向（Viewer direction，v）**：这是从阴影点到相机的向量。
* **表面法线（Surface normal，n）**：这是垂直于表面的单位向量，指向表面的“外侧”。
* **光源方向（Light direction，l）**：这是从阴影点到每个光源的向量。
* **表面参数**：包括颜色（Color）、光泽度（Shininess）等。

![image-20230524212230088](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/cYvsRAIPdzWexhf.png)

请注意，虽然这种阴影过程可能使物体表面出现像阴影一样的效果，但这并不等于生成了阴影。

阴影是当一部分光线被其他物体遮挡，从而使某个区域没有直接接收到光线时产生的。

在本地阴影计算过程中，我们不考虑光线是否被其他物体遮挡，因此不会生成阴影。

要生成阴影，我们需要进行额外的计算，例如使用阴影映射（Shadow Mapping）或射线跟踪（Ray Tracing）等技术。

### 4. 漫反射（Diffuse Reflection）

当光线照射到一个物体的表面时，它可能会在各个方向均匀地反射出去，这就是漫反射。

漫反射常常用来模拟非光滑表面，例如纸、布料或者粗糙的石头等。

![image-20230524213400950](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/McFHvK38NpRYm46.png)

#### 兰伯特余弦定律（Lambert’s Cosine Law）

这个定律是描述漫反射的基础。它指出，被物体表面吸收的光线强度正比于光源方向与表面法线的夹角的余弦。

$$
cos \theta = l \cdot n
$$

这就是为什么在光源直射的地方（即光源方向与表面法线夹角为0度，余弦值为1）漫反射的强度最大，而在光源侧照的地方（即夹角接近90度，余弦值接近0）漫反射的强度最小。

![image-20230524213413517](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/64SPOKmuWc3hAr1.png)

#### 光照衰减（Light Falloff）

光源到物体表面的距离越远，物体表面接收到的光线强度就越小。

这是因为光的强度按照与距离的平方的反比进行衰减。

这个现象通常用公式 $I/r^2$ 来表示，其中 $I$ 是光源的初始强度，$r$ 是光源到物体表面的距离。

![image-20230524213947799](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/Yj1h5yRa3Mw9sNF.png)

#### 兰伯特着色（Lambertian Shading）

兰伯特余弦定律和光照衰减公式来计算物体表面的颜色。

$$
L_d = k_d(I/r^2)max(0, n·l)
$$

其中 $L\_d$ 是着色点的颜色，$k\_d$ 是物体表面的反射系数（包含定义材质的RGB颜色），$I$ 是光源的初始强度，$r$ 是光源到着色点的距离，$n$ 是着色点的表面法线，$l$ 是光源方向，$n·l$ 是$n$和$l$的点积（即$n$和$l$的夹角的余弦）。

这个公式表明，漫反射的颜色取决于光源强度、物体表面的反射系数、光源与着色点的距离以及光源方向与表面法线的夹角。

漫反射与眼睛的位置角度无关。

![image-20230524212230088](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/cYvsRAIPdzWexhf-20230719181729868.png)

### 5. 高光（Specular）

高光是由於物體表面的鏡面反射導致的，通常會在物體表面產生亮點。

![image-20230525171548697](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/O8jLuZ1vDqxrYn6.png)

引入半程向量 $h$ ，降低計算成本。镜面反射方向与观察点接近，则半程向量h与法线方向n接近。

$$
\begin{aligned} & \mathbf{h}=\operatorname{bisector}(\mathbf{v}, \mathbf{l}) \\ & \text { （半程向量） } \\ & =\frac{\mathbf{v}+\mathbf{l}}{\|\mathbf{v}+\mathbf{l}\|} \\ & \end{aligned}
$$

![image-20230525171611030](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/Pwh3Vqtip8scQMX.png)

由於 $cos\alpha$ 的容忍度太大，對導致道光太大，所以一般引入 $p$ 次冪，降低容忍度。一般使用100-200次幂。

$$
\begin{aligned} L_s & =k_s\left(I / r^2\right) \max (0, \cos \alpha)^p \\ & =k_s\left(I / r^2\right) \max (0, \mathbf{n} \cdot \mathbf{h})^p \end{aligned}
$$

值得注意的是，使用了半程向量的模型稱為Blinn-Phong反射模型，沒有使用半程向量的叫做Phong反射模型。

### 6. 環境光照（Ambient）

環境光通常被視為恆定和均勻的，不依賴於表面的方向或位置。

$$
L_a = k_a I_a
$$

其中， $L\_a$ 表示環境光反射光， $k\_a$ 是環境光係數。

### 7. Blinn-Phone Reflection Model

![image-20230525172542652](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/agQhoxeqEs9IRpB.png) \$$ \begin{aligned} L & =L\_a+L\_d+L\_s \\\ & =k\_a I\_a+k\_d\left(I / r^2\right) \max (0, \mathbf{n} \cdot \mathbf{l})+k\_s\left(I / r^2\right) \max (0, \mathbf{n} \cdot \mathbf{h})^p \end{aligned} \$$

### Reference

\[1] Fundamentals of Computer Graphics 4th

\[2] GAMES101 Lingqi Yan

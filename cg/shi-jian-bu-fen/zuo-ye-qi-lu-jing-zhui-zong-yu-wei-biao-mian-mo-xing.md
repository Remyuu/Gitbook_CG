# 作业七：路径追踪与微表面模型

remooooremoooo.com摘　要：我们在 HW.5 构建了Whitted-Style Ray Tracing算法光线追踪项目，在 HW.6 利用BVH加速结构加速了求交过程。这次，我们构建Path Tracing的光线追踪，并且利用多线程加速渲染。最后使用微表面模型为项目提供更具粗糙感的材质。第二部分主要讲了Cook-Torrance模型的基本理论与代码实现。关键词：计算机；图形学；c++；Path Tracing；Cook-Torrance模型；

本文分为两个部分：**路径追踪代码实现**和**微材质模型**。

我们在 [HW.5](https://remoooo.com/cg/858.html) 构建了Whitted-Style Ray Tracing算法光线追踪项目，在 [HW.6](https://remoooo.com/cg/869.html) 利用BVH加速结构加速了求交过程。这次，我们构建Path Tracing的光线追踪，并且利用多线程加速渲染。最后使用微表面模型为项目提供更具粗糙感的材质。

另外需要注意，本文关于微表面模型的内容主要来源于 [Ref.5](https://graphicscompendium.com/gamedev/15-pbr) ，主要讲了Cook-Torrance模型的基本理论与代码实现。

本文基本解说了框架的全部内容，如内容有误恳请指出。本项目是关于渲染一个CornellBox场景，最终的效果大致如下图所示：

![main](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230725190420203.png)

参数1:{SSP:64, res:{784, 784}, 并行: false, RussianRoulette = 0.8}, 渲染时间:{4101 seconds},

参数2:{SSP:64, res:{784, 784}, 并行: true, RussianRoulette = 0.8, cookTorrance, PDF = GGX}, 渲染时间:{3415 seconds}

作业七框架[下载地址🔗](https://remoooo.com/usr/uploads/2023/07/62061168.zip) （自建小水管下载慢请见谅）

### 项目流程 - main.cpp

按照惯例，我们从main函数开始分析。

这个项目的流程非常简单：\*\*设置好场景，然后渲染。\*\*接下来我们详细看看。

首先初始化Scene对象，并且设置场景分辨率。

```c++
// Change the definition here to change resolution
Scene scene(784, 784);
```

创建四种材质——红色、绿色、白色和灯光。这些材质使用了`DIFFUSE`类型并分别设置了不同的漫反射系数（`Kd`）。

```c++
Material* red = new Material(DIFFUSE, Vector3f(0.0f));
red->Kd = Vector3f(0.63f, 0.065f, 0.05f);
...
Material* light = new Material(DIFFUSE, (8.0f * Vector3f(0.747f+0.058f, 0.747f+0.258f, 0.747f) + 15.6f * Vector3f(0.740f+0.287f,0.740f+0.160f,0.740f) + 18.4f *Vector3f(0.737f+0.642f,0.737f+0.159f,0.737f)));
light->Kd = Vector3f(0.65f);
```

创建康奈尔场景的物体，然后添加到场景中。

```c++
MeshTriangle floor("../models/cornellbox/floor.obj", white);
...
MeshTriangle light_("../models/cornellbox/light.obj", light);

scene.Add(&floor);
...
scene.Add(&light_);
```

构建BVH，用于加速光线与场景中物体的碰撞检测。

```c++
scene.buildBVH();
```

最后，创建一个渲染器对象 `r` 渲染场景，并且记录渲染的时间。

```c++
Renderer r;

auto start = std::chrono::system_clock::now();
r.Render(scene);
auto stop = std::chrono::system_clock::now();
```

以上就是项目的大致流程。

### 物体抽象基类 - Object

Object类定义了一个物体在光线追踪算法中需要的所有基本行为。它使用了纯虚函数，表明这是一个接口，需要被具体的物体类（`MeshTriangle`、`Sphere`和`Triangle`）所继承并实现这些方法。

详细的说明请看下面的代码注释：

```c++
Object() {}
virtual ~Object() {}
virtual bool intersect(const Ray& ray) = 0;// 用于判断一条射线是否与该物体相交
virtual bool intersect(const Ray& ray, float &, uint32_t &) const = 0;// 也是用于检测射线与物体是否相交，但此函数还会返回交点的参数化表示和相交点索引
virtual Intersection getIntersection(Ray _ray) = 0;// 返回射线与该物体的交点信息
virtual void getSurfaceProperties(const Vector3f &, const Vector3f &, const uint32_t &, const Vector2f &, Vector3f &, Vector2f &) const = 0;// 该函数用于获取物体表面的属性，如表面的法线、纹理坐标等
virtual Vector3f evalDiffuseColor(const Vector2f &) const =0;// 评估物体在特定纹理坐标下的漫反射颜色
virtual Bounds3 getBounds()=0;// 返回物体的边界框
virtual float getArea()=0;// 返回物体的表面积，每一个形状的计算方法都可以不一样
virtual void Sample(Intersection &pos, float &pdf)=0;// 从物体表面采样一个点，用于光源采样。`pos` 参数是采样点的信息，`pdf` 是该点的概率密度函数值。
virtual bool hasEmit()=0;// 判断该物体是否发光，也就是是否为光源。
```

基于这个类，我们还创建了三个具体的物体类：`MeshTriangle`、`Sphere`和`Triangle`。这三个类都是物体类`Object`的子类，用于在三维空间中表示不同的几何形状。

由于我们需要光线追踪渲染画面，所以我们需要实现一个重要的操作`intersect`，用于检测一个光线是否与物体相交。

另外，每个类都有一个`Material`类型的数据成员`m`，表示物体的材料。材料定义了物体的颜色、纹理、发射光线等属性。

***

在Triangle.hpp中， `rayTriangleIntersect` 使用的是Möller-Trumbore算法，用于确定射线是否与三维空间中的三角形相交，如果相交，它还可以计算出交点的精确位置。详细代码解释请查看我另一篇[文章🔗](https://remoooo.com/cg/9-1.html#ci\_title10)，主要的步骤写在代码注释中了。

```c++
bool rayTriangleIntersect(const Vector3f& v0, const Vector3f& v1,
                          const Vector3f& v2, const Vector3f& orig,
                          const Vector3f& dir, float& tnear, float& u, float& v){
    // 首先，计算出三角形两边的向量(edge1和edge2)，然后根据射线方向dir和边edge2的向量积（外积）来计算一个新的向量pvec。
    Vector3f edge1 = v1 - v0;
    Vector3f edge2 = v2 - v0;
    Vector3f pvec = crossProduct(dir, edge2);
    float det = dotProduct(edge1, pvec);
    if (det == 0 || det < 0)
        return false;
    // 然后，通过计算pvec与边edge1的点积（内积），得到一个determinant（行列式）值。如果这个值为0或负数，说明射线与三角形平行或射线在三角形的反向，此时应返回false。
    Vector3f tvec = orig - v0;
    u = dotProduct(tvec, pvec);
    if (u < 0 || u > det)
        return false;
    // 之后，计算tvec和edge1的向量积得到qvec，并计算其与dir的点积得到v。如果v小于0或者u+v大于det，返回false。
    Vector3f qvec = crossProduct(tvec, edge1);
    v = dotProduct(dir, qvec);
    if (v < 0 || u + v > det)
        return false;

    float invDet = 1 / det;
    // 最后，如果通过了所有的测试，说明射线与三角形有交点。计算交点的深度tnear，以及在三角形内部的barycentric坐标(u, v)。
    tnear = dotProduct(edge2, qvec) * invDet;
    u *= invDet;
    v *= invDet;

    return true;
}
```

由于篇幅的原因，这三个类我们就只挑一些重点讲解。

#### Triangle

接下来，一个 `Triangle` 对象表示一个三维空间中的三角形。

**构造函数**

```c++
Triangle(Vector3f _v0, Vector3f _v1, Vector3f _v2, Material* _m = nullptr)
    : v0(_v0), v1(_v1), v2(_v2), m(_m)
{
    e1 = v1 - v0;
    e2 = v2 - v0;
    normal = normalize(crossProduct(e1, e2));
    area = crossProduct(e1, e2).norm()*0.5f;
}
```

每个三角形都有三个顶点（v0、v1和v2），两个边向量（e1和e2），一个法线向量（normal），一个面积（area），以及一个材质指针（m）。在三角形的构造函数中，根据输入的三个顶点，计算了边向量，法线向量，以及面积。其中，面积（area）的计算方法是`e1`和`e2`的叉积的模长的一半。

**三角形相关操作**

这里有三个函数（`Sample`、`getArea`和`hasEmit`）被直接重写了。

```c++
...
void Sample(Intersection &pos, float &pdf){
    float x = std::sqrt(get_random_float()), y = get_random_float();
    pos.coords = v0 * (1.0f - x) + v1 * (x * (1.0f - y)) + v2 * (x * y);
    pos.normal = this->normal;
    pdf = 1.0f / area;
}
float getArea(){
    return area;
}
bool hasEmit(){
    return m->hasEmission();
}
```

1. `Sample`函数在三角形的表面上随机采样一个点，然后返回：
   1. 这个点的信息（包括位置和法线向量）
   2. 采样点的概率密度函数值（pdf）
2. `getArea`函数返回三角形的面积。
3. `hasEmit`函数检查三角形的材质是否有发光。

#### MeshTriangle

这个`MeshTriangle`类也是`Object`类的子类。它表示一个由许多三角形组成的3D模型或网格。也就是说， `MeshTriangle` 对象内可能包含了许多 `Triangle` 对象。举个例子说，一个立方体模型，你可以使用12个三角形（每个面2个三角形，共6个面）来表示。

`MeshTriangle`类还包括一些额外的功能，如计算模型的AABB边界框、 `BVHAccel`对象等。

**构造函数**

以下伪代码简洁地描述了MeshTriangle的构造流程：

```c++
MeshTriangle(string filename, Material* mt) {
    // 1. 加载模型文件
    loader.LoadFile(filename);

    // 2. 为每个面创建一个Triangle对象并存储
    for (每个面 in 模型) {
        Triangle tri = 创建三角形(面的顶点, mt);
        triangles.push_back(tri);
    }

    // 3. 计算模型的包围盒和总面积
    bounding_box = 计算包围盒(模型的所有顶点);
    area = 计算总面积(所有的三角形);

    // 4. 创建一个用于快速交集测试的BVH
    bvh = 创建BVH(所有的三角形);
}
```

构造函数接受一个文件名`filename`和一个材质`mt`，然后使用`objl::Loader`来加载3D模型。在加载模型之后，它遍历模型的所有三角形，并创建对应的`Triangle`对象。与此同时，计算并存储了整个模型的边界框，以及所有三角形的总面积。

在框架中，我们使用了`objl::Loader`类读取.obj文件。调用`loader.LoadFile(filename)`完成加载。然后访问`loader.LoadedMeshes`获取加载的3D模型数据。

```c++
objl::Loader loader;
loader.LoadFile(filename);
...
auto mesh = loader.LoadedMeshes[0];
```

加载时我们注意到一句断言，这是检查一个模型是否只有唯一的网格，如果有多个网格或没有网格，则触发断言错误。

```c++
assert(loader.LoadedMeshes.size() == 1);
```

初始化模型顶点的最小和最大值，用于计算3D模型的轴对齐包围盒。

```c++
Vector3f min_vert = Vector3f{std::numeric_limits<float>::infinity(),
                             std::numeric_limits<float>::infinity(),
                             std::numeric_limits<float>::infinity()};
Vector3f max_vert = Vector3f{-std::numeric_limits<float>::infinity(),
                             -std::numeric_limits<float>::infinity(),
                             -std::numeric_limits<float>::infinity()};
```

接下来我们需要了解objl库中mesh的数据结构，也就是objl::Loader loader里面存储了什么。

* `MeshName`: 储存了网格（mesh）的名字
* `Vertices`: 存储模型中所有的顶点数据，包括位置，法线和纹理坐标。
* `Indices`: 存储模型中所有的面（通常为三角形）数据，每个面由一组指向`Vertices`中顶点的索引构成。
* `MeshMaterial`: 存储模型中的所有材质数据，包括漫反射颜色、镜面高光颜色、纹理等属性。

每个三角形的顶点信息会连续的存储在`Vertices`里，所以我们每三个顶点作为一组构建`Triangle`。然后设置AABB。

```c++
for (int i = 0; i < mesh.Vertices.size(); i += 3) {
    std::array<Vector3f, 3> face_vertices;

    for (int j = 0; j < 3; j++) {
        auto vert = Vector3f(mesh.Vertices[i + j].Position.X,
                             mesh.Vertices[i + j].Position.Y,
                             mesh.Vertices[i + j].Position.Z);
        face_vertices[j] = vert;

        min_vert = Vector3f(std::min(min_vert.x, vert.x),
                            std::min(min_vert.y, vert.y),
                            std::min(min_vert.z, vert.z));
        max_vert = Vector3f(std::max(max_vert.x, vert.x),
                            std::max(max_vert.y, vert.y),
                            std::max(max_vert.z, vert.z));
    }
    triangles.emplace_back(face_vertices[0], face_vertices[1],
                           face_vertices[2], mt);
}
bounding_box = Bounds3(min_vert, max_vert);
```

最后计算所有三角形的面积，并且构建BVH加速结构。在循环中，代码首先将所有三角形的指针存入`ptrs`，然后计算所有三角形的面积之和。然后将所有三角形指针都传入到BVH构造函数中。

```c++
std::vector<Object*> ptrs;
for (auto& tri : triangles){
    ptrs.push_back(&tri);
    area += tri.area;
}
bvh = new BVHAccel(ptrs);
```

**网格三角形相关操作**

**1. 面片属性的计算**

首先是**面片属性的计算** -- `getSurfaceProperties`，这里需要计算出以下几个属性：

1. **某三角形的法线向量**`N`：这个好做，直接找三角形两个边做一个叉积。
2. **纹理坐标**`st`：对三角形顶点的纹理坐标进行插值得到，uv是交点在三角形内部的barycentric坐标，下面详细说说。

```c++
void getSurfaceProperties(const Vector3f& P, const Vector3f& I,
                          const uint32_t& index, const Vector2f& uv,
                          Vector3f& N, Vector2f& st) const{
    const Vector3f& v0 = vertices[vertexIndex[index * 3]];
    const Vector3f& v1 = vertices[vertexIndex[index * 3 + 1]];
    const Vector3f& v2 = vertices[vertexIndex[index * 3 + 2]];
    Vector3f e0 = normalize(v1 - v0);
    Vector3f e1 = normalize(v2 - v1);
    N = normalize(crossProduct(e0, e1));
    const Vector2f& st0 = stCoordinates[vertexIndex[index * 3]];
    const Vector2f& st1 = stCoordinates[vertexIndex[index * 3 + 1]];
    const Vector2f& st2 = stCoordinates[vertexIndex[index * 3 + 2]];
    st = st0 * (1 - uv.x - uv.y) + st1 * uv.x + st2 * uv.y;
}
```

这个函数我们在`Triangle`也看到了，但是在`MeshTriangle`中，有所不同。

正如它们的名称所暗示，`Triangle`表示一个独立的三角形，而`MeshTriangle`表示一组相互连接的三角形，也就是一个三角形网格。`Triangle`的`getSurfaceProperties`会直接使用储存在`Triangle`类内的顶点和纹理坐标信息。而我们的`MeshTriangle`有多个三角形，于是我们通过参数`index`得知需获取的三角形。

对于纹理坐标的计算，首先我们知道UV坐标用于将2D纹理映射到3D模型上的过程中。使用uv坐标对三角形顶点的st坐标进行加权求和，以获得交点的st坐标。这被称为插值。这些坐标定义了3D模型的每个顶点在2D纹理上的对应位置。

在该函数中，

* `st0`, `st1`, 和 `st2` 是三角形顶点对应的纹理坐标。这些坐标指定了顶点在纹理贴图中的位置。
* `1 - uv.x - uv.y`，`uv.x`和`uv.y` 分别对应三角形三个顶点的权重。换句话说，如果你在三角形的一个顶点，那么该顶点的权重为1，其他顶点的权重为0。
* 另外，`stCoordinates`是事先又美术人员定义好的，程序员不需要关心。

根据Möller Trumbore算法，决定了st0对应(1 - uv.x - uv.y), st1对应uv.x, st2对应uv.y。

总结一下该函数的作用：用每个顶点的纹理坐标 `st` 乘以它对应的权重，然后把它们加起来。这是一种插值方法，可以用来找出三角形内任意点的纹理坐标。

**2. uv坐标在特定材质上的漫反射颜色**

`evalDiffuseColor`这个函数是用来计算一个给定二维纹理坐标在特定材质上的漫反射颜色的。

光线在撞击物体表面后会按照一定的规则反射，这个规则受到物体表面材质的影响。漫反射颜色就是描述这个反射效果的一种方式，它代表了物体表面对光线的反射能力。

当pattern项分别设置为0，默认和1时的效果图：

![image-20230719155043106](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230719155043106.png)

```c++
Vector3f evalDiffuseColor(const Vector2f& st) const
{
    float scale = 5;
    float pattern =
        (fmodf(st.x * scale, 1) > 0.5) ^ (fmodf(st.y * scale, 1) > 0.5);
    return lerp(Vector3f(0.815, 0.235, 0.031),
                Vector3f(0.937, 0.937, 0.231), pattern);
}
```

### 碰撞点信息类 - Intersection

#### 碰撞信息结构体

这个类用来保存光线与物体交点的信息。关于每一项的作用我写在了下面的注释中供大家查阅。

```c++
struct Intersection
{
    Intersection(){
        happened=false;
        coords=Vector3f();
        normal=Vector3f();
        distance= std::numeric_limits<double>::max();
        obj =nullptr;
        m=nullptr;
    }
// happened 表示是否真的发生了交点。
//如果光线并没有碰到任何物体，那么happened就会是false。
    bool happened;
// coords 表示交点的坐标。
//如果happened为true，那么coords就会包含光线与物体相交的准确位置。
    Vector3f coords;
// coords 表示交点的纹理坐标。
//它用于获取物体表面在交点位置的纹理信息。
    Vector3f tcoords;
// normal 表示交点处的法向量。
//法向量是垂直于物体表面的向量，用于确定物体的朝向，它在光照计算中起着关键作用。
    Vector3f normal;
// emit表示交点处的光源发射值。
//如果交点所在的物体是光源，这个向量就是非零的。
    Vector3f emit;
// 表示光线的原点到交点的距离。
    double distance;
// 指向光线所碰撞的物体。
    Object* obj;
// 指向交点处物体的材质，包含物体的颜色、光滑度、反射率等。
    Material* m;
};
```

#### 获取碰撞信息

这个函数其实是在Triangle和Sphere以及BVHAccel里面的，但是该函数离不开Intersection结构体，同时为了排版，所以干脆放在这一章了。

**Intersection in Triangle class**

首先，直接贴出源代码：

```c++
inline Intersection Triangle::getIntersection(Ray ray)
{
    Intersection inter;

    if (dotProduct(ray.direction, normal) > 0)
        return inter;
    double u, v, t_tmp = 0;
    Vector3f pvec = crossProduct(ray.direction, e2);
    double det = dotProduct(e1, pvec);
    if (fabs(det) < EPSILON)
        return inter;

    double det_inv = 1. / det;
    Vector3f tvec = ray.origin - v0;
    u = dotProduct(tvec, pvec) * det_inv;
    if (u < 0 || u > 1)
        return inter;
    Vector3f qvec = crossProduct(tvec, e1);
    v = dotProduct(ray.direction, qvec) * det_inv;
    if (v < 0 || u + v > 1)
        return inter;
    t_tmp = dotProduct(e2, qvec) * det_inv;

    if (t_tmp < 0)
    {
        return inter;
    }

    inter.distance = t_tmp;
    inter.coords = ray(t_tmp);
    inter.happened = true;
    inter.m = m;
    inter.normal = normal;
    inter.obj = this;

    return inter;
}
```

### 渲染器类 - Renderer

Renderer的运作流程非常简单：循环为屏幕的每一个像素生成图像。

以下是一些简单的说明文字：

* spp（samples per pixel）：每个像素的采样数量，表示光线追踪算法将在一个像素中投射多少条光线。
* framebuffer：一个一维数组，其大小为width\*height，它用于存储场景中每个像素的颜色。
* `scene.castRay(Ray(eye_pos, dir), 0)`函数：调用光线进行追踪算法。这个函数将射出一条光线，计算出这条光线在场景中所碰到的物体的颜色。这个颜色值然后被加到framebuffer中对应像素的颜色上。

所以渲染Renderer类的重点就是这一行`castRay`。而`castRay`在Scene类中，也是本文的重点。

### 关于intersect的一些说明

我们在项目中看到大量的intersect函数，让人眼花缭乱。比如初次看到intersect()，在三角形类中直接返回了true。但是实际上我们会疑问，难道不应该包含判断逻辑？而不是直接返回0或1。还有BVHAccel里面的Intersect()，Bounds3里面的IntersectP()，具体对象的getIntersection之间的关系等等。

关于这点，特此简单说明流程。

按照流程，首先在castRay()函数中，我们调用的是BVHAccel的Intersect()，然后BVHAccel的Intersect()会在BVH数据结构中找到叶子节点的AABB。然后会调用Bounds3的IntersectP判断结点的包围盒与光线是否相交，

* 如果不相交：返回Intersection类的默认构造（空的碰撞数据结构）；
* 如果当前节点是叶节点：直接调用对象（object）的`getIntersection`方法计算光线与物体的交点。

接下来`getIntersection`方法会返回对应于他们物体类型的Intersection数据结构，下面分别是Sphere、MeshTriangle和Triangle的`getIntersection`方法。

```c++
Intersection getIntersection(Ray ray){
    Intersection result;
    ...
    return result;
}
```

```c++
Intersection getIntersection(Ray ray){
    Intersection intersec;
    if (bvh) intersec = bvh->Intersect(ray);
    return intersec;
}
```

其中，Triangle的`getIntersection`方法标记了`override`，意思是在类定义内部的是函数声明，而在类定义外部的是函数定义。因此真正执行的部分是inline：

```c++
inline Intersection Triangle::getIntersection(Ray ray){
    ...
}
```

### 场景类 - Scene

再讲`castRay`之前，我们先简单浏览一下Scene的大致形态。

这个类包含了场景中所有的物体和灯光，还包含了一些渲染参数，如场景的宽度、高度、视场、背景色等。接下来对这个类的主要部分做一些说明：

**成员变量：**

* `width` 和 `height` 是场景的像素宽度和高度，`fov` 是摄像机的视场角`backgroundColor` 是场景的背景色。
* `objects` 和 `lights` 分别存储场景中的物体和光源。`objects` 是一个指向 `Object` 类型的指针的向量，`lights` 是一个包含 智能指针\<Light> 类型的向量。
* `bvh` 是一个指向 `BVHAccel` 的指针，用于存储场景的边界体层次（Bounding Volume Hierarchy，BVH）结构，以加速光线与物体的相交计算。
* `maxDepth` 和 `RussianRoulette` 用于控制路径追踪算法的细节（我们一会再讲）。

**成员函数：**

* `Add` 函数用于向场景中添加物体或光源。
* `HandleAreaLight` 函数用于处理面光源的光线追踪。
* `reflect` 和 `refract` 函数分别用于计算光线的反射和折射方向。`refract` 函数实现了斯涅尔定律，用于计算光线在进入不同介质时的折射方向。需要特别处理两种情况：光线从物体内部射出，和光线从物体外部射入。
* `fresnel` 函数用于计算菲涅尔方程，得到反射光和透射光的比例。也就是光线在经过两种不同介质的界面时，反射和透射的比例。
* `intersect`、`buildBVH`、`castRay`、`sampleLight` 和 `trace` 函数的功能在后文详细展开。

至此，这个 `Scene` 类就封装了一个光线追踪渲染场景的所有数据和操作。接下来详细说说这个类的一些细节。

***

在 Scene.cpp 中，首先有 `buildBVH()` 函数，该函数创建一个边界体层次结构(BVH)。

然后，`intersect(const Ray &ray) const` 函数用于查找由 `ray` 定义的光线与场景中的任何物体的交点。它通过使用先前构建的BVH来提高效率。

`sampleLight(Intersection &pos, float &pdf) const` 函数用于对场景中的发光物体的随机采样。这个函数首先计算出所有发光物体的面积总和，然后在这个面积中随机选择一个位置作为采样点。

```c++
void Scene::sampleLight(Intersection &pos, float &pdf) const{
    float emit_area_sum = 0;
    for (uint32_t k = 0; k < objects.size(); ++k) {
        if (objects[k]->hasEmit()){
            emit_area_sum += objects[k]->getArea();
        }
    }
    float p = get_random_float() * emit_area_sum;
    emit_area_sum = 0;
    for (uint32_t k = 0; k < objects.size(); ++k) {
        if (objects[k]->hasEmit()){
            emit_area_sum += objects[k]->getArea();
            if (p <= emit_area_sum){
                objects[k]->Sample(pos, pdf);
                break;
            }
        }
    }
}
```

**输入**： `Intersection` 引用 `pos`，浮点数引用 `pdf`。

这两个参数都是输出参数，也就是说，这个方法会改变它们的值。`Intersection` 类型的对象用于存储光线与物体的交点信息，`pdf` 表示选取这个交点的概率密度。

**输出**：这个方法没有返回值，它的结果通过改变 `pos` 和 `pdf` 的值返回。

**具体流程**：

1. 首先，这个方法通过第一个循环计算出所有发光物体的面积之和 `emit_area_sum`。对于每一个物体，它先调用 `hasEmit` 方法检查这个物体是否能发光，如果能发光，就调用 `getArea` 方法得到这个物体的面积，并加到 `emit_area_sum` 中。
2. 接着，它生成一个随机数 `p`，范围在 0 到 `emit_area_sum` 之间。这个随机数 `p` 用于随机选择一个发光物体。
3. 然后，它通过第二个循环来选择发光物体。在这个循环中，它先检查每一个物体是否能发光，如果能发光，就加上这个物体的面积，然后检查 `p` 是否小于或等于当前的 `emit_area_sum`。如果是，那么这个物体就是被选中的物体。然后，它会调用这个物体的 `Sample` 方法，在这个物体上随机选取一个点，更新 `pos` 和 `pdf` 的值。最后，它会跳出循环，结束这个方法。

#### 路径追踪实现 - castRay()

终于讲到castRay了，我们要使用路径追踪（Path Tracing）算法实现光线追踪函数。

```c++
// Implementation of Path Tracing
Vector3f Scene::castRay(const Ray &ray, int depth) const
{
    // TO DO Implement Path Tracing Algorithm here
}
```

接下来，介绍一下关键过程：光源采样和间接光照计算。

也就是说，我们可以将PT算法大致分为两部分：直接光照（Direct Illumination）和间接光照（Indirect Illumination）。

先给出一些基本定义：

* `p`: 碰撞点，是光线与场景中物体的交点。
* `wo`: outgoing direction，是从交点反射到相机的方向。
* `wi`: incoming direction，是光线从光源反射到交点的方向。

以下是伪代码，具体了上述过程：

```
shade(p, wo)
    // 先计算直接光源
    sampleLight(inter, pdf_light)
    Get x, ws, NN, emit from inter
    Shoot a ray from p to x
    If the ray is not blocked in the middle
        L_dir = emit * eval(wo, ws, N) * dot(ws, N) * dot(ws, NN) / |x-p|^2 / pdf_light

    // 再计算间接光源
    L_indir = 0.0
    Test Russian Roulette with probability RussianRoulette wi = sample(wo, N)
    Trace a ray r(p, wi)
    If ray r hit a non-emitting object at q
        L_indir = shade(q, wi) * eval(wo, wi, N) * dot(wi, N) / pdf(wo, wi, N) / RussianRoulette

    Return L_dir + L_indir
```

确保已经清晰地理解 Path Tracing 的实现方式，接下来开始写代码。

```c++
Vector3f Scene::castRay(const Ray &ray, int depth) const
{
    const float EPLISON = 0.0001f;

    Intersection p_inter = intersect(ray);
    if (!p_inter.happened)
        return Vector3f();// 默认构造零向量-黑色

    if (p_inter.m->hasEmission())// 是否是自发光
        return p_inter.m->getEmission();// 直接返回发光颜色

    // Get the intersection between ray and object plane
    Intersection x_inter;
    float pdf_light = 0.0f;
    // Sample light source at intersection point
    sampleLight(x_inter, pdf_light);

    // Get x, ws, N, NN, emit from inter
    Vector3f p = p_inter.coords;// 物体交点的坐标
    Vector3f x = x_inter.coords;// 光源交点的坐标
    Vector3f ws_dir = (x - p).normalized();// 物体到光源向量
    float ws_distance = (x - p).norm();// 物体到光源距离
    Vector3f N = p_inter.normal.normalized();// 物体交点的法向
    Vector3f NN = x_inter.normal.normalized();// 光源交点的法向
    Vector3f emit = x_inter.emit;// 光源交点的颜色向量

    // Shoot a ray from p to x
    Vector3f l_dir(0.0f), l_indir(0.0f);// 详见下说明

    Ray ws_ray(p, ws_dir);// 做一条从p到光点的Ray
    Intersection ws_ray_inter = intersect(ws_ray);// 然后求交
    // If the ray is not blocked in the middle
    // 即检查从物体p到光源x的直线路径是否被其他物体阻挡
    if(ws_ray_inter.distance - ws_distance > -EPLISON) {// 详见下说明
        l_dir = emit * p_inter.m->eval(ray.direction, ws_ray.direction, N)
                * dotProduct(ws_ray.direction, N)
                * dotProduct(-ws_ray.direction, NN)
                / (ws_distance * ws_distance)
                / pdf_light;
    }

    // Test Russian Roulette with probability RussianRoulette
    if(get_random_float() <= RussianRoulette) {// 详见下说明
        Vector3f wi_dir = p_inter.m->sample(ray.direction, N).normalized();
        Ray wi_ray(p_inter.coords, wi_dir);
        // If ray r hit a non-emitting object at q
        Intersection wi_inter = intersect(wi_ray);
        // 有检测到碰撞 且 碰撞点不发光，则开始计算间接光照
        if (wi_inter.happened && (!wi_inter.m->hasEmission())) {
            // 详见下说明
            l_indir = castRay(wi_ray, depth + 1) * p_inter.m->eval(ray.direction, wi_ray.direction, N)
                      * dotProduct(wi_ray.direction, N)
                      / p_inter.m->pdf(ray.direction, wi_ray.direction, N)
                      / RussianRoulette;
        }
    }

    return l_dir + l_indir;
}
```

需要说明几点：

1. `ray` 或 `wo_ray`: 这是我们正在处理的光线，称为出射光线 (outgoing ray)，表示从某点出发向某个方向传播的光线。在`castRay()`函数中，`ray`是作为参数传递的光线。
2.  `intersect`:会调用位于bvh的重载方法，具体如下：

    ```c++
    Intersection BVHAccel::Intersect(const Ray& ray) const
    {
        Intersection isect;
        if (!root)
            return isect;
        isect = BVHAccel::getIntersection(root, ray);
        return isect;
    }
    ```
3. `p_inter`：这是**光线与物体表面**的交点信息。具体来说，`p_inter`是一个`Intersection`类型的对象。
4. `x_inter`：这是**光线与光源**的交点信息。与`p_inter`类似，`x_inter`也是一个`Intersection`类型的对象。
5. `l_dir`（direct illumination）: 这是由光源直接照射到表面的光照贡献。在路径追踪中，这通常是通过从表面点采样光源并计算直接照明来得到的。
6. `l_indir`（indirect illumination）: 这是由环境反射到表面的间接光照贡献。在路径追踪中，这通常是通过采样表面的BRDF并递归追踪反射光线来计算的。
7. `ws_ray`：这是从点`p`到点`x`的光线。在计算直接光照时，需要向光源发射一条新的光线，以检查物体是否直接可以看到光源（即中间没有其他物体阻挡）。这条光线就是`ws_ray`。

这里重点说一下直接光照的BRDF公式。此处的公式考虑了几何项和光源采样的pdf，所以公式和我[之前文章🔗](https://remoooo.com/cg/10.html#ci\_title13)中给出的公式有所不同。

直接光照的数学公式如下：

$$
L_{dir} = \frac{ L_i \cdot f_r(w_i, w_o, N) \cdot \cos(\theta) \cdot \cos(\theta') }{ r^2 \cdot p(w_i)}\\ \begin{aligned} & where,\\ & L_{dir}:直接照明下的辐射度\\ & L_i:光源发出的辐射度\\ & f_r(wi, wo, N) : BRDF\\ & cos(\theta):入射光线 wi 和表面法线 N 之间的角度的余弦值\\ & cos(\theta'):出射光线 -{ws}_{dir} 和光源的法线向量 \text{NN} 之间的角度的余弦值\\ & r^2:衰减函数\\ & p(w_i):选择光源方向的概率密度函数(PDF)\\ \end{aligned}
$$

转换为代码就是：

```c++
l_dir = emit * p_inter.m->eval(ray.direction, ws_ray.direction, N)
        * dotProduct(ws_ray.direction, N)
        * dotProduct(-ws_ray.direction, NN)
        / (ws_distance * ws_distance)
        / pdf_light;
```

其中，eval函数如下，它描述了光照强度如何随着入射光和出射光方向的改变而变化。这个函数的输入是光线的入射方向 `ray.direction`，反射方向 `ws_ray.direction`，和表面的法线 `N`，输出是一个衡量反射光照强度的值。在代码中，漫反射BRDF = ρ/π。

```c++
...
float cosalpha = dotProduct(N, wo);
if (cosalpha > 0.0f) {
    Vector3f diffuse = Kd / M_PI;
    return diffuse;
}
else
    return Vector3f(0.0f);
...
```

计算完直接光照（direct illumination）之后，我们开始利用俄罗斯轮盘赌计算间接光照（indirect illumination）。流程大致如下：

在Scene.hpp中我们定义了`RussianRoulette`的数值，只有当取得的随机数小于`RussianRoulette`，我们才会计算间接光照。也就是说，有可能某个像素一次间接光照都没有被计算到。

假如我们现在需要计算一次间接光照，我们就为其生成一个新的光线`wi_ray`，这个光线的方向是基于当前交点的表面材质和原始光线方向进行采样得到的。这部分实现了基于材质的重要性采样。在代码中，我们是在半球上做了**均匀采样**。具体的计算方法看下面代码：

```c++
...
// uniform sample on the hemisphere
float x_1 = get_random_float(), x_2 = get_random_float();
float z = std::fabs(1.0f - 2.0f * x_1);
// r - 半球上点到原点的距离; phi - 极坐标系下的角度
float r = std::sqrt(1.0f - z * z), phi = 2 * M_PI * x_2;
Vector3f localRay(r*std::cos(phi), r*std::sin(phi), z);
return toWorld(localRay, N);
...
```

最后开始说间接光照`l_indir`的计算。间接光照的计算公式：

$$
L_o = shade(q, -w_i) \cdot f_r \cdot cosine \cdot \frac{1}{pdf(w_i)}
$$

但是这里

```c++
l_indir = castRay(wi_ray, depth + 1) * p_inter.m->eval(ray.direction, wi_ray.direction, N)
          * dotProduct(wi_ray.direction, N)
          / p_inter.m->pdf(ray.direction, wi_ray.direction, N)
          / RussianRoulette;
```

`castRay(wi_ray, depth + 1)`：这是一个递归调用，代表的是从当前交点发射新的光线，并获取该光线在所有物体上的反射后产生的光照贡献。

* 最后为什么要除以`RussianRoulette`？

因为在使用Russian Roulette的时候，我们随机生成一个数，如果这个数大于一个阈值（这里是`RussianRoulette`变量），我们就终止光线追踪。当我们终止某条光线路径的追踪时，我们实际上是在放弃了所有这条光线路径可能的后续反射，这些放弃的反射可能会对最终的光线信息有所贡献。

因此，为了补偿终止追踪的影响，我们把保留下来的光线强度进行放大，放大的倍数就是`1 / RussianRoulette`。这样可以确保所有保留的光线路径强度的期望值等于它们实际的强度，从而保证了光线追踪算法的无偏性。关于这一点我在我[上一篇文章🔗](https://remoooo.com/cg/10.html#ci\_title36)中也有提及。

至此，我们就基本把整个项目解释完毕了。除了BVH的构建，那一个部分在我[之前的文章🔗](https://remoooo.com/cg/869.html#ci\_title3)有涉及。

最终，我们实现了如下渲染：

![SSP: 64](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230717204216578.png)

***

接下来我再来讲讲如何用多线程计算加速渲染。

### 多线程加速

#### c++多线程快速介绍

对于不太熟悉c++多线程的读者，这里给出一个例子，方便大家快速入门。

```c++
#include <iostream>
#include <thread>

// 这是我们将在两个线程中运行的函数
void printMessage(std::string message) {
    std::cout << message << std::endl;
}

int main() {
    // 创建并运行两个线程
    std::thread thread1(printMessage, "Hello from thread 1");
    std::thread thread2(printMessage, "Hello from thread 2");

    // 等待两个线程都结束
    thread1.join();
    thread2.join();

    return 0;
}
```

在我们的主函数结束前，我们需要等待所有的线程都结束，所以我们必须调用`join()`函数。如果线程没有结束，但是主程序提前结束了，这可能会导致意外。

#### 部署多线程

一个比较简单的方法是使用OpenMP，这里不再赘述。

另一种就是最直接的方法，手动分块+mutex，代码如下：

```c++
// change the spp value to change sample ammount
int spp = 4;
int thread_num = 16;
int thread_height = scene.height / thread_num;
std::vector<std::thread> threads(thread_num);
std::cout << "SPP: " << spp << "\n";

std::mutex mtx;
float process=0;
float Reciprocal_Scene_height=1.f/ (float)scene.height;
auto castRay = [&](int thread_index)
{
    int height = thread_height * (thread_index + 1);
    for (uint32_t j = height - thread_height; j < height; j++)
    {
        for (uint32_t i = 0; i < scene.width; ++i) {
            // generate primary ray direction
            float x = (2 * (i + 0.5) / (float)scene.width - 1) *
                      imageAspectRatio * scale;
            float y = (1 - 2 * (j + 0.5) / (float)scene.height) * scale;
            Vector3f dir = normalize(Vector3f(-x, y, 1));
            for (int k = 0; k < spp; k++)
                framebuffer[j*scene.width+i] += scene.castRay(Ray(eye_pos, dir), 0) / spp;
        }
        mtx.lock();
        process = process + Reciprocal_Scene_height;
        UpdateProgress(process);
        mtx.unlock();
    }
};

for (int k = 0; k < thread_num; k++){
    threads[k] = std::thread(castRay,k);
    std::cout << "Thread[" << k << "] Started:" << threads[k].get_id() << "\n";
}
for (int k = 0; k < thread_num; k++){
    threads[k].join();
}
UpdateProgress(1.f);
```

但是比较出乎意料的是，在本人的macOS上，单线程与多线程（8t）的速度竟然是相同的。本人配置如下：

* **Model Name**: MacBook Pro - 14-inch, 2021
* **Chip**: Apple M1 Pro
* **Total Number of Cores**: 8 (6 performance and 2 efficiency)
* **Memory**: 16G
* **System Version**: macOS 13.2.1 (22D68)

![16.threads vs 1.thread](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230721233007828.png)

#### 优化进度显示

让每个线程跟踪它自己的进度，发现的的确确是各个线程都在同步计算对应的区域。

```c++
auto castRay = [&](int thread_index)
{
    // Create a local progress variable for this thread
    float thread_progress = 0.f;
    int height = thread_height * (thread_index + 1);
    for (uint32_t j = height - thread_height; j < height; j++)
    {
        for (uint32_t i = 0; i < scene.width; ++i) {
            // ... Rest of your code ...
        }
        // Update this thread's progress
        thread_progress += Reciprocal_Scene_height;
        std::cout << "Thread[" << thread_index << "] Progress: " << thread_progress << "\n";
    }
};
```

![image-20230724204159872](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230724204159872.png)

经过测试，在我的设备上使用2线程速度是最优的，约加速了70%。在我的任务管理器中也印证了这个说法（程序设置线程数为 2 ）：

![image-20230724204345895](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230724204345895.png)

猜测M1pro芯片的调度策略是当遇到大量并行计算时，会优先只启动两个大核心协同合作，而不像windows那样全部核心启动（一般来说）。

### 微表面模型 - Microfacet Models

* **微表面理论的提出**

早在1967年物理学家Torrance和Sparrow就已经在论文《Theory for Off-Specular Reflection From Roughened Surfaces》中提出了微表面模型（Microfacet Models）的理论。

在计算机图形学中的应用则要归功于Robert L. Cook和Kenneth E. Torrance的工作。他们在1982年的论文《A Reflectance Model for Computer Graphics》中将微表面模型引入到了计算机图形学领域，为模拟现实世界中的各种材质提供了一种物理基础的方法。

* **微表面模型是什么**

微表面模型是一种在计算机图形学中用于模拟光线从粗糙表面反射和折射的理论模型。这种模型的基本假设是，一个复杂的微观表面可以被一个简化的宏观表面替代，而宏观表面的散射函数（BSDF）则能匹配微观表面的总体方向散射行为。换句话说，微表面模型试图通过统计方法来模拟光线如何在粗糙表面上散射，而不是试图精确模拟每一个微观表面的细节。

微表面模型为BRDF提供了一个具体的、基于物理的形式，而BRDF是光照模型的核心组件。

* **为什么提出微表面模型**

用论文《A Reflectance Model for Computer Graphics》的原话来说就是：

> why images rendered with previous models often look plastic
>
> 为什么以前的渲染模型看起来有“塑料感”

也就是说，模型的提出的**其中一个**目的是为了解决“塑料感”。

#### 定义

在以前的着色模型中，我们有总反射率r的公式：

$$
\begin{aligned} & r=k_a+\sum_{i=0}^n l_c \cdot \left(r_d+r_s\right)\\ With,\\ &\begin{aligned} & r_d=k_d \cdot (\vec{n} \cdot \vec{l}) \\ & r_s=k_s \cdot (\vec{h} \cdot \vec{n})^p \end{aligned} \end{aligned}
$$

$r\_d, r\_s$分别表示漫反射率和镜面反射率。

Cook-Torrance提供了一个该方程的变体：

$$
r=k_a+\sum_{i=0}^n l_c \cdot(\vec{n} \cdot \vec{l}) \cdot \left(d \cdot r_d+s \cdot r_s\right)\\
$$

这个方程引入了两个新的概念：

1. 漫反射项$r\_d$和镜面反射项$r\_s$分别通过变量d和s控制
2. 法向量和光源向量的点积从漫反射率中分离出来，使它成为求和的一部分。这样可以使漫反射率成为一个常量，后续如果需要也可以修改这个定义。

另外，镜面反射率$r\_s$中包含一个$(n⋅l)$的除数，与求和中的$(n⋅l)$相消，这会让读者十分困惑。接下来我解释一下原因。

* 关于$s$和$d$

由于$s$和$d$是用于控制漫反射项与高光项的平衡的，所以两者之间存在以下关系：

$$
s+d=1
$$

因此我们一般会忽略$d$，化简Cook-Torrance公式：

$$
r=k_a+\sum_{i=0}^n l_c \cdot(\vec{n} \cdot \vec{l}) \cdot \left((1-s) \cdot r_d+s \cdot r_s\right)
$$

不难理解$s$与$d$之间的关系：根据能量守恒，一种材质反射的总光能（包括漫反射和镜面反射部分）不能超过它吸收的光能。因此，如果一个材质镜面反射的光能很多，那么它漫反射的光能必须较少，反之亦然。例如，一个非常光滑的表面（如镜子或金属）会有很高的镜面反射，但几乎没有漫反射。而一个非常粗糙的表面（如砖墙或布料）则会有很高的漫反射，但几乎没有镜面反射。

* 关于$r\_s$

$$
r_s=\frac{D * G * F}{4 *(\vec{n} \cdot \vec{l}) *(\vec{n} \cdot \vec{v})}
$$

需要注意的是，文献和其他地方有时会在公式的分母中使用$π$代替$4$。这是在 Cook-Torrance 的论文中出现的错误，因此在许多地方都被重复引用了。这个公式正确的推导应该是使用$4$。然而，虽然这是一个常数因子的小差别，所以如果没有完全准确地做到也并不是特别重要。

在 Cook-Torrance 的镜面反射率（r\_s）公式中，D、G、F 是三个可以选择不同形式的函数，它们分别代表着分布函数（Distribution）、几何衰减函数（Geometry）和菲涅耳反射函数（Fresnel）。

#### 微表面

微表面模型（Microfacet Model）将物体表面视为由无数微小面元组成的，每个面元都可以有自己的法线。这样的模型能够更好地模拟物体表面的细节，包括粗糙和光滑等特性。

![image-20230725110927825](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230725110927825.png)

其中，表面的法线和每个微面元的法线可能并不相同。对于非常光滑的表面，比如完美的镜子，所有的微面元都面向相同的方向，也就是表面的法线 $n$。然而，对于哑光或粗糙的表面，如哑光漆或石头，微面元面向的方向是随机的。又或者，在粗糙的表面上，这些微面元可能会遮挡其他的面元，给面元投下阴影。

#### Cook-Torrance 模型

Cook-Torrance 模型试图解释上面这三种现象：

1. 微面元的分布（通过分布函数 D，Normal Distribution Function）：不同的微面元会朝向不同的方向，这取决于表面的粗糙度。这个分布描述了对于观察者而言，反射光线的微面元的比例。
2. 微面元之间的相互遮挡和阴影效应（通过几何衰减函数 G，Geometric Attenuation Function）：这个函数描述了微面元之间的相互遮挡和阴影效应。在粗糙的表面上，一些微面元可能会被其他面元遮挡，或者给其他面元投下阴影。
3. 光线和微面元的交互（通过菲涅耳反射函数 F，Fresnel Function）：这决定了光线在接触到微面元后，有多少光线会被反射，有多少会被吸收或穿透。

虽然 G 和 F 函数对于最终的渲染结果的贡献可能较小，但它们仍然是模型中重要的组成部分。在物理基础的 BRDF 模型中，分布函数 D 是一个非常关键的部分，因为它决定了光线如何从物体表面的各个部分反射回来，从而影响了物体的视觉效果。常用的分布函数包括贝克曼分布、GGX分布等，选择不同的分布函数可以模拟出不同类型材质的视觉效果。

#### Fresnel项

这一部分是Cook-Torrance模型用于解释菲涅耳效应的。

在图形学中，一般使用Schlick近似：

$$
F=F_0+\left(1-F_0\right) *(1-(\vec{v} \cdot \vec{h}))^5
$$

其中，

* $F\_0$是光线垂直入射时的反射率，通过物体的折射率 n 计算出来
* v 和 h 分别是视线方向和半程向量（入射光线和反射光线的平均方向）

这个公式假设反射率随着入射角的变化而线性变化，但是在边缘区域（入射角接近90度）的反射率增加的更快，因此加入了一个5次方的项，这样就可以更好地模拟这个现象。

最后，这个公式可以用 C++ 实现如下：

```c++
double Fresnel_Schlick(double n, const Vector3d &v, const Vector3d &h) {
    double F0 = ((n - 1) * (n - 1)) / ((n + 1) * (n + 1));
    double cosTheta = std::max(dot(v, h), 0.0);
    return F0 + (1.0 - F0) * std::pow(1.0 - cosTheta, 5);
}
```

#### 几何衰减项

在 Cook-Torrance 模型中，几何衰减通常被定义为两个衰减因子的最小值，分别对应于这两种效应。这两个衰减因子可以通过一些简单的公式计算出来，称为Torrance-Sparrow 几何遮蔽模型。

$$
G=\min \left(1, \frac{2(\vec{h} \cdot \vec{n})(\vec{n} \cdot \vec{v})}{(\vec{v} \cdot \vec{h})}, \frac{2(\vec{h} \cdot \vec{n})(\vec{n} \cdot \vec{l})}{(\vec{v} \cdot \vec{h})}\right)
$$

其中，

* `V` 是视线方向
* `H` 是半向量（光线方向和视线方向的平均）
* `N` 是表面法线
* `n_dot_v` 和 `n_dot_l` 分别是法线和视线方向、光线方向的点积。

这种模型比较适合当表面粗糙度较低（也就是表面更接近完全镜面反射）时，可以用c++如下表示：

```c++
double v_dot_h = dotProduct(V, H);
double n_dot_h = dotProduct(N, H);
double G1 = 2 * n_dot_h * n_dot_v / v_dot_h;
double G2 = 2 * n_dot_h * n_dot_l / v_dot_h;
double G = clamp(0, 1, std::min(G1, G2));// 注意，框架中重写了clamp方法，此处是没有错误的～
```

#### D函数

最后，我们来说最关键、贡献最大的一项，高光反射的微观表面项。

Blinn-Phong模型的高光反射部分就是一个可能的D函数选择，它会使得高光部分具有类似Blinn-Phong模型的特性，但这并不是唯一的选择。实际上，有很多可能的NDF，每个都有不同的表面粗糙度和反射特性。

最常用的NDF可能是GGX（Generalized Trowbridge-Reitz）模型，它能很好地处理各种从光滑到粗糙的表面。另一个常用的选择是Beckmann模型，它通常对中等到高粗糙度的表面表现得更好。

这里我们使用各向同性的Beckmann模型：

$$
D=\frac{e^{-\frac{\tan ^2 \alpha}{m^2}}}{\pi m^2 \cos ^4 \alpha}
$$

其中，

* $m$ 代表表面粗糙度的参数
* $\alpha$ 是半向量$H$和表面法线$N$之间的角度，用$acos(\text{n\_dot\_h})$来计算。

```c++
double m = (type == MaterialType::MICROFACET_DIFFUSE) ? 0.6 : 0.2;
double alpha = acos(n_dot_h);
double D = exp(-pow(tan(alpha)/m, 2)) / (M_PI*m*m*pow(cos(alpha), 4));
```

当表面变得非常粗糙时，这些微平面的朝向会变得越来越随机，导致反射光也变得更加分散。

Cook and Torrance推荐使用的就是Beckmann分布。但是现在更多的使用_GGX_ distribution，一会深入学习再说了。

#### 代码具体实现

我们需要实现的Cook-Torrance公式：

$$
r=k_a+\sum_{i=0}^n l_c \cdot(\vec{n} \cdot \vec{l}) \cdot \left((1-s) \cdot r_d+s \cdot r_s\right)
$$

方才我们已经给出了DGF三项的具体代码，我们可以直接写出Cook-Torrance公式中的$r\_s$项：

```c++
Vector3f Material::cookTorrance(const Vector3f &wi, const Vector3f &wo, const Vector3f &N) {
    auto V = wo;
    auto L = wi;
    auto H = normalize(V + L);
    auto type = m_type;

    double n_dot_v = dotProduct(N, V);
    double n_dot_l = dotProduct(N, L);
    if(!(n_dot_v > 0 && n_dot_l > 0)) return 0;

    double n_air = 1, n_diff = 1.2, n_glos = 1.2;
    double n2 = (type == MaterialType::MICROFACET_DIFFUSE) ? n_diff : n_glos;
    double r0 = (n_air-n2)/(n_air+n2); r0*=r0;
    double F = r0+(1-r0)*pow(1 - n_dot_v, 5);

    double v_dot_h = dotProduct(V, H);
    double n_dot_h = dotProduct(N, H);
    double G1 = 2 * n_dot_h * n_dot_v / v_dot_h;
    double G2 = 2 * n_dot_h * n_dot_l / v_dot_h;
    double G = clamp(0, 1, std::min(G1, G2));

    double m = (type == MaterialType::MICROFACET_DIFFUSE) ? 0.6 : 0.2;
    double alpha = acos(n_dot_h);
    double D = exp(-pow(tan(alpha)/m, 2)) / (M_PI*m*m*pow(cos(alpha), 4));

    auto ans = F * G * D / (n_dot_l * n_dot_v * 4);
    return ans;
}
```

$r\_d $直接用$\frac{1}{\pi}$，这个假设基于Lambertian反射的性质，Lambertian反射是一种理想的、完全漫反射的表面。

然后$k\_a$表示环境光（Ambient）的反射系数。环境光被用来模拟场景中的全局照明。这里直接忽略了环境光，即令$k\_a=0$。

```c++
case MICROFACET_DIFFUSE:
{
    float cosalpha = dotProduct(N, wo);
    if (cosalpha > 0.0f) {
        auto ans = Ks * cookTorrance(wi, wo, N) + Kd * eval_diffuse(wi, wo, N);
        return ans;
    }
    else
        return Vector3f(0.0f);
    break;
}
```

另外，材质的两个系数设置如下：

```c++
Material *white_m = new Material(MICROFACET_DIFFUSE, Vector3f(0.0f));
white_m->Kd = Vector3f(0.725f, 0.71f, 0.68f);
white_m->Ks = Vector3f(1,1,1) - white_m->Kd;
```

以下就是最终渲染的结果，球特别使用了微表面模型MICROFACET\_DIFFUSE，SSP=64，分辨率784, 784：

![m, SSP64](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/image-20230725181340489.png)

### Reference

1. Fundamentals of Computer Graphics 4th
2. GAMES101 Lingqi Yan
3. 《A Reflectance Model for Computer Graphics》- 357290.357293
4. https://zhuanlan.zhihu.com/p/152226698
5. https://graphicscompendium.com/gamedev/15-pbr
6. 《Theory for Off-Specualr Reflection From Roughened Surfaces》
7. https://hanecci.hatenadiary.org/entry/20130511/p1

# 作业三：手撸着色器渲染小奶牛

### 作业三：手撸着色器渲染小奶牛

![所有小奶牛](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/uBKF4fjhDNJznGC.png)

内容：

1. **插值算法，实现法向量、颜色、纹理颜色的插值。**
2. **实现投影矩阵**
3. **实现 Blinn-Phong 模型计算 Fragment Color**
4. **实现纹理映射**
5. **实现凹凸映射（TBN矩阵）**
6. **实现位移片元着色器**

### 题目

[最新修正版本作业3框架下载链接](http://www.smartchair.org/general\_file\_upload/serve/Hw3.zip?f\_key=AMIfv95i9nBv3Uf1h8xqjK4LIK8T9DcLi4x0bxruE1gmbPRy5SUCAhG-6Pvwbet2J08kVaw80yqRhF-\_AuOvGxZFA8gPTnEDk1E00C4eAu6Ujp22J2xprRkOGx4zNtZN-z8VBS-D5phswYYwTqIDpadyxfB1D6ruHzu0QUPUTuCp8YWp3bpve9pttZeM8gtZ1XTf0WDRNoaqwv1o1CjCgsDcSd4Ni5s9qEdlWd4HfJMzyZCdXteEqqox6IEm\_Kau\_iMZhedTm\_Hv99esET-XKzFcik-K7xiZU22tyYCRNNxP5cu58PHqf1VIA5\_Osok3zIIIItQNWGKL)

#### 1. 插值算法，实现法向量、颜色、纹理颜色的插值。

rasterize\_triangle(const Triangle& t) in rasterizer.cpp

```c++
//Screen space rasterization
void rst::rasterizer::rasterize_triangle(const Triangle& t, const std::array<Eigen::Vector3f, 3>& view_pos) 
{
    // TODO: From your HW3, get the triangle rasterization code.
    // TODO: Inside your rasterization loop:
    //    * v[i].w() is the vertex view space depth value z.
    //    * Z is interpolated view space depth for the current pixel
    //    * zp is depth between zNear and zFar, used for z-buffer

    // float Z = 1.0 / (alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
    // float zp = alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
    // zp *= Z;

    // TODO: Interpolate the attributes:
    // auto interpolated_color
    // auto interpolated_normal
    // auto interpolated_texcoords
    // auto interpolated_shadingcoords

    // Use: fragment_shader_payload payload( interpolated_color, interpolated_normal.normalized(), interpolated_texcoords, texture ? &*texture : nullptr);
    // Use: payload.view_pos = interpolated_shadingcoords;
    // Use: Instead of passing the triangle's color directly to the frame buffer, pass the color to the shaders first to get the final color;
    // Use: auto pixel_color = fragment_shader(payload);
}
```

#### 2. 实现投影矩阵

get\_projection\_matrix() in main.cpp

```c++
Eigen::Matrix4f get_projection_matrix(float eye_fov, float aspect_ratio, float zNear, float zFar)
{
    // TODO: Use the same projection matrix from the previous assignments
}
```

#### 3. 实现 Blinn-Phong 模型计算 Fragment Color

phong\_fragment\_shader() in main.cpp

```c++
Eigen::Vector3f phong_fragment_shader(const fragment_shader_payload& payload)
{
    Eigen::Vector3f ka = Eigen::Vector3f(0.005, 0.005, 0.005);
    Eigen::Vector3f kd = payload.color;
    Eigen::Vector3f ks = Eigen::Vector3f(0.7937, 0.7937, 0.7937);

    auto l1 = light{{20, 20, 20}, {500, 500, 500}};
    auto l2 = light{{-20, 20, 0}, {500, 500, 500}};

    std::vector<light> lights = {l1, l2};
    Eigen::Vector3f amb_light_intensity{10, 10, 10};
    Eigen::Vector3f eye_pos{0, 0, 10};

    float p = 150;

    Eigen::Vector3f color = payload.color;
    Eigen::Vector3f point = payload.view_pos;
    Eigen::Vector3f normal = payload.normal;

    Eigen::Vector3f result_color = {0, 0, 0};
    for (auto& light : lights)
    {
        // TODO: For each light source in the code, calculate what the *ambient*, *diffuse*, and *specular* 
        // components are. Then, accumulate that result on the *result_color* object.
        
    }

    return result_color * 255.f;
}
```

#### 4. 实现纹理映射

texture\_fragment\_shader() in main.cpp

在实现 Blinn-Phong 的基础上，将纹理颜色视为公式中的 `kd` ，实现 Texture Shading Fragment Shader。

```c++
Eigen::Vector3f texture_fragment_shader(const fragment_shader_payload &payload) {
    Eigen::Vector3f return_color = {0, 0, 0};
    if (payload.texture) {
        // TODO: Get the texture value at the texture coordinates of the current fragment

    }
    Eigen::Vector3f texture_color;
    texture_color << return_color.x(), return_color.y(), return_color.z();

    Eigen::Vector3f ka = Eigen::Vector3f(0.005, 0.005, 0.005);
    Eigen::Vector3f kd = texture_color / 255.f;
    Eigen::Vector3f ks = Eigen::Vector3f(0.7937, 0.7937, 0.7937);

    auto l1 = light{{20,  20,  20},
                    {500, 500, 500}};
    auto l2 = light{{-20, 20,  0},
                    {500, 500, 500}};

    std::vector<light> lights = {l1, l2};
    Eigen::Vector3f amb_light_intensity{10, 10, 10};
    Eigen::Vector3f eye_pos{0, 0, 10};

    float p = 150;

    Eigen::Vector3f color = texture_color;
    Eigen::Vector3f point = payload.view_pos;
    Eigen::Vector3f normal = payload.normal;

    Eigen::Vector3f result_color = {0, 0, 0};

    for (auto &light: lights) {
        // TODO: For each light source in the code, calculate what the *ambient*, *diffuse*, and *specular* 
        // components are. Then, accumulate that result on the *result_color* object.

    }

    return result_color * 255.f;
}
```

#### 5. 实现凹凸映射

bump\_fragment\_shader() in main.cpp

在实现 Blinn-Phong 的基础上，通过获取TBN矩阵将法线空间转换到世界坐标，同时通过参数控制凹凸量。

```c++
Eigen::Vector3f bump_fragment_shader(const fragment_shader_payload &payload) {

    Eigen::Vector3f ka = Eigen::Vector3f(0.005, 0.005, 0.005);
    Eigen::Vector3f kd = payload.color;
    Eigen::Vector3f ks = Eigen::Vector3f(0.7937, 0.7937, 0.7937);

    auto l1 = light{{20,  20,  20},
                    {500, 500, 500}};
    auto l2 = light{{-20, 20,  0},
                    {500, 500, 500}};

    std::vector<light> lights = {l1, l2};
    Eigen::Vector3f amb_light_intensity{10, 10, 10};
    Eigen::Vector3f eye_pos{0, 0, 10};

    float p = 150;

    Eigen::Vector3f color = payload.color;
    Eigen::Vector3f point = payload.view_pos;
    Eigen::Vector3f normal = payload.normal;


    float kh = 0.2, kn = 0.1;

    // TODO: Implement bump mapping here
    // Let n = normal = (x, y, z)
    // Vector t = (x*y/sqrt(x*x+z*z),sqrt(x*x+z*z),z*y/sqrt(x*x+z*z))
    // Vector b = n cross product t
    // Matrix TBN = [t b n]
    // dU = kh * kn * (h(u+1/w,v)-h(u,v))
    // dV = kh * kn * (h(u,v+1/h)-h(u,v))
    // Vector ln = (-dU, -dV, 1)
    // Normal n = normalize(TBN * ln)


    Eigen::Vector3f result_color = {0, 0, 0};
    result_color = normal;

    return result_color * 255.f;
}
```

#### 6. 实现位移片元着色器

displacement\_fragment\_shader() in main.cpp

在实现 Bump mapping 的基础上，实现 displacement mapping。

```c++
Eigen::Vector3f displacement_fragment_shader(const fragment_shader_payload &payload) {

    Eigen::Vector3f ka = Eigen::Vector3f(0.005, 0.005, 0.005);
    Eigen::Vector3f kd = payload.color;
    Eigen::Vector3f ks = Eigen::Vector3f(0.7937, 0.7937, 0.7937);

    auto l1 = light{{20,  20,  20},
                    {500, 500, 500}};
    auto l2 = light{{-20, 20,  0},
                    {500, 500, 500}};

    std::vector<light> lights = {l1, l2};
    Eigen::Vector3f amb_light_intensity{10, 10, 10};
    Eigen::Vector3f eye_pos{0, 0, 10};

    float p = 150;

    Eigen::Vector3f color = payload.color;
    Eigen::Vector3f point = payload.view_pos;
    Eigen::Vector3f normal = payload.normal;

    float kh = 0.2, kn = 0.1;

    // TODO: Implement displacement mapping here
    // Let n = normal = (x, y, z)
    // Vector t = (x*y/sqrt(x*x+z*z),sqrt(x*x+z*z),z*y/sqrt(x*x+z*z))
    // Vector b = n cross product t
    // Matrix TBN = [t b n]
    // dU = kh * kn * (h(u+1/w,v)-h(u,v))
    // dV = kh * kn * (h(u,v+1/h)-h(u,v))
    // Vector ln = (-dU, -dV, 1)
    // Position p = p + kn * n * h(u,v)
    // Normal n = normalize(TBN * ln)


    Eigen::Vector3f result_color = {0, 0, 0};

    for (auto &light: lights) {
        // TODO: For each light source in the code, calculate what the *ambient*, *diffuse*, and *specular* 
        // components are. Then, accumulate that result on the *result_color* object.


    }

    return result_color * 255.f;
}
```

### 题解解析

#### 1. 实现三角形渲染（HW2基础上增加颜色、法向量和纹理颜色坐标插值）

代码内容大致与HW2相同。

首先使用Bounding Box缩小渲染范围。

```c++
auto v = t.toVector4();

float min_x = std::floor(std::min({v[0].x(), v[1].x(), v[2].x()}));
float max_x = std::ceil(std::max({v[0].x(), v[1].x(), v[2].x()}));
float min_y = std::floor(std::min({v[0].y(), v[1].y(), v[2].y()}));
float max_y = std::ceil(std::max({v[0].y(), v[1].y(), v[2].y()}));

// Iterating through each pixel in the bounding box
for (float x = min_x; x <= max_x; x++) {
    for (float y = min_y; y <= max_y; y++) {
        ......//后文代码均在此处
    }
}
```

逐个判断像素是否在三角形内：

```c++
if (insideTriangle(x, y, t.v)){......//后文代码均在此处}
```

如果该像素在三角形内部，则开始渲染这个像素。

首先获取这个三角形的深度值 `z_interpolated` ，与深度缓冲区中的深度值 `depth_buf[get_index(x, y)]` 比较，判断当前像素是否需要被绘制。

```c++
auto [alpha, beta, gamma] = computeBarycentric2D(x, y, t.v);
float w_reciprocal = 1.0 / (alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
float z_interpolated =
        alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
z_interpolated *= w_reciprocal;
if (z_interpolated < depth_buf[get_index(x, y)]) {
	......//后文代码均在此处
}
```

如果像素需要被绘制，则需要实现法向量、颜色、纹理颜色的插值。也就是计算以下参数:

* auto interpolated\_color // 插值后的颜色
* auto interpolated\_normal // 插值后的法向量
* auto interpolated\_texcoords // 插值后的纹理坐标
* auto interpolated\_shadingcoords // 插值后的内部点位置

然后用以下代码将这些信息写入渲染器中。

* fragment\_shader\_payload payload( interpolated\_color, interpolated\_normal.normalized(), interpolated\_texcoords, texture ? &\*texture : nullptr);
* payload.view\_pos = interpolated\_shadingcoords;
* auto pixel\_color = fragment\_shader(payload);

由于我们在上面计算出了当前像素的重心坐标系数分别是： `[alpha, beta, gamma]` ，可以直接利用 `interpolate()` 函数在三个顶点之间线性插值。

比方说颜色插值：

* t.color\[0]、t.color\[1]、t.color\[2]：这些参数是三角形的顶点颜色，分别表示三个顶点的颜色值。

```c++
// 颜色插值
auto interpolated_color = interpolate(alpha, beta, gamma, t.color[0], t.color[1], t.color[2], 1);
// 法向量插值
auto interpolated_normal = interpolate(alpha, beta, gamma, t.normal[0], t.normal[1], t.normal[2], 1);
// 纹理颜色坐标插值
auto interpolated_texcoords = interpolate(alpha, beta, gamma, t.tex_coords[0], t.tex_coords[1], t.tex_coords[2], 1);
// 内部点位置插值
auto interpolated_shadingcoords = interpolate(alpha, beta, gamma, view_pos[0], view_pos[1], view_pos[2], 1);

```

接下来吧上面的差值结果传入片段着色器。

```c++
fragment_shader_payload payload(interpolated_color, interpolated_normal.normalized(), interpolated_texcoords, texture ? &*texture : nullptr);
payload.view_pos = interpolated_shadingcoords;
auto pixel_color = fragment_shader(payload);
```

接下来详细讲解上面这段代码：

1.  **第一行**

    `fragment_shader_payload payload(interpolated_color, interpolated_normal.normalized(), interpolated_texcoords, texture ? &*texture : nullptr);`

创建了名为一个 `payload` 的 `fragment_shader_payload` 对象，并将一些插值后的数据作为参数传递给它。具体参数的含义如下：

* `interpolated_color`：插值后的颜色值，表示当前像素点的颜色。
* `interpolated_normal.normalized()`：插值后的法向量，并进行了归一化处理，表示当前像素点的法向量。
* `interpolated_texcoords`：插值后的纹理坐标，用于纹理映射。
* `texture ? &*texture : nullptr`：判断是否有纹理可用，如果有，将纹理指针传递给 `payload` 对象，否则传递空指针。

2.  **第二行**

    `payload.view_pos = interpolated_shadingcoords;`

将插值后的内部点位置 `interpolated_shadingcoords` 赋值给 `payload` 对象的 `view_pos` 成员变量。这个内部点位置表示当前像素在观察坐标系中的位置，通常用于光照计算。

3.  **第三行**

    `auto pixel_color = fragment_shader(payload);`

调用片段着色器函数 `fragment_shader`，并将 `payload` 对象作为参数传递给它，以计算像素的最终颜色。计算得到的颜色将赋值给变量 `pixel_color`。

最后调用 `set_pixel` 函数，函数会自动将处理好的像素颜色传入 `frame_buf` 中。

最后更新深度缓冲区的数值。

```c++
// Set the pixel color if it should be painted
Eigen::Vector2i p = { (float)x,(float)y};
set_pixel(p, pixel_color); //设置颜色
depth_buf[get_index(x, y)] = z_interpolated;//更新z值
```

#### 2. 投影变换矩阵

HW2已经写过，这里直接抄过来。

```c++
Eigen::Matrix4f get_projection_matrix(float eye_fov, float aspect_ratio, float zNear, float zFar)
{
    // TODO: Use the same projection matrix from the previous assignments
    Eigen::Matrix4f projection = Eigen::Matrix4f::Identity();
    Eigen::Matrix4f M = Eigen::Matrix4f ::Identity();
    float fov = 0.5*eye_fov*MY_PI/180;
    float top = tan(fov) * zNear;
    float bottom = -top;
    float right = top * aspect_ratio;
    float left = -right;
    M << 2 * abs(zNear) / (right - left), 0, (right + left) / (right - left), 0,
            0, 2 * abs(zNear) / (top - bottom), (top + bottom) / (top - bottom), 0,
            0, 0, (abs(zNear) + abs(zFar)) / (abs(zNear) - abs(zFar)), 2 * abs(zFar * zNear) / (abs(zNear) - abs(zFar)),
            0, 0, -1, 0;
    return M;
}
```

**2.1 着色模型改为`normal_fragment_shader`**

在 main.cpp 的main函数中找到

```c++
std::function<Eigen::Vector3f(fragment_shader_payload)> active_shader = phong_fragment_shader;
```

将代码改为：

```c++
std::function<Eigen::Vector3f(fragment_shader_payload)> active_shader = normal_fragment_shader;
```

这里对于不熟c++的同学可能有疑惑，接下来讲讲 `std::function` 的简单用法。

> 为了简化例子，我们使用类似 `int` 或者 `std::string` 这种类型让我们的例子简单明了。
>
> 现在有一个函数，传入 `int` ，返回值类型 `std::string` 。
>
> ```c++
> std::string intToString(int x) {
> return std::to_string(x);
> }
> ```
>
> 然后创建一个 `std::function` ，可以容下所有函数尖括号中相同签名的函数（传入返回参数类型匹配）。
>
> ```c++
> std::function<std::string(int)> myFunction;
> ```
>
> 然后可以让 `intToString` 接受 `myFunction`
>
> ```c++
> myFunction = intToString;
> ```
>
> 现在呢， `myFunction` 函数和 `intToString` 就是完全一样的了。
>
> ```c++
> std::string result = myFunction(123);  
> // result now holds the string "123"
> ```

在渲染管线代码中，`std::function<Eigen::Vector3f(fragment_shader_payload)>` 是一个函数对象（function object），可以容纳所有类型匹配的函数。这是用来分配不同的片段着色器函数(如 `normal_fragment_shader` 或 `phong_fragment_shader)` 给光栅化器，允许动态改变着色方法。

接下来就可以看到效果了。

![普通小奶牛](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/7GtLzea4VDyi3ZQ.png)

#### 3. 完成Blinn-Phong反射模型

![image-20230525172542652](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/agQhoxeqEs9IRpB-20230719182302650.png) \$$ \begin{aligned} L & =L\_a+L\_d+L\_s \\\ & =k\_a I\_a+k\_d\left(I / r^2\right) \max (0, \mathbf{n} \cdot \mathbf{l})+k\_s\left(I / r^2\right) \max (0, \mathbf{n} \cdot \mathbf{h})^p \end{aligned} \$$

* 在作业框架中，已经事先生成了ambient, diffuse, and specular 反射系数（`ka, kd, ks`），需要注意的是，这个反射系数包含了环境光的；两个光源的位置和强度（`l1, l2`），环境光强度常数（ `amb_light_intensity` ）。与此同时，之前光栅化的信息保存在了 `payload` 中，包括物体材质颜色 `color` ，位置信息 `point` 和法线 `normal` 。
* 并且创建了 `result_color` 用于保存Blinn-Phong处理的结果。
* 对于每个光源，它计算到光源的方向向量( `l` )和相机的方向向量( `v` )，以及用于高光项的半向量(h)。衰减因子（ `r` ），距离该点越远，光的强度就越低。代码中直接对 `l` 自身做一次点积处理获得向量 `l` 的模的平方。
* 最后求和得到一束光的光照的结果 `(ambient + diffuse + specular)` ，然后累加到 `result_color` 中。最终得到经过Blinn-Phong Reflection Model 的光照结果。

```c++
// TODO: For each light source in the code, calculate what the *ambient*, *diffuse*, and *specular* 
// components are. Then, accumulate that result on the *result_color* object.
auto v = eye_pos - point; //v为出射光方向（指向眼睛）
auto l = light.position - point; //l为指向入射光源方向
auto h = (v + l).normalized(); //h为半程向量即v+l归一化后的单位向量
auto r = l.dot(l); //衰减因子
auto ambient = ka.cwiseProduct(amb_light_intensity);
auto diffuse = kd.cwiseProduct(light.intensity / r) * std::max(0.0f, normal.normalized().dot(l.normalized()));
auto specular = ks.cwiseProduct(light.intensity / r) * std::pow(std::max(0.0f, normal.normalized().dot(h)), p);
result_color += (ambient + diffuse + specular);
```

![Blinn-Phong小奶牛](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/C2IG3mw5rTiH1sJ.png)

#### 4. 完成纹理映射

这个部分和上part基本一致，只是改变了 漫反射系数 的 kd 数值。

我们只需要填充这个部分：

```c++
if (payload.texture) {
    // TODO: Get the texture value at the texture coordinates of the current fragment
    return_color = payload.texture->getColor(payload.tex_coords.x(), payload.tex_coords.y());
}
```

代码解释如下：

* 当payload的材质贴图为空的时候，不渲染纹理。
* 向纹理贴图类中的 `getColor()` 函数传入uv法线坐标获取当前法线坐标的颜色值。

接下来只需要将RGB写入 `kd` ，归一化就行了。

```c++
...
texture_color << return_color.x(), return_color.y(), return_color.z();
...
Eigen::Vector3f kd = texture_color / 255.f;
...
```

接下来仔main.cpp将纹理文件改为「spot\_texture.png」：

```c++
//    auto texture_path = "hmap.jpg";
auto texture_path = "spot_texture.png";
```

效果如下：

![纹理映射小奶牛](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/TCaMGpVSYHlFEbX.png)

**4.1 潜在的BUG**

如果我们检测传入 `getColor()` 函数的参数 uv，我们会发现有时候传入的参数不在0到1之间。这个时候有几率会导致程序崩溃。我们需要控制uv在0到1之间。

```c++
Eigen::Vector3f getColor(float u, float v)
{
    if (u < 0) {u = 0;}
    if (u > 1) {u = 1;}
    if (v < 0) {v = 0;}
    if (v > 1) {v = 1;}
    auto u_img = u * width;
    auto v_img = (1 - v) * height;
    auto color = image_data.at<cv::Vec3b>(v_img, u_img);
    return Eigen::Vector3f(color[0], color[1], color[2]);
}
```

至于原因目前还没深入研究 // TODO

#### 5. 实现凹凸贴图

首先将贴图改回「**hmap.jpg**」，渲染函数改为「**bump\_fragment\_shader**」：

```c++
auto texture_path = "hmap.jpg";
//    auto texture_path = "spot_texture.png";
...
std::function<Eigen::Vector3f(fragment_shader_payload)> active_shader = bump_fragment_shader;
```

在Phong模型的基础上，通过TBN矩阵实现凹凸反射。、

什么是TBN：

* **将纹理坐标对应到模型空间的矩阵**
* The acronym TBN stands for **Tangent, Bitangent, Normal**, and it's used in the context of bump mapping or normal mapping in 3D computer graphics, including the fragment shader.
* **Tangent**: A vector that is perpendicular to the surface of the object and aligned with the direction of increasing texture U-coordinate (along the width of the texture).
* **Bitangent (or Binormal)**: A vector perpendicular to the surface of the object and aligned with the direction of increasing texture V-coordinate (along the height of the texture).
* **Normal**: A vector pointing directly out from the surface of the object, it is perpendicular to both the Tangent and Bitangent vectors.

```c+=
// TODO: Implement bump mapping here
// Let n = normal = (x, y, z)
// Vector t = (x*y/sqrt(x*x+z*z),sqrt(x*x+z*z),z*y/sqrt(x*x+z*z))
// Vector b = n cross product t
// Matrix TBN = [t b n]
// dU = kh * kn * (h(u+1/w,v)-h(u,v))
// dV = kh * kn * (h(u,v+1/h)-h(u,v))
// Vector ln = (-dU, -dV, 1)
// Normal n = normalize(TBN * ln)
```

构建TBN矩阵：

```c++
auto x = normal.x();
auto y = normal.y();
auto z = normal.z();
Eigen::Vector3f t(x*y/sqrt(x*x+z*z),sqrt(x*x+z*z),z*y/sqrt(x*x+z*z));
Eigen::Vector3f b = normal.cross(t);
Eigen::Matrix3f TBN; //TBN矩阵: 将纹理坐标对应到模型空间中
TBN <<
    t.x(), b.x(), normal.x(),
    t.y(), b.y(), normal.y(),
    t.z(), b.z(), normal.z();
```

![image-20230531110424151](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/es1nkRUud6bQ8Mx.png)

通过纹理微分与缩放因子，计算 $dU, dV$ ，分别代表 $u,v$ 方向上的增量。一下是详细步骤：

1. 前两行代码从payload中获取纹理坐标 `u, v` ，范围是在0到1之间。
2. 三四行代码用于获取材质贴图的大小，用于计算纹理微分（比如 `u + 1.0f / w, v` ）
3. `dU` 是在u方向上的微分， `dV` 是v方向上的微分。
4. `getColor` 函数用于从获取纹理贴图获取颜色。通过链式调用 `norm()` 函数获取颜色（光照）的强度。
5.  `(u + 1.0f / w, v)` 对当前点稍微右边的纹理进行采样， `(u, v)` 对当前点的纹理进行采样。 `dV` 同理。

    ![凹凸贴图微分](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/jSeUq8LdpsziA7f.png)
6. `kh,kn` 用于控制**纹理高度的变化**。定义： `float kh = 0.2, kn = 0.1;` 。

```c++
auto u = payload.tex_coords.x();
auto v = payload.tex_coords.y();
auto w = payload.texture->width;
auto h = payload.texture->height;

auto dU = kh*kn*(payload.texture->getColor(u + 1.0f / w, v).norm() - payload.texture->getColor(u, v).norm());
auto dV = kh*kn*(payload.texture->getColor(u, v + 1.0f / h).norm() - payload.texture->getColor(u, v).norm());
```

接下来，根据写出 (dU, dV) 的垂直向量 `ln` （扩充到三维）。

```c++
Eigen::Vector3f ln{ -dU,-dV,1.0f };
```

最后利用TBN矩阵转换到世界坐标，归一化，并且输出颜色到result\_color：

```c++
normal = TBN * ln;
Eigen::Vector3f result_color = normal.normalized();

return result_color * 255.f;
```

效果：

![凹凸小奶牛](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/mot1jGDMcvQ42PT.png)

#### 6. 实现位移片元着色器

在凹凸贴图的基础上做修改。

凹凸贴图不影响物体的形状，只是通过扰动法向量实现凹凸感。

位移片元着色器则会获取模型的顶点，然后通过 `kn * normal * payload.texture->getColor(u, v).norm()` 抬高或压低顶点 `point` ，**方向是法向量的方向**。

```c++
......
auto dV = kh * kn * (payload.texture->getColor(u, v + 1.0f / h).norm() - payload.texture->getColor(u, v).norm());

Eigen::Vector3f ln{ -dU,-dV,1.0f };
//将物体表面拉高
point += (kn * normal * payload.texture->getColor(u, v).norm());
normal = TBN * ln;
normal.normalized();
Eigen::Vector3f result_color = {0, 0, 0};


for (auto &light: lights) {
	......
```

接下来，使用Blinn-Phong Reflection Model处理所有光线。

```c++
auto v = eye_pos - point; //v为出射光方向（指向眼睛）
auto l = light.position - point; //l为指向入射光源方向
auto h = (v + l).normalized(); //h为半程向量即v+l归一化后的单位向量
auto r = l.dot(l); //衰减因子
auto ambient = ka.cwiseProduct(amb_light_intensity);
auto diffuse = kd.cwiseProduct(light.intensity / r) * std::max(0.0f, normal.normalized().dot(l.normalized()));
auto specular = ks.cwiseProduct(light.intensity / r) * std::pow(std::max(0.0f, normal.normalized().dot(h)), p);
result_color += (ambient + diffuse + specular);
```

就大功告成，完成所有牛牛的渲染了。

![位移片元贴图小奶牛](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/vLW7KOpe4aTglY9.png)

### Reference

\[1] Fundamentals of Computer Graphics 4th

\[2] GAMES101 Lingqi Yan

\[3] https://zhuanlan.zhihu.com/p/465058581

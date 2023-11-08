# 计算机图形学六：着色（着色模型、图形管线、纹理映射）

本章內容：

1. 平面着色（Flat Shading）、Gouraud着色和Phong着色
2. 图形管线的工作流程
3. 纹理映射的概念

## 计算机图形学六：着色（着色模型、图形管线、纹理映射）

详细介绍了计算机图形学中的三种基本几何着色技术：平面着色（Flat Shading）、Gouraud着色和Phong着色，以及图形管线的工作流程。最后讲解纹理映射的概念。

### 1. 着色模型

在 GAMES101 中称为：Shading Frequencies。一般表示的是着色模型。

* 着色频率会导致什么？

![image-20230522221329080](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/gHveXRI2QTjsCrw.png)

上面三个球用的是同一个模型，但是经过不同的着色频率，结果就会不一样。

大多基于局部光照模型，它们主要考虑的是直接的光源影响，而没有考虑场景中其他物体对当前物体的间接光照影响。对于这种全局光照效果的模拟，通常需要使用更复杂的算法，如光线追踪或者辐射度方法等。

有几种常见的基于局部光照着色模型：

1. Flat Shading着色
2. Gouraud着色
3. Phong着色

#### 1.1 Flat Shading着色

![image-20230522222804798](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/fgyrvQdLkHGYeK3.png)

这是最基本的着色技术，每个面（多边形）都被赋予一个单一的颜色。

这种颜色是通过计算多边形表面与光源之间的角度得到的，它不考虑多边形内部的任何变化。

* 优点：计算速度快
* 缺点：由于没有光照的变化，造成的效果通常显得较为生硬，不真实

#### 1.2 Gouraud着色

![image-20230522222813278](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/jstXGaAg8ilP9Vc.png)

Gouraud着色是一种插值着色方法，它首先在**每个顶点**处计算颜色，然后在多边形内部对颜色进行插值。

* 优点：可以使多边形内部的颜色变化看起来更加平滑，从而产生更加真实的视觉效果。
* 缺点：它并不能处理高光（specular highlights）

#### 1.3 Phong着色

![image-20230522222820077](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/dHOZoIzFCkAEwJj.png)

在每个像素处计算光照，而不仅仅是在顶点上。

Phong着色首先在顶点上计算法线（表面的朝向），然后在多边形内部对这些法线进行插值。

接着，Phong着色在每个像素处使用这些插值得到的法线来计算光照。

* 区分一下 Blinn-Phong Reflectance Model。这是一种反射模型，用于描述物体表面对光线的反射特性。
* 优点：每个像素处都进行了光照计算，因此Phong着色可以产生非常逼真的高光效果
* 缺点：需要更多的计算资源

#### 1.4 三种着色方法总结

适用的情况：

* **平面着色**：如果渲染速度是首要考虑因素，或者渲染的图形是由许多小面构成
* **Gouraud着色**：如果你需要比平面着色更好的视觉效果，但又不希望花费过多的计算资源，那么Gouraud着色是一个很好的折中方案。但请注意，它不能很好地处理高光效果。
* **Phong着色**：如果你需要最高的视觉质量，特别是需要精细的高光效果，那么Phong着色是最好的选择。但请注意，Phong着色的计算成本也是最高的。

#### 1.5 顶点法线（Per-Vertex Normal Vectors）

用于光照计算，决定物体的表面如何反射光线。

一个顶点法线是一个单位向量，通常表示3D物体表面在某点的方向或"朝向"。在顶点处，这个点就是顶点自身。

在许多情况下，一个顶点的法线可以通过求平均值来从其相邻的面的法线计算得出。例如，如果一个顶点是三个面的共享点，那么这个顶点的法线可以通过将这三个面的法线向量相加，然后归一化（即，使其长度为1）来得到。这种方法通常可以产生良好的结果，特别是在模拟平滑表面时。

![image-20230522224802310](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/xMd9f2g8lKSDWcj.png) \$$ N\_v=\frac{\sum\_i N\_i}{\left\\|\sum\_i N\_i\right\\|} \$$ 然而，也有时候你可能想要明确地定义每个顶点的法线。这可能是因为你想要模拟一些特殊的表面特性，如镜面反射，或者你的模型包含一些尖锐的边缘，这些边缘在自然中不可能有平滑的表面。

* 有一点是非常重要的：顶点法线应该始终是**单位向量**。如果它们的长度不是1，那么在光照计算中可能会出现错误。

### 2. 图形管线(Graphics Pipeline)

图形管线(Graphics Pipeline)，或者称为实时渲染管线。

它描述了从3D模型数据到最终显示在屏幕上的2D像素图像的转换过程。

实时渲染管线的步骤：

1. **模型转换(Model Transformation)**：这是第一步，每个对象的顶点（通常定义在对象的局部坐标空间）被转换到世界坐标空间。
2. **视图转换(View Transformation)**：然后，这些世界坐标空间的点被转换到视图空间或者相机空间。这个步骤通常涉及将世界坐标系变换到以相机为中心的坐标系。
3. **投影转换(Projection Transformation)**：接下来，进行投影转换，将3D空间内的点投影到2D的图像平面上。这个过程会产生透视效果，使得远离相机的物体看起来比近处的物体小。
4. **光栅化(Rasterization)**：在这个阶段，管线接受顶点并转化为像素，构成片元(Fragments)，这些片元用于创建最终的2D图像。在此过程中，渲染器会确定哪些像素属于哪些多边形，并将颜色，深度和其他属性分配给这些像素。
5. **片元着色(Fragment Shading)**：这个阶段的输入是光栅化阶段产生的片元。着色器通常在这个阶段进行，计算每个片元的最终颜色。这里可以执行各种操作，包括纹理映射，光照计算等。
6. **测试和混合(Test and Blending)**：在最后阶段，通常会进行一些测试和操作，如深度测试（判断哪个片元应该出现在最前面），模板测试，混合操作（如透明度混合）等。最后生成的像素将被发送到帧缓冲区，等待显示到屏幕上。

类似下图所示：

![image-20230522230031593](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/dIb2ZGz1TSmsCya.png)

### 3. 着色器程序(Shader Programs)

上面的图形管线是一种概念上的模型，讲述的是将3D模型转换为2D屏幕图像和渲染的过程。

而着色器程序是图形管线中某个具体的小程序。这些程序在GPU上执行。常见的着色器程序包括顶点着色器（Vertex Shaders）、几何着色器（Geometry Shaders）、片元着色器（Fragment Shaders）等。

* 顶点着色器主要负责处理顶点数据，例如变换顶点的位置，计算顶点的颜色等。
* 几何着色器可以创建或者销毁几何体，例如从一个点生成一组点来形成一个粒子系统。
* 片元着色器负责处理光栅化阶段生成的片元，如根据光照模型和材质属性计算片元的颜色。

一个最基本的顶点着色器：

```glsl
#version 330 core
layout (location = 0) in vec3 aPos;
void main()
{
    gl_Position = vec4(aPos, 1.0);
}
```

此着色器接收位置为一个输入（`aPos`），并直接将其用作顶点的位置。

一个非常基本的片元着色器：

```glsl
#version 330 core
out vec4 FragColor;
void main()
{
    FragColor = vec4(1.0, 0.5, 0.2, 1.0);
}
```

此着色器为所有片元输出同一颜色，这是一个常数颜色，RGBA值为(1.0, 0.5, 0.2, 1.0)。

以下是一个入门的OpenGL实例代码，读者可以按需阅读：

> 注意，此示例默认您已经在你的开发环境中设置好了GLFW和GLAD。

```c++
#include <glad/glad.h>
#include <GLFW/glfw3.h>

// Vertex shader source code
const char *vertexShaderSource = "#version 330 core\n"
    "layout (location = 0) in vec3 aPos;\n"
    "void main()\n"
    "{\n"
    "   gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);\n"
    "}\0";

// Fragment shader source code
const char *fragmentShaderSource = "#version 330 core\n"
    "out vec4 FragColor;\n"
    "void main()\n"
    "{\n"
    "   FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);\n"
    "}\n\0";

int main()
{
    glfwInit();

    // Create a windowed mode window and its OpenGL context
    GLFWwindow* window = glfwCreateWindow(800, 600, "Hello Triangle", NULL, NULL);

    glfwMakeContextCurrent(window);

    // Load OpenGL function pointers
    gladLoadGLLoader((GLADloadproc)glfwGetProcAddress);

    // Compile and setup the shaders
    // ... (use the above shader source code to compile and link into a shader program)

    // Define the viewport dimensions
    glViewport(0, 0, 800, 600);

    // Render loop
    while (!glfwWindowShouldClose(window))
    {
        // Input
        // ...

        // Render
        glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        // Draw the triangle
        // ...

        // Swap buffers
        glfwSwapBuffers(window);
        glfwPollEvents();    
    }

    glfwTerminate();
    return 0;
}

```

推荐一个在线写Shader的网站。

http://shadertoy.com

### 4. 纹理映射

纹理映射（Texture Mapping）是一种在计算机图形中将图片（称为纹理）映射到3D对象表面的技术。

![image-20230523165114802](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/ZwNUmF9HGLy5thq.png)

纹理映射的基本步骤：

1. **设置纹理坐标**: 对于你的模型上的每个顶点，你需要定义一个纹理坐标（通常称为UV坐标）。这告诉渲染器每个顶点对应到纹理图像的哪个部分。
2. **在着色器中使用纹理**: 在你的着色器程序（如片元着色器）中，你可以使用UV坐标来查找纹理中的颜色，然后使用这个颜色来计算片元的最终颜色。

![image-20230523165617233](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/RhYi3Ps1DqMZ4Cc.png)

#### 平铺纹理(Tiled Textures)

"平铺纹理"（Tiled Textures）也称为"重复纹理"（Repeating Textures），是一种将纹理设计成能够在水平和垂直方向上无缝连接的方式，这样就可以覆盖大面积的物体表面而不产生明显的重复感或边缘。

一般来说，纹理图像的边长都是2的幂，例如128x128。

#### 纹理和着色的区别

**纹理（Texture）**：将图像（纹理贴图）映射到3D对象的表面。可以给物体带来如颜色、反射率、透明度、凹凸等特性。

**着色（Shading）**：计算每个像素（或者称为片元）的颜色的过程。着色过程使用各种着色模型或着色算法，如Phong模型、Gouraud模型等。可以给物体带来如材质、纹理、光源信息、相机位置等因素，来计算像素的颜色。

* 不同的材质就是不同的着色方法。

### Reference

\[1] Fundamentals of Computer Graphics 4th

\[2] GAMES101 Lingqi Yan

# 重心坐标法（贝塞尔坐标法）判断三角形与点的关系

简单地说就是，给定一个三角形ABC和任意一个点 $(x,y)$ ，这个点的坐标都可以用点ABC线性表示。且重心坐标的性质是：他们的系数之和为1。也就是说$(x, y)= a A + b B + c C, a+b+c=1$

至于这三个系数怎么求有点复杂，最容易理解的方法是用面积法。

但是有个结论很简洁，也是上面算法用到的：如果点在三角形内部，则三个系数属于（0,1）之间；如果点在三角形边上，则至少一个系数等于0或1；如果点在三角形外面，则至少有一个系数大于1或小于0。

***

### 判断点与三角形关系算法思想

贝塞尔坐标（也被称为重心坐标或三角形坐标）用于在三角形中描述一个点的位置。对于三角形的顶点 A, B, C 和内部或边界上的点 P，P 的贝塞尔坐标（u, v, w）可以通过以下公式得到：

首先，我们需要计算向量：

```
Vector v0 = C - A
Vector v1 = B - A
Vector v2 = P - A
```

然后，我们计算以下点积：

```
float dot00 = dot(v0, v0)
float dot01 = dot(v0, v1)
float dot02 = dot(v0, v2)
float dot11 = dot(v1, v1)
float dot12 = dot(v1, v2)
```

这些点积用于计算贝塞尔坐标的分母和分子。最后，我们计算 u 和 v：

```
float invDenom = 1 / (dot00 * dot11 - dot01 * dot01)
float u = (dot11 * dot02 - dot01 * dot12) * invDenom
float v = (dot00 * dot12 - dot01 * dot02) * invDenom
```

其中，`invDenom` 它可以视为对角线乘积减去非对角线乘积的倒数，这是用来计算二维向量投影的公式的一部分。

至于 w，它可以由以下公式得到：

```
w = 1 - u - v
```

注意，u, v 和 w 的和总是等于 1，这是贝塞尔坐标的性质。而且，如果点 P 在三角形内部，那么 u、v 和 w 都在 0 和 1 之间。如果点 P 在三角形的边界上，那么 u、v 和 w 中至少有一个等于 0 或 1。如果点 P 在三角形的外部，那么 u、v 和 w 中至少有一个小于 0 或大于 1。

请注意这个计算方法对于退化的三角形（三个顶点在同一直线上）无效，因为此时分母 `dot00 * dot11 - dot01 * dot01` 会等于零，不能作为除数。

```c++
static bool insideTriangle(int x, int y, const Vector3f* _v)
{
    // Compute vectors
    Vector2f v0 = _v[2].head<2>() - _v[0].head<2>();
    Vector2f v1 = _v[1].head<2>() - _v[0].head<2>();
    Vector2f v2 = Vector2f(x, y) - _v[0].head<2>();

    // Compute dot products
    float dot00 = v0.dot(v0);
    float dot01 = v0.dot(v1);
    float dot02 = v0.dot(v2);
    float dot11 = v1.dot(v1);
    float dot12 = v1.dot(v2);

    // Compute barycentric coordinates
    float invDenom = 1.0f / (dot00 * dot11 - dot01 * dot01);
    float u = (dot11 * dot02 - dot01 * dot12) * invDenom;
    float v = (dot00 * dot12 - dot01 * dot02) * invDenom;

    // Check if point is in triangle
    return (u >= 0) && (v >= 0) && (u + v < 1);
}
```

***

### 计算重心坐标的参数的方法

```c++
static std::tuple<float, float, float> computeBarycentric2D(float x, float y, const Vector3f* v)
{
    float c1 = (x*(v[1].y() - v[2].y()) + (v[2].x() - v[1].x())*y + v[1].x()*v[2].y() - v[2].x()*v[1].y()) / (v[0].x()*(v[1].y() - v[2].y()) + (v[2].x() - v[1].x())*v[0].y() + v[1].x()*v[2].y() - v[2].x()*v[1].y());
    float c2 = (x*(v[2].y() - v[0].y()) + (v[0].x() - v[2].x())*y + v[2].x()*v[0].y() - v[0].x()*v[2].y()) / (v[1].x()*(v[2].y() - v[0].y()) + (v[0].x() - v[2].x())*v[1].y() + v[2].x()*v[0].y() - v[0].x()*v[2].y());
    float c3 = (x*(v[0].y() - v[1].y()) + (v[1].x() - v[0].x())*y + v[0].x()*v[1].y() - v[1].x()*v[0].y()) / (v[2].x()*(v[0].y() - v[1].y()) + (v[1].x() - v[0].x())*v[2].y() + v[0].x()*v[1].y() - v[1].x()*v[0].y());
    return {c1,c2,c3};
}
```

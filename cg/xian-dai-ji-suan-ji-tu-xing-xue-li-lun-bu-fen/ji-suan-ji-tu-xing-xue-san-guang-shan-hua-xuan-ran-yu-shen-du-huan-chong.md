# 計算機圖形學三：光柵化渲染與深度緩衝

本章內容：（未完待續）

1. 圖像、像素與幾何
2. 採樣 Sampling
3. 深度緩衝 Z-buffering

## 第三章 光柵化渲染與深度緩衝 Raster Images\&Z-buffer

上一章節內容：

1. 三維變換 3D transformations
2. 觀測變換 viewing tranformation
3. 具體代碼實現

上一章難度較大，數學推導過程較為複雜，理論理解也有一定難度。這一章難度較低。

本章學習內容：

1. 光柵概述
2. 深度緩衝 Z-buffer

簡述：

> 光柵化渲染（Rasterization）是一種在電腦圖形學中常用的渲染技術，它將3D模型的幾何形狀和材質投影到2D畫面上，並根據每個像素的位置、光照和材質等信息進行計算，最終生成2D畫面。這種渲染技術的運作原理是將3D模型中的三角形面片按照其在視角中的位置關係進行排序，然後利用像素著色器對每個像素進行顏色填充。由於光柵化渲染運算速度快、容易實現，因此在遊戲和實時圖形應用中得到廣泛應用。
>
> 光柵化渲染過程主要分為三個步驟：幾何著色、光照計算和像素著色。在幾何著色階段，將3D模型中的每個三角形面片變換到視角空間中，並進行視錐體剪裁，去除不在視錐體內的面片。然後，計算每個面片的法向量和材質，以便在光照計算階段進行使用。在光照計算階段，對每個面片進行簡單的光照計算，以決定每個面片上各點的亮度和顏色。在像素著色階段，將每個面片的像素投影到屏幕上，然後對每個像素進行插值和紋理映射，計算其最終顏色。最後，將所有像素的顏色存儲到顯示器的顯存中，顯示到屏幕上，完成渲染過程。
>
> 儘管光柵化渲染速度快，但其精度和真實感相對較低，且對於大型場景和複雜物體的渲染能力也較有限。因此，在許多應用中，例如電影、動畫等專業領域，更多地採用基於物理的渲染技術，例如光線追蹤（Ray Tracing）等。

### 3.1 圖像、像素與幾何 Images, Pixels, and Geometry

這一章的內容只需了解，不用太深入。

#### 3.1.1 屏幕像素表示 Coordinates of Pixel

* 屏幕的每一個像素座標都是整數
* 一般來說，如果屏幕有 $i\cdot j$ 個像素，則：
  * 左下角是 $(0,0)$ ，右上角是 $(i-1,j-1)$
  * 每一個像素的中心是： $(x,y)$
* 某些圖形API中， $y$ 軸可能是向下的。

![image-20230411202315489](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/G5W8Mq3knXJUDmF.png)

#### 3.1.2 像素值 Pixel Value

* 目前來說，一個邏輯像素只代表一種顏色
* 8位图像中，每一個像素可能的數值 $0, 1/255, 2/255,. . . , 254/255, 1$
* 至於高動態範圍（HDR）的像素，將在最後兩個章節詳細討論

#### 3.1.3 三角形 Fundamental Shape Primitives

* 為什麼三角形被認為是一種基本的形狀基元？
  * 三角形是二維平面中最簡單的多邊形
  * 容易判斷某點是否在三角形內：只需做三次叉積

### 3.2 用像素近似三角形 Pixel Values Approximate a Triangle

![image-20230412105333153](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/cxLoVG6bfXYnmpF.png)

|     Input    |   Output   |
| :----------: | :--------: |
| 三角形在屏幕上的投影位置 | 用一組像素模擬三角形 |

$$
R_\text{Input} \Rightarrow R_\text{Output}
$$

其中一個簡單的方法就是：**採樣**。

#### 3.2.1 採樣

* 在某個點對函數求值就是採樣（Sampling）。
*   偽代碼如下：

    ```
    for (int x = 0; x < xmax; ++x)
        output[x] = f(x);
    ```
* 對下面整個平面進行遍歷掃描（也就是每一個正方形），規定：
  * 若某個正方形的中心在三角形內部，則這個正方形像素=1；反之

![image-20230419170031940](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/MgZ8JS3ix7nE1eK.png) ![image-20230419170059182](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/oenPwjcZaOmB3Ri.png)

*   偽代碼實現如下：

    ```
    for (int x = 0; x < xmax; ++x)
        for (int y = 0; y < ymax; ++y)
          image[x][y] = inside(tri, x + 0.5, y + 0.5);
    ```
*   具體怎麼判斷某個點是否在三角形內呢？也就是這個inside函數怎麼實現？

    * inside(tri, x, y)實現原理：
      * 三次叉積。三次作叉積就可以判斷$Q$點是否在三角形 $P\_0 P\_1 P\_2$ 內

    ![image-20230419172033297](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/t2XAMkaUGc8QhvW.png)
* 我們一般**隨意**處理點在線上的問題。

#### 3.2.2 優化採樣

* 3.2.1 的採樣方法是對整個平面每一個像素做inside()。
* 我們真的需要對**每一個**像素採樣嗎？？
*   答案：不需要。我們可以使用最簡單的方法邊框盒子法（**Bounding Box**）。

    簡單的說，就是用一個盒子框住三角形，沒被盒子覆蓋的像素，肯定也不會被三角形覆蓋，所以一定不用渲染。如下圖所示。

    ![image-20230419172601918](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/LyrfYbu6UXHDoqQ.png)

### 3.3 深度緩衝 Z-buffer

簡而言之，為了保證3D場景中物體的深度排序正確，我們可以使用Z-buffer算法。

當渲染一個場景時，每個物體都有一個與鏡頭距離的深度值。在沒有深度排序的情況下，有些物體可能會被渲染在其他物體的前面，導致視覺上的錯誤。為了避免這種情況，z-buffer算法通過維護一個深度緩衝區（也就是z-buffer）來跟踪每個像素的深度值。當渲染一個像素時，系統會檢查其深度值是否比已經渲染的像素的深度值更靠前。如果是，則這個像素將被渲染；否則，它將被丟棄，因為它被後面的物體遮擋住了。

* 注意，在這一步，我們把所有**深度值都看成正數**！！
* 也就是說，z小 -> 靠近攝像頭

算法步驟：

1.  將 $m\*n$ 深度緩衝區全部設置為無窮大 $∞$

    （c++中的無窮大是：`std::numeric_limits<float>::infinity()`）
2. 遍歷所有三角形
   1. 遍歷所有在三角形內的像素
      1. 如果這個像素的深度小於緩衝區的，則取代他。
      2. 否則，不動。

偽代碼：

```
for (each triangle T)
    for (each sample (x,y,z) in T)
        if (z < zbuffer[x,y]) // closest sample so far
            framebuffer[x,y] = rgb; // update color
            zbuffer[x,y] = z; // update depth
        else
            ;// do nothing, this sample is occluded
```

&#x20;圖示：

![image-20230419175545879](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo\_t/vA5pLg1YrJuFCbc.png)

* 算法時間複雜度：$O(n)$

### 3.4 coding

```
//Screen space rasterization
void rst::rasterizer::rasterize_triangle(const Triangle& t) {
    auto v = t.toVector4();

    // TODO : Find out the bounding box of current triangle.
    // iterate through the pixel and find if the current pixel is inside the triangle

    // If so, use the following code to get the interpolated z value.
    //auto[alpha, beta, gamma] = computeBarycentric2D(x, y, t.v);
    //float w_reciprocal = 1.0/(alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
    //float z_interpolated = alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
    //z_interpolated *= w_reciprocal;

    // TODO : set the current pixel (use the set_pixel function) to the color of the triangle (use getColor function) if it should be painted.

    // Find bounding box of triangle
    int x_min = std::max(0, static_cast<int>(std::floor(std::min({v[0].x(), v[1].x(), v[2].x()}))));
    int x_max = std::min(700, static_cast<int>(std::ceil(std::max({v[0].x(), v[1].x(), v[2].x()}))));
    int y_min = std::max(0, static_cast<int>(std::floor(std::min({v[0].y(), v[1].y(), v[2].y()}))));
    int y_max = std::min(700, static_cast<int>(std::ceil(std::max({v[0].y(), v[1].y(), v[2].y()}))));

    for (int y = y_min; y < y_max; ++y) {
        for (int x = x_min; x < x_max; ++x) {
            // Compute barycentric coordinates of pixel with respect to triangle
            auto [alpha, beta, gamma] = computeBarycentric2D(x, y, t.v);
            if (alpha >= 0 && beta >= 0 && gamma >= 0) {
                // Pixel is inside triangle, interpolate z and set pixel color
                float w_reciprocal = 1.0 / (alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
                float z_interpolated = alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
                z_interpolated *= w_reciprocal;
                auto color = t.getColor();
                set_pixel(Eigen::Vector3f(x, y, z_interpolated), color);
            }
        }
    }
}
```

### Reference

\[1] Fundamentals of Computer Graphics 4th

\[2] GAMES101 Lingqi Yan
